# Sentry Upgrades
This document outlines the steps required to successfully migrate the Sentry Helm chart from version 11.6.0 (Sentry image 21.6.3) to the latest version, 25.12.0. We are currently using `helmfile` for our releases.

## Upgrade from 11.6.0 to 14.1.0

### Steps

1. **Internal Redis**: 
   - If internal Redis is being used, remove the StatefulSets and the associated volumes. The Redis chart will be upgraded and will no longer use Redis slave functionality.

2. **Remove Fixed Image Versions**:
   - If Sentry, Snuba, or Relay image versions are fixed, remove them from the configuration.

    ```yaml
    # Remove fixed image versions
    images:
      relay:
        tag: "21.6.3"
      sentry:
        tag: "21.6.3"
      snuba:
        tag: "21.6.3"
    ```

3. **Apply Upgrade with JSON Patches**:
   - The Snuba migrations will fail without this modification. Use `helmfile` with `jsonPatches` to modify the manifests as follows:

    ```yaml
    jsonPatches:
      - target:
          group: batch
          version: v1
          name: sentry-snuba-db-init
        patch:
          - op: replace
            path: /spec/template/spec/containers/0/command
            value:
              - /bin/bash
              - -ec
              - |
                for ((i=0;i<3;i++)); do
                  export CLICKHOUSE_HOST=sentry-clickhouse-$i.sentry-clickhouse-headless;
                  snuba bootstrap --no-migrate --force;
                done
          - op: add
            path: /spec/template/spec/containers/0/env/3
            value:
              name: CLICKHOUSE_SINGLE_NODE
              value: "true"
              
      - target:
          group: batch
          version: v1
          name: sentry-snuba-migrate
        patch:
          - op: replace
            path: /spec/template/spec/containers/0/command
            value:
              - /bin/bash
              - -ec
              - |
                for ((i=0;i<3;i++)); do
                  export CLICKHOUSE_HOST=sentry-clickhouse-$i.sentry-clickhouse-headless;
                  snuba migrations migrate --force;
                done
          - op: add
            path: /spec/template/spec/containers/0/env/3
            value:
              name: CLICKHOUSE_SINGLE_NODE
              value: "true"

      - target:
          kind: ConfigMap
          name: sentry-snuba
        patch:
          - op: replace
            path: /data/settings.py
            value: |
              import os
              from snuba.settings import *

              env = os.environ.get

              DEBUG = env("DEBUG", "0").lower() in ("1", "true")

              # ClickHouse Options
              CLUSTERS = [
                {
                  "host": env("CLICKHOUSE_HOST", "sentry-clickhouse"),
                  "port": 9000,
                  "user":  env("CLICKHOUSE_USER", "default"),
                  "password": env("CLICKHOUSE_PASSWORD", ""),
                  "max_connections": int(os.environ.get("CLICKHOUSE_MAX_CONNECTIONS", 100)),
                  "database": env("CLICKHOUSE_DATABASE", "default"),
                  "http_port": 8123,
                  "storage_sets": {
                      "cdc",
                      "discover",
                      "events",
                      "events_ro",
                      "metrics",
                      "migrations",
                      "outcomes",
                      "querylog",
                      "sessions",
                      "transactions",
                      "transactions_ro",
                      "transactions_v2",
                      "errors_v2",
                      "errors_v2_ro",
                      "profiles",
                  },
                  "single_node": True,
                },
              ]

              # Redis Options
              REDIS_HOST = "sentry-sentry-redis-master" # Adapt this to the current Redis host
              REDIS_PORT = 6379
              REDIS_PASSWORD = ""
              REDIS_DB = int(env("REDIS_DB", 1))

              # No Python Extension Config Given
    ```

4. **Deploy the Upgrade**:
   - Apply the changes using `helmfile` with the appropriate production environment variables.

    ```bash
    helmfile -e production apply
    ```


### Problems

1. **ClickHouse Replication/Sharding**:
   - ClickHouse now uses replication and sharding for Snuba data. However, Snuba migrations won’t automatically change the current `MergeTree` tables to `ReplicatedMergeTree` tables. If there are multiple ClickHouse pods, Snuba migrations are likely to be incomplete, causing some migrations to get stuck in the `InProgress` stage. This will result in all subsequent migrations and upgrades failing.
   - Manually reversing migrations using the `snuba migrations reverse ...` command (see [Snuba Migrations Guide](https://github.com/getsentry/snuba/blob/master/MIGRATIONS.md#reverse-a-single-migration)) will not resolve the issue, as the migrations will be inconsistent across different ClickHouse databases.

2. **MergeTree Table Limitations**:
   - Retaining `MergeTree` tables will cause issues in future upgrades, as Snuba now expects replicated tables managed by ZooKeeper. Querying and writing data can become inconsistent, leading to errors such as: `Sorry, the events for this issue could not be found.` This happens because events might be written to different ClickHouse nodes.

3. **Single MergeTree Node Risks**:
   - Reducing ClickHouse to a single `MergeTree` node will result in data loss on other nodes, making ClickHouse less available and more vulnerable to failure.

4. **Manual Migration Issues**:
   - Manually migrating each table from `MergeTree` to `ReplicatedMergeTree` using the `CREATE TABLE ... ENGINE = ReplicatedMergeTree(...)` command can cause data loss. Data across nodes might differ, and ZooKeeper will attempt to sync data from the master node to other replicated nodes, potentially deleting data on replicated nodes.
   - Furthermore, manually modifying data on each ClickHouse node without ZooKeeper's awareness will lead to ZooKeeper getting stuck whenever a pod restarts.

5. **ZooKeeper Issues**:
   - In higher versions of Sentry, ClickHouse ZooKeeper may encounter errors and get stuck in an `invalid magic number` state after restarts.

---

### Solution - Exporting Current Data & Using External ClickHouse

#### 1. Scale Down Snuba Deployments
   - Before proceeding with the migration, scale down all Snuba deployments.

#### 2. Increase ClickHouse Volume Size and Resource Limits
   - Increase the first ClickHouse volume to **90Gi** and remove container resource limits. Note: the size should be greater than 3 * size of the currently used storage
   - Ensure there are sufficient resources, especially RAM (at least **16Gi** of free RAM depending on the data size).
   - Enter the `clickhouse-0` pod bash and set the maximum memory usage to handle the data export:

    ```bash
    clickhouse-client --host 127.0.0.1 --query="SET max_memory_usage = 34359738368"
    ```

#### 3. Data Export

   - Since the `_dist` tables contain data from all ClickHouse nodes, copy the data into temporary `MergeTree` tables. The following six tables contain necessary data for migration:

    ```
    errors_local
    outcomes_hourly_local
    outcomes_raw_local
    sessions_hourly_local
    sessions_raw_local
    transactions_local
    ```

   - **Create Temporary MergeTree Tables**:

    ```sql
    CREATE TABLE default.errors_local_tmp AS default.errors_local;
    CREATE TABLE default.outcomes_hourly_local_tmp AS default.outcomes_hourly_local;
    CREATE TABLE default.outcomes_raw_local_tmp AS default.outcomes_raw_local;
    CREATE TABLE default.sessions_hourly_local_tmp AS default.sessions_hourly_local;
    CREATE TABLE default.sessions_raw_local_tmp AS default.sessions_raw_local;
    CREATE TABLE default.transactions_local_tmp AS default.transactions_local;
    ```

   - **Copy Data to Temporary Tables**:
     - Some tables require the `FINAL` modifier to ensure all data from all nodes is copied:

    ```sql
    INSERT INTO errors_local_tmp SELECT * FROM errors_dist FINAL;
    INSERT INTO outcomes_hourly_local_tmp SELECT * FROM outcomes_hourly_dist FINAL;
    INSERT INTO outcomes_raw_local_tmp SELECT * FROM outcomes_raw_dist;
    INSERT INTO sessions_hourly_local_tmp SELECT * FROM sessions_hourly_dist FINAL;
    INSERT INTO sessions_raw_local_tmp SELECT * FROM sessions_raw_dist;
    INSERT INTO transactions_local_tmp SELECT * FROM transactions_dist;
    ```

   - **Handle Deprecated Columns**:
     - If `_tags_flattened` and `_contexts_flattened` columns still exist in `transactions_dist`, drop them and re-run the `INSERT` command for `transactions_local_tmp`:

    ```sql
    ALTER TABLE transactions_dist DROP COLUMN _tags_flattened;
    ALTER TABLE transactions_dist DROP COLUMN _contexts_flattened;
    ```

#### 4. Export Data in Native Format

   - Exit the SQL terminal and export the data from the temporary tables using the Native format:

    ```bash
    clickhouse-client --host 127.0.0.1 --query="SELECT * FROM default.errors_local_tmp" --format=Native > /var/lib/clickhouse/backup/errors_local.native

    clickhouse-client --host 127.0.0.1 --query="SELECT * FROM default.outcomes_hourly_local_tmp" --format=Native > /var/lib/clickhouse/backup/outcomes_hourly_local.native

    clickhouse-client --host 127.0.0.1 --query="SELECT * FROM default.outcomes_raw_local_tmp" --format=Native > /var/lib/clickhouse/backup/outcomes_raw_local.native

    clickhouse-client --host 127.0.0.1 --query="SELECT * FROM default.sessions_hourly_local_tmp" --format=Native > /var/lib/clickhouse/backup/sessions_hourly_local.native

    clickhouse-client --host 127.0.0.1 --query="SELECT * FROM default.sessions_raw_local_tmp" --format=Native > /var/lib/clickhouse/backup/sessions_raw_local.native

    clickhouse-client --host 127.0.0.1 --query="SELECT * FROM default.transactions_local_tmp" --format=Native > /var/lib/clickhouse/backup/transactions_local.native
    ```

#### External ClickHouse Deployment

1. **Create External ClickHouse Release**:
   - Define an external ClickHouse release in your Helm configuration using the `bitnami/clickhouse` chart.

    ```yaml
    releases:
      - name: sentry-{{ .Environment.Name }}-clickhouse
        namespace: {{ $namespace }}
        chart: bitnami/clickhouse
        version: 6.2.27
        wait: true
        timeout: 1800
        values:
          - helm/environments/clickhouse.values.yaml
    ```

2. **Configure External ClickHouse**:
   - In the `helm/environments/clickhouse.values.yaml` file, ensure sufficient resources and volume sizes are allocated. Since there are known issues with ZooKeeper, enable the internal `ClickHouse Keeper` (this feature is still experimental).
   - Adjust `operation_timeout_ms` and `session_timeout_ms` to allow enough time for the data migration process.

    ```yaml
    auth:
      username: sentry

    shards: 1

    persistence:
      storageClass: csi-cinder-sc-delete
      size: 50Gi

    resources:
      limits:
        cpu: 2
        ephemeral-storage: 4Gi
        memory: 6Gi
      requests:
        cpu: 500m
        ephemeral-storage: 1Gi
        memory: 2Gi

    zookeeper:
      enabled: false

    keeper:
      enabled: true
    ```

3. **Disable Internal ClickHouse**:
   - Ensure the internal ClickHouse deployment is disabled in the values file.

    ```yaml
    clickhouse:
      enabled: false
    ```

4. **Enable External ClickHouse**:
   - Set up the external ClickHouse configuration by specifying the external host, ports, credentials, and database.

    ```yaml
    externalClickhouse:
      ## Hostname or IP address of external ClickHouse
      host: "sentry-production-clickhouse"
      tcpPort: 9000
      httpPort: 8123
      username: sentry
      password: "" # Check the sentry-production-clickhouse secret for the password
      database: default
      singleNode: false
      clusterName: default
    ```

5. **Remove Previous JSON Patches**:
   - Remove the `jsonPatches` that were applied in the earlier steps, as they are no longer needed with the external ClickHouse setup.

6. **Apply the Changes**:
   - Run `helmfile apply` to deploy the changes. Wait until everything is up and running.

    ```bash
    helmfile -e production apply
    ```

7. **Scale Down Snuba Deployments**:
   - Scale down all Snuba deployments before continuing.

8. **Verify the External ClickHouse Tables**:
   - Check if all 34 tables have been created in the external ClickHouse.

    ```bash
    clickhouse-client --host 127.0.0.1 --user sentry --password <password>
    
    # Then list all tables
    SHOW TABLES;
    ```

9. **Verify Replicated MergeTree Tables**:
   - Ensure the necessary tables (e.g., `errors_local`, `outcomes_hourly_local`) are using `ReplicatedMergeTree`.

    ```bash
    SHOW CREATE TABLE errors_local FORMAT TabSeparatedRaw;
    SHOW CREATE TABLE outcomes_hourly_local FORMAT TabSeparatedRaw;
    SHOW CREATE TABLE outcomes_raw_local FORMAT TabSeparatedRaw;
    SHOW CREATE TABLE sessions_hourly_local FORMAT TabSeparatedRaw;
    SHOW CREATE TABLE sessions_raw_local FORMAT TabSeparatedRaw;
    SHOW CREATE TABLE transactions_local FORMAT TabSeparatedRaw;
    ```

   - The output should confirm the engine is set to `ReplicatedMergeTree`:

    ```sql
    Engine = Replicated...MergeTree('/clickhouse/tables/<StorageSet>/{shard}/default/<table>', '{replica}', ...)
    ```

#### Importing Backup Data to the External ClickHouse

1. **Increase Timeouts for Migrations**:
   - In the `productive clickhouse.values.yaml` file, increase `operation_timeout_ms` and `session_timeout_ms` to a large value to ensure that migrations and upgrades complete without timeouts.

    ```yaml
    defaultConfigurationOverrides: |
      <clickhouse>
        <!-- Macros -->
        <macros>
          <shard from_env="CLICKHOUSE_SHARD_ID"></shard>
          <replica from_env="CLICKHOUSE_REPLICA_ID"></replica>
          <layer>sentry-production-clickhouse</layer>
        </macros>
        <!-- Log Level -->
        <logger>
          <level>information</level>
        </logger>
        <!-- Cluster configuration -->
        <remote_servers>
          <default>
            <shard>
              <replica>
                <host>sentry-production-clickhouse-shard0-0.sentry-production-clickhouse-headless.sentry-production.svc.cluster.local</host>
                <port>9000</port>
                <user from_env="CLICKHOUSE_ADMIN_USER"></user>
                <password from_env="CLICKHOUSE_ADMIN_PASSWORD"></password>
              </replica>
              <replica>
                <host>sentry-production-clickhouse-shard0-1.sentry-production-clickhouse-headless.sentry-production.svc.cluster.local</host>
                <port>9000</port>
                <user from_env="CLICKHOUSE_ADMIN_USER"></user>
                <password from_env="CLICKHOUSE_ADMIN_PASSWORD"></password>
              </replica>
              <replica>
                <host>sentry-production-clickhouse-shard0-2.sentry-production-clickhouse-headless.sentry-production.svc.cluster.local</host>
                <port>9000</port>
                <user from_env="CLICKHOUSE_ADMIN_USER"></user>
                <password from_env="CLICKHOUSE_ADMIN_PASSWORD"></password>
              </replica>
            </shard>
          </default>
        </remote_servers>
        <!-- ClickHouse Keeper Configuration -->
        <keeper_server>
          <tcp_port>2181</tcp_port>
          <server_id from_env="KEEPER_SERVER_ID"></server_id>
          <log_storage_path>/bitnami/clickhouse/keeper/coordination/log</log_storage_path>
          <snapshot_storage_path>/bitnami/clickhouse/keeper/coordination/snapshots</snapshot_storage_path>

          <coordination_settings>
            <operation_timeout_ms>100000</operation_timeout_ms>
            <session_timeout_ms>300000</session_timeout_ms>
            <raft_logs_level>trace</raft_logs_level>
          </coordination_settings>

          <raft_configuration>
            <server>
              <id>0</id>
              <hostname from_env="KEEPER_NODE_0"></hostname>
              <port>9444</port>
            </server>
            <server>
              <id>1</id>
              <hostname from_env="KEEPER_NODE_1"></hostname>
              <port>9444</port>
            </server>
            <server>
              <id>2</id>
              <hostname from_env="KEEPER_NODE_2"></hostname>
              <port>9444</port>
            </server>
          </raft_configuration>
        </keeper_server>
        <!-- ZooKeeper Configuration -->
        <zookeeper>
          <node>
            <host from_env="KEEPER_NODE_0"></host>
            <port>2181</port>
          </node>
          <node>
            <host from_env="KEEPER_NODE_1"></host>
            <port>2181</port>
          </node>
          <node>
            <host from_env="KEEPER_NODE_2"></host>
            <port>2181</port>
          </node>
        </zookeeper>
      </clickhouse>
    ```

2. **Free Resources**:
   - Remove resources of `sentry-production-clickhouse-shard0` to allow ClickHouse to use as many resources as possible during the migration.

3. **Deploy a ClickHouse Client Pod**:
   - Deploy a `clickhouse-client` pod to the production namespace to facilitate the data import.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: clickhouse-client
      namespace: sentry-production
      labels:
        app: clickhouse-client
    spec:
      volumes:
        - name: clickhouse-0
          persistentVolumeClaim:
            claimName: sentry-clickhouse-data-sentry-clickhouse-0
      containers:
        - name: clickhouse-client
          image: yandex/clickhouse-client
          command: [ "sleep" ]
          args: [ "infinity" ]
          env:
            - name: CLICKHOUSE_HOST
              value: "sentry-production-clickhouse"
            - name: CLICKHOUSE_PORT
              value: "9000"
            - name: CLICKHOUSE_USER
              value: "sentry"
            - name: CLICKHOUSE_PASSWORD
              value: ""  # Get this from the sentry-production-clickhouse secret
          volumeMounts:
            - mountPath: /clickhouse-0
              name: clickhouse-0
      restartPolicy: Never
    ```

4. **Access ClickHouse Client**:
   - Open two terminal sessions (bash) for the `clickhouse-client` to parallelize the data import.

5. **Remove Existing Data in the External Clickhouse**:
   - Use the following queries to remove existing data:
    ```sql
    TRUNCATE TABLE errors_local;
    TRUNCATE TABLE outcomes_hourly_local
    TRUNCATE TABLE outcomes_raw_local
    TRUNCATE TABLE sessions_hourly_local
    TRUNCATE TABLE sessions_raw_local
    TRUNCATE TABLE transactions_local
    ```

6. **Import Data to the External ClickHouse**:
   - Use the following commands to import the previously exported data into the external ClickHouse instance:

    ```bash
    clickhouse-client --host $CLICKHOUSE_HOST --port $CLICKHOUSE_PORT --user $CLICKHOUSE_USER --password $CLICKHOUSE_PASSWORD --query="INSERT INTO default.errors_local FORMAT Native" < /clickhouse-0/backup/errors_local.native

    clickhouse-client --host $CLICKHOUSE_HOST --port $CLICKHOUSE_PORT --user $CLICKHOUSE_USER --password $CLICKHOUSE_PASSWORD --query="INSERT INTO default.outcomes_hourly_local FORMAT Native" < /clickhouse-0/backup/outcomes_hourly_local.native

    clickhouse-client --host $CLICKHOUSE_HOST --port $CLICKHOUSE_PORT --user $CLICKHOUSE_USER --password $CLICKHOUSE_PASSWORD --query="INSERT INTO default.outcomes_raw_local FORMAT Native" < /clickhouse-0/backup/outcomes_raw_local.native

    clickhouse-client --host $CLICKHOUSE_HOST --port $CLICKHOUSE_PORT --user $CLICKHOUSE_USER --password $CLICKHOUSE_PASSWORD --query="INSERT INTO default.sessions_hourly_local FORMAT Native" < /clickhouse-0/backup/sessions_hourly_local.native

    clickhouse-client --host $CLICKHOUSE_HOST --port $CLICKHOUSE_PORT --user $CLICKHOUSE_USER --password $CLICKHOUSE_PASSWORD --query="INSERT INTO default.sessions_raw_local FORMAT Native" < /clickhouse-0/backup/sessions_raw_local.native

    clickhouse-client --host $CLICKHOUSE_HOST --port $CLICKHOUSE_PORT --user $CLICKHOUSE_USER --password $CLICKHOUSE_PASSWORD --query="INSERT INTO default.transactions_local FORMAT Native" < /clickhouse-0/backup/transactions_local.native
    ```

7. **Verify Data Import**:
   - Use the following SQL queries to confirm that the data has been successfully imported into the external ClickHouse instance:

    ```sql
    SELECT COUNT(*) FROM errors_local;
    SELECT COUNT(*) FROM outcomes_hourly_local;
    SELECT COUNT(*) FROM outcomes_raw_local;
    SELECT COUNT(*) FROM sessions_hourly_local;
    SELECT COUNT(*) FROM sessions_raw_local;
    SELECT COUNT(*) FROM transactions_local;
    ```

8. **Scale Up**:
   - Scale up all deployments and check if the application is running correctly with the newly imported data in the external ClickHouse.

---

## Upgrade from 11.4.0 to 15.3.0

1. **Increase Active Deadline for Jobs and CronJobs**:
   - The default `activeDeadlineSeconds` for jobs and cronjobs in the chart is too short (set to 100s). Increase the values as follows:

    ```yaml
    sentry:
      cleanup:
        activeDeadlineSeconds: 600
    hooks:
      activeDeadlineSeconds: 1800

    snuba:
      cleanupErrors:
        activeDeadlineSeconds: 600
      cleanupTransactions:
        activeDeadlineSeconds: 600
    ```

2. **Use Existing Secrets**:
   - Configure Sentry and PostgreSQL to use existing secrets for passwords.

    ```yaml
    user:
      existingSecret: "sentry-admin-password"
      existingSecretKey: "admin-password"

    externalPostgresql:
      existingSecret: sentry-postgresql
      existingSecretKeys:
        password: 'password'
    ```

---

## Upgrade from 15.3.0 to 16.0.0

1. **Check for Existing Sentry Secret**:
   - If the `sentry-sentry-secret` secret exists, reference it in the upgrade to avoid generating a new secret. Add the following to your configuration:

    ```yaml
    sentry:
      # To avoid generating a new sentry-secret, reference the existing secret
      existingSecret: "sentry-sentry-secret"
      existingSecretKey: "key"
    ```

---

## Upgrade from 16.0.0 to 17.0.0

1. **Scale Down Sentry Worker**:
   - Before upgrading, it’s recommended to scale down the `sentry-worker` to avoid errors during the process.

2. **Delete RabbitMQ Queues**:
   - In the `rabbitmq-0` pod, execute the following commands to delete specific queues. This will prevent the error `PRECONDITION_FAILED - inequivalent arg 'durable' for queue 'counters-0'`.

    ```bash
    rabbitmqctl delete_queue counters-0
    rabbitmqctl delete_queue triggers-0
    ```

---

## Upgrade from 17.0.0 to 18.0.0

1. **Remove Deprecated CronJobs**:
   - Remove `cleanupErrors` and `cleanupTransactions` cronjob values as they are no longer included in the new chart.

2. **Add Existing Secrets for Mail and ClickHouse**:
   - If using secrets for `mail` and `externalClickhouse`, add the following values to your configuration:

    ```yaml
    mail:
      existingSecret: sentry-mail-password
      existingSecretKey: password

    externalClickhouse:
      existingSecret: sentry-clickhouse
      existingSecretKey: admin-password
    ```

3. **Handle Kafka Topic Errors**:
   - If you encounter the error `arroyo.errors.ConsumerError: KafkaError{code=UNKNOWN_TOPIC_OR_PART,val=3,str="Subscribed topic not available: ingest-replay-recordings: Broker: Unknown topic or partition"}`, resolve it by creating the missing topic in the Kafka pod:

    ```bash
    kafka-topics.sh --create --topic <topic> --bootstrap-server 127.0.0.1:9092
    ```

    - For example, if the missing topic is `ingest-replay-recordings`, use:

    ```bash
    kafka-topics.sh --create --topic ingest-replay-recordings --bootstrap-server 127.0.0.1:9092
    ```
---

## Upgrade from 18.0.0 to 20.0.0

### Important Note:
- In version 20, the `snuba migrate` command will fail due to compatibility issues with the newer version of ClickHouse.
- If the Snuba migration job fails, remove the job (not the pod) so the upgrade can proceed. Alternatively, disable the migration job altogether by adding the following to your values file:

    ```yaml
    snubaMigrate:
      enabled: false
    ```

1. **Kafka Configuration**:
   - Ensure that ZooKeeper is still enabled for Kafka. Disable `kraft` for this version to continue using ZooKeeper.

    ```yaml
    kafka:
      kraft:
        enabled: false
      zookeeper:
        enabled: true
    ```

2. **Sentry Image Upgrade**:
   - After applying the changes, upgrade the Sentry image to a hard stop version to ensure compatibility:

    ```yaml
    images:
      relay:
        tag: "23.6.2"
      sentry:
        tag: "23.6.2"
      snuba:
        tag: "23.6.2"
    ```

---

## Upgrade from 20.0.0 to 20.12.1 (Using Hard-Stopped Image 23.11.0)

1. **Kafka Migration Considerations**:
   - If you're considering migrating from ZooKeeper to Kraft, it's recommended **not** to do so at this step. However, if you still decide to migrate, this step must be done before v22.
   - In v22, the `snuba-db-init` job will fail because the Kafka pod name will change and won't be reflected in the init job. You will need to find the current ZooKeeper cluster ID:

    ```bash
    kubectl exec -it <zookeeper-pod> -- zkCli.sh get /cluster/id
    ```

2. **Apply the Following Configuration for Kafka Migration**:

    ```yaml
    kafka:
      zookeeper:
        enabled: true
        metadata:
          migration:
            enable: true
      controller:
        replicaCount: 3
        controllerOnly: true
        zookeeperMigrationMode: true
      broker:
        replicaCount: 0
        zookeeperMigrationMode: true
      kraft:
        enabled: true
        clusterId: "<cluster_id>"
    ```

3. **Complete the Migration**:
   - Once all Kafka brokers have successfully migrated, set the `broker.zookeeperMigrationMode=false` to fully transition the brokers:

    ```yaml
    broker:
      zookeeperMigrationMode: false
    ```

4. **Turn Off Migration Mode and Stop ZooKeeper**:
   - To finalize the migration, switch off the migration mode on the Kafka controllers and stop ZooKeeper:

    ```yaml
    controller:
      zookeeperMigrationMode: false
    zookeeper:
      enabled: false
    ```
---

## Upgrade from 20.12.1 to 22.1.1

1. **Kafka Changes (Moving to Kraft)**:
   - Starting from version 22.1.1, ZooKeeper is no longer used for Kafka, and Kraft must be enabled.
   - Migrating Kafka from ZooKeeper to Kraft is not recommended, so it's better to install a fresh Kafka instance.
   
2. **Remove Kafka StatefulSet and Volumes**:
   - Update the Kafka configuration to disable ZooKeeper and set up Kraft for Kafka brokers.

    ```yaml
    kafka:
      provisioning:
        resources:
          limits:
            cpu: 1
          requests:
            cpu: 500m
      zookeeper:
        enabled: false
      controller:
        replicaCount: 3
        podAnnotations:
          ulimits.nri.containerd.io/container.kafka: |
            - type: RLIMIT_NOFILE
              hard: 65536
              soft: 65536
        resources:
          limits:
            cpu: 750m
            ephemeral-storage: 2Gi
            memory: 2Gi
          requests:
            cpu: 500m
            ephemeral-storage: 500Mi
            memory: 1Gi
      broker:
        replicaCount: 0
      kraft:
        enabled: true
        existingClusterIdSecret: sentry-kafka-kraft-cluster-id
    ```

3. **Apply the Changes**:
   - Once the changes are applied, Kafka will no longer use ZooKeeper.

4. **Restore Snuba Migrations**:
   - From this version, the Snuba migrations can be restored. Enable Snuba migrations by updating the configuration:

    ```yaml
    snubaMigrate:
      enabled: true
    ```

5. **Handle InProgress Migrations (if any)**:
   - If there were failed Snuba migration jobs in version 20, remove any `InProgress` migrations:
   
   - Deploy a manual Snuba migration pod:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        app: sentry
        release: sentry
        chart: sentry-22.1.1
      name: sentry-snuba-migrate-manual
    spec:
      containers:
      - name: snuba-migrate
        image: getsentry/snuba:24.2.0
        command: ["/bin/sh"]
        args: ["-c", "tail -f /dev/null"]
        env:
        - name: LOG_LEVEL
          value: debug
        - name: SNUBA_SETTINGS
          value: /etc/snuba/settings.py
        - name: DEFAULT_BROKERS
          value: sentry-kafka:9092
        - name: CLICKHOUSE_MAX_CONNECTIONS
          value: "10000"
        envFrom:
        - secretRef:
            name: sentry-snuba-env
        resources:
          limits:
            cpu: 2000m
            memory: 1Gi
          requests:
            cpu: 700m
            memory: 1Gi
        volumeMounts:
        - mountPath: /etc/snuba
          name: config
          readOnly: true
      restartPolicy: Never
      volumes:
      - configMap:
          name: sentry-snuba
        name: config
    ```

   - Enter the pod shell and check for the `InProgress` migrations:

    ```bash
    snuba migrations list
    ```

   - Get the `migration_id` and `group` and reverse the migration:

    ```bash
    snuba migrations reverse --group <group> --migration-id <migration_id>
    ```

   - For example:

    ```bash
    snuba migrations reverse --group querylog --migration-id 0001_querylog
    ```

6. **Re-run the Snuba Migrations**:
   - If the `activeDeadlineSeconds` limit was exceeded, re-run the Snuba migrations job, or use the manual pod to execute:

    ```bash
    snuba migrations migrate --force
    ```

7. **Fix Index Issues in Snuba Consumers**:
   - If you encounter errors like `ValueError: Found wrong number (2) of indexes for sentry_groupedmessage(project_id, id)` in some Snuba consumer pods, enter the ClickHouse database and run the following query to drop the problematic index:

    ```sql
    DROP INDEX sentry_groupedmessage_project_id_id_515aaa7e_uniq;
    ```

8. **PostgreSQL Error Handling**:
   - If PostgreSQL reports `duplicate key value violates unique constraint "sentry_environmentproject_project_id_29250c1307d3722b_uniq"`, you can safely ignore this error. This issue is tracked here: [Sentry Issue #8004](https://github.com/getsentry/sentry/issues/8004).

---

## Upgrade from 22.1.1 to 23.1.0

1. **Remove Ingest-Consumer Deployments**:
   - Before upgrading to 23.1.0, remove all `ingest-consumer` deployments as part of the cleanup process.

---

## Upgrade from 23.1.0 to 25.12.0

1. **Enable Redis-HA (High Availability) External**:
   - Configure external Redis-HA in Helm to provide high availability for Redis.

2. **Helmfile Configuration for Redis-HA**:
   - Add the Redis-HA configuration to your Helmfile:

    ```yaml
    - name: sentry-{{ .Environment.Name }}-redis
      namespace: {{ $namespace }}
      chart: dandydev/redis-ha
      version: 4.26.1
      wait: true
      timeout: 1800
      values:
        - helm/environments/redis.values.yaml
        - helm/environments/{{ .Environment.Name }}/redis.values.yaml
      secrets:
        - helm/environments/{{ .Environment.Name }}/redis.secrets.yaml
    ```

3. **Values Configuration for Redis-HA**:
   - Set the following values for Redis in your values file:

    ```yaml
    redis:
      enabled: false

    externalRedis:
      host: sentry-{{ .Environment.Name }}-redis-redis-ha-haproxy
      port: 6379
      # Check {{ .Environment.Name }}/sentry.secrets.yaml for the password
      password: ""
    ```

4. **Remove `defaultConfigurationOverrides` in Clickhouse**

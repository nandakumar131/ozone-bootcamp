# Ozone Bootcamp

## SCM

21 January 2025

| **Timings** | **Topic** |
| --- | --- |
| 09:30 - 10:00 | Initialization and Start Up |
| 10:00 - 10:30 | SCM Architecture |
| 10:30 - 11:00 | Node Lifecycle |
| 11:00 - 11:15 | Break |
| 11:15 - 11:30 | Pipeline Lifecycle |
| 11:30 - 11:45 | Container Lifecycle |
| 11:45 - 12:00 | Block Management |
| 12:00 - 12:30 | Safemode |
| 12:30 - 12:45 | Disk Layout |
| 12:45 - 13:45 | Lunch |
| 13:45 - 14:00 | High Availability |
| 14:00 - 14:15 | Decommissioning |
| 14:15 - 14:45 | Recover from two SCM failure |
| 14:45 - 15:15 | Datanode Heartbeat Protocol |
| 15:15 - 15:30 | Break |
| 15:30 - 16:00 | Container Replication |
| 16:00 - 16:30 | Container Balancer |
| 16:30 - 17:00 | Q & A |

# Initialization and Start Up

### SCM Initialization

- Why initialization?
- What happens if we don’t initialize?
    - Start SCM Without init

    ```bash
    ozone scm
    ```

- SCM initialization overview

![SCM-Initialization.drawio.png](flowcharts/scm-initialization.png)

- What is Primordial SCM?
    - Why do we need it?
        - Root Certificate to have chain of trust
        - Cluster ID Generation
- init vs bootstrap

### Exercise

- Run init on non-primordial SCM

    ```bash
    ozone scm --init
    ```

- Run bootstrap on primordial SCM

    ```bash
    ozone scm --bootstrap
    ```

- Run init on primordial SCM

    ```bash
    ozone scm --init
    ```

- Run bootstrap on non-primordial SCM without primordial SCM running

    ```bash
    ozone scm --bootstrap
    ```

- Start primordial SCM

    ```bash
    ozone --daemon start scm
    ```

- Run bootstrap on non-primordial SCM (with primordial SCM running)

    ```bash
    ozone scm --bootstrap
    ```

- Start non-primordial SCMs

    ```bash
    ozone --daemon start scm
    ```

- Start the datanodes

    ```bash
    ozone --daemon start datanode
    ```


Check the SCM service

```bash
ozone admin scm roles
```

The datanodes should get registered in SCMs

```bash
ozone admin datanode list
```

[http://127.0.0.1:9876/](http://127.0.0.1:9876/)

## OM Initialization

- How is it different from SCM initialization?
- init vs bootstrap on OM
- Initialize OM nodes

    ```bash
    ozone om --init
    ```

- Start OM service

    ```bash
    ozone --daemon start om
    ```


- Check the status

    ```bash
    ozone admin om roles
    ```

    [http://127.0.0.1:9874/](http://127.0.0.1:9874/)


## Problem

In a healthy cluster, the scm db directory and metadata directory got corrupted (or got deleted) on the primordial SCM.

The SCM process will crash.

What will happen during restart?

What will be the behaviour in CM deployed Ozone?

How to fix and make the cluster healthy?

# SCM Architecture

What does SCM manage?

![SCM.png](flowcharts/scm.png)

How does it manage?

![SCM Storage.drawio.png](flowcharts/scm-storage.png)

# Metadata

### SCM RocksDB Tables

Pipeline Table

```bash
ozone debug ldb --db=/data/metadata/scm.db value-schema --cf=pipelines | jq '.Pipeline |  keys'
```

Container Table

```bash
ozone debug ldb --db=/data/metadata/scm.db value-schema --cf=containers | jq '.ContainerInfo |  keys'
```

Deleted Blocks Table

```bash
ozone debug ldb --db=/data/metadata/scm.db value-schema --cf=deletedBlocks | jq '.DeletedBlocksTransaction |  keys'
```

## Viewing in-memory data vs data stored in the scm db

---

To view the metadata, we have to generate some first.

```bash
ozone sh volume create /vol
ozone sh bucket create /vol/buck
ozone sh key put
ozone sh key put /vol/buck/CONTRIBUTING.md  CONTRIBUTING.md
ozone sh key list /vol/buck
ozone sh key cat /vol/buck/CONTRIBUTING.md
```

Let’s query and check the metadata now

### Pipeline Information

In-Memory

```bash
ozone admin pipeline list
```

In Database

```bash
ozone debug ldb --db=/data/metadata/scm.db scan --cf=pipelines
```

### Container Information

In-Memory

```bash
ozone admin container list
```

In Database

```bash
ozone debug ldb --db=/data/metadata/scm.db scan --cf=containers
```

Why is there no data in the db?

### Node Information

We don’t store node information in db

In-Memory

```bash
ozone admin datanode list
```

### Block Information

We only store deleted block information until it’s purged. We don’t cache this information in-memory

```bash
ozone debug ldb --db=/data/metadata/scm.db scan --cf=deletedBlocks
```

### Problem

Write a key into the cluster and delete it. It should now be present in `deletedBlocks` table.

Query and check the entry in the table.

([https://issues.apache.org/jira/browse/HDDS-11643](https://issues.apache.org/jira/browse/HDDS-11643))

![SCM Architecture.drawio.png](flowcharts/scm-architecture.png)

## End Points

Block Protocol Server

```bash
curl -s http://127.0.0.1:9876/jmx | grep -B3 -A 32 ScmBlockLocationProtocolService
```

Container Protocol Server

```bash
curl -s http://127.0.0.1:9876/jmx | grep -B3 -A 32 StorageContainerLocationProtocolService
```

Datanode Protocol Server

```bash
curl -s http://127.0.0.1:9876/jmx | grep -B3 -A 32 StorageContainerDatanodeProtocolService
```

Certificate Server

```bash
curl -s http://127.0.0.1:9876/jmx | grep -B3 -A 32 SCMSecurityProtocolService
```

[http://127.0.0.1:9876/#!/metrics/rpc](http://127.0.0.1:9876/#!/metrics/rpc)

## Services

Pipeline Creator Service

```bash
jstack <scm pid> | grep BackgroundPipelineCreator
```

Replication Manager

```bash
jstack <scm pid> | grep ReplicationManager
```

Block Deletion Service

```bash
jstack <scm pid> | grep SCMBlockDeletingService
```

# Node Lifecycle

## Node State Transition
![node-state-transition.drawio.png](flowcharts/node-state-transition.png)

## Node State Flow
![node-state-flow.drawio.png](flowcharts/node-state-flow.png)

### Note
This setup has the following configuration changes
```bash
ozone.scm.stale.node.interval is 1m
ozone.scm.dead.node.interval is 2m
```
You can check the configs in http://127.0.0.1:9876/#!/config


- What is `HEALTHY_READONLY` state?
    - Used during upgrade
    - The datanodes are moved to this state when SCM is finalized but the datanodes are yet to be finalized.

- What happens when a datanode is marked as stale?
- What happens when a datanode is marked as dead?


## Exercise
Putting Datanode into `HEALTHY_READONLY` state.

Run the datanode list command to check the current state of the datanode
```bash
ozone admin datanode list
```
Stop the datanode process and manually modify the `layoutVersion` in the `VERSION` file
```bash
ozone --daemon stop datanode
vi /data/metadata/dnlayoutversion/VERSION
ozone --daemon start datanode
```

Run the datanode list command to check the current state of the datanode
```bash
ozone admin datanode list
```

Check if the upgrade finalization code was executed
```bash
grep -i finalize /var/log/hadoop/ozone-hadoop-datanode*.log
```

If a datanode is stuck in `HEALTHY_READONLY` state, it means that the datanode is not able to run upgrade finalization.
Check the datanode logs for more details.


### What is `Operational State`?
- IN_SERVICE
- ENTERING_MAINTENANCE
- IN_MAINTENANCE
- DECOMMISSIONING
- DECOMMISSIONED

Operational State along with Health State will give the overall state of the datanode.

# Pipeline Lifecycle

![PipelineStatemachine.drawio.png](flowcharts/pipeline-statemachine.png)

Pipeline failure

# Container Lifecycle

![container-state-transition.drawio.png](flowcharts/container-state-transition.png)

![container-state-flow.drawio.png](flowcharts/container-state-flow.png)

TODO: Container creation logic

Quasi closed to close — Origin Node ID logic

# Block Management

Block Allocation

Block Deletion - Deletion Ack

Delete Block retry count - How to reset the retry count for txns

Get the count of pending block delete transactions from SCM db

# Safemode

## Pre-check

- What is pre-check?
- What problem does pre-check logic solve?

## Container Safemode Rule

At least one replica of all the CLOSED/QUASI_CLOSED containers should be reported to SCM for this rule validation to succeed.

## Datanode Safemode Rule

## Healthy Pipeline Safemode Rule

## One Replica Pipeline Safemode Rule

### OM Operations

| **Client Operation** | **Expected Behaviour** |
| --- | --- |
| Volume Create | Should Succeed. |
| Bucket Create | Should Succeed. |
| Volume List | Should Succeed. |
| Bucket List | Should Succeed. |
| Key/File List | Should Succeed. |
| Key/File Write | Should fail with Safemode Exception |
| Key/File Read | It should succeed if at least one DN holding the replica is registered. |

### SCM Admin Operations

| **Admin Command** | **Expected Behaviour** |
| --- | --- |
| Close Container | Should Succeed. The container will be moved to Closing state |
| CreatePipeline | Should fail with Safemode Exception |
| GetPipeline | Should Succeed. |

```bash
ozone admin safemode status --verbose
```

Check SCM Web UI for details.

### Problem

Try to put cluster into safemode and verify the client commands.

# Disk Layout

![scm-metadata.png](flowcharts/scm-metadata.png)

Cat the version file

Read Raft Log using ldb tool

# High Availability

![HA Architecture.drawio.png](flowcharts/ha-architecture.png)

There is not much difference in HA and Non-HA code path in SCM. We have single node Ratis for Non-HA setup.

Datanodes seen by SCMs might vary if there are network partitions.

# Decommissioning

What happens if you remove node without decommissioning and add new nodes?

# Recover from two SCM failure

Ratis log parser

# Datanode Heartbeat Protocol

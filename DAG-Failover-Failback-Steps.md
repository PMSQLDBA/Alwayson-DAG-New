Since our lab is on SQL Server 2022/16.x, use the SQL Server 2022+ Distributed AG failover procedure. 

Microsoft documents that Distributed AG failover is manual and uses `FORCE_FAILOVER_ALLOW_DATA_LOSS`; 

for a planned no-data-loss failover, we first synchronize the Distributed AG and use `REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT = 1` before switching roles. ([Microsoft Learn][1])

Our current logical direction is:

```text
AGAlpha
PMSQL01 = Global Primary
PMSQL02 = Local Secondary
        |
        | AGDistributed
        v
AGBeta
PMSQL03 = Forwarder
PMSQL04 = Local Secondary
```

A planned failover moves the Distributed AG from **AGAlpha → AGBeta**. 

Failback uses the same process in reverse. ([Microsoft Learn][1])

1. **Verify the current roles first.** Run this on PMSQL01 and PMSQL03:

```sql
SELECT
    ag.name AS AGName,
    ar.replica_server_name,
    ars.role_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar
    ON ag.group_id = ar.group_id
LEFT JOIN sys.dm_hadr_availability_replica_states ars
    ON ar.replica_id = ars.replica_id
WHERE ag.name IN ('AGAlpha','AGBeta','AGDistributed');
GO
```

Before failover, you should conceptually have:

```text
AGAlpha        = Global Primary side
AGBeta         = Forwarder side
AGDistributed  = Healthy
```

Do not continue if the local AGs or Distributed AG are unhealthy. 

Microsoft recommends verifying the member AGs and Distributed AG synchronization before role transition. ([Microsoft Learn][1])

2. **Confirm local AG primary replicas use synchronous commit.** Run on the current primary of AGAlpha and the current primary of AGBeta:

```sql
SELECT DISTINCT
    ag.name AS [Availability Group],
    ar.replica_server_name AS [Replica],
    ar.availability_mode_desc AS [Availability Mode]
FROM sys.availability_replicas AS ar
JOIN sys.availability_groups AS ag
    ON ar.group_id = ag.group_id
JOIN sys.dm_hadr_database_replica_states AS rs
    ON ar.group_id = rs.group_id
    AND ar.replica_id = rs.replica_id
WHERE ag.name IN ('AGAlpha','AGBeta')
  AND rs.is_primary_replica = 1
ORDER BY [Availability Group];
GO
```

For your planned failover, the current AGAlpha and AGBeta primaries should report:

```text
SYNCHRONOUS_COMMIT
```

Microsoft says the global primary and forwarder should be synchronous before performing the no-data-loss procedure. ([Microsoft Learn][1])

3. **Temporarily change the Distributed AG link to synchronous commit.** Run this on **both PMSQL01 and PMSQL03**:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
MODIFY AVAILABILITY GROUP ON
'AGAlpha' WITH
(
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT
),
'AGBeta' WITH
(
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT
);
GO
```

This has to be issued on both sides because Distributed AG synchronization depends on both member AG definitions using synchronous commit. ([Microsoft Learn][1])

4. **Wait until the Distributed AG reports SYNCHRONIZED.** Run on the current global primary, PMSQL01:

```sql
SELECT
    ag.name,
    DB_NAME(drs.database_id) AS database_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.last_hardened_lsn
FROM sys.dm_hadr_database_replica_states AS drs
JOIN sys.availability_groups AS ag
    ON drs.group_id = ag.group_id
WHERE ag.name = 'AGDistributed'
ORDER BY database_name;
GO
```

Do not continue until `America` shows:

```text
synchronization_state_desc = SYNCHRONIZED
```

Microsoft explicitly says to wait for the Distributed AG to reach `SYNCHRONIZED` before moving to the next step. ([Microsoft Learn][1])

5. **Require the secondary AG to harden commits.** Run on the current global primary, PMSQL01:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
SET
(
    REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT = 1
);
GO
```

For SQL Server 2022+, this setting makes the global primary wait for the synchronized secondary side, which is the mechanism Microsoft uses in the planned no-data-loss Distributed AG failover procedure. 

It can reduce performance while enabled. ([Microsoft Learn][1])

6. **Run one final health check.** Run on PMSQL01 and PMSQL03:

```sql
SELECT
    ag.name AS [AG Name],
    DB_NAME(drs.database_id) AS database_name,
    ar.replica_server_name AS replica,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.last_hardened_lsn
FROM sys.dm_hadr_database_replica_states AS drs
JOIN sys.availability_groups AS ag
    ON drs.group_id = ag.group_id
JOIN sys.availability_replicas AS ar
    ON drs.replica_id = ar.replica_id
WHERE ag.name IN ('AGAlpha','AGBeta','AGDistributed')
ORDER BY ag.name, database_name, replica;
GO
```

We want the relevant replicas healthy and the Distributed AG synchronized before changing roles. ([Microsoft Learn][1])

7. **Demote AGAlpha from global primary.** Run this on PMSQL01:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
SET
(
    ROLE = SECONDARY
);
GO
```

At this point the Distributed AG becomes temporarily unavailable. 

Microsoft warns that after this step you should continue through the remaining failover steps rather than abandoning the procedure midway. ([Microsoft Learn][1])

Our state is briefly:

```text
AGAlpha
PMSQL01
   |
   | SECONDARY role
   |
AGDistributed
TEMPORARILY UNAVAILABLE

AGBeta
PMSQL03
waiting to become Global Primary
```

8. **Perform the actual DAG failover on PMSQL03.** Connect to PMSQL03, which is the forwarder and intended new global primary:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
FORCE_FAILOVER_ALLOW_DATA_LOSS;
GO
```

Despite the command name, Microsoft uses this same command for Distributed AG planned failover after the synchronization safeguards above have been completed. After the command succeeds, AGBeta becomes the global primary side and AGAlpha becomes the forwarder side. ([Microsoft Learn][1])

Your new topology is:

```text
             AGDistributed
                    |
       +------------+------------+
       |                         |
     AGBeta                    AGAlpha
 Global Primary              Forwarder
       |                         |
    PMSQL03                   PMSQL01
       |                         |
    PMSQL04                   PMSQL02
```

9. **Clear the synchronized-secondary requirement on the new forwarder.** Run this on PMSQL01, which is now the forwarder side:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
SET
(
    REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT = 0
);
GO
```

Microsoft specifically calls for clearing this setting on the previous global primary after the role transition. ([Microsoft Learn][1])

10. **Optionally change the Distributed AG back to asynchronous commit.** Since our lab normally uses asynchronous replication between AGAlpha and AGBeta, run this on **both PMSQL03 and PMSQL01**:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
MODIFY AVAILABILITY GROUP ON
'AGAlpha' WITH
(
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT
),
'AGBeta' WITH
(
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT
);
GO
```

Microsoft recommends this as an optional post-failover step when asynchronous mode is preferred for the inter-AG link. ([Microsoft Learn][1])

11. **Validate that AGBeta is now the global primary side.** Run on PMSQL03:

```sql
SELECT
    ag.name AS AGName,
    ar.replica_server_name,
    ars.role_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar
    ON ag.group_id = ar.group_id
LEFT JOIN sys.dm_hadr_availability_replica_states ars
    ON ar.replica_id = ars.replica_id
WHERE ag.name = 'AGDistributed';
GO
```

Also verify `America` is available on the AGBeta side:

```sql
SELECT
    DB_NAME(database_id) AS DatabaseName,
    synchronization_state_desc,
    synchronization_health_desc,
    database_state_desc,
    is_primary_replica
FROM sys.dm_hadr_database_replica_states
WHERE DB_NAME(database_id) = 'America';
GO
```

At this point, our Distributed AG has successfully failed over from AGAlpha to AGBeta. ([Microsoft Learn][1])

12. **To fail back from AGBeta → AGAlpha, perform the same process in reverse.** PMSQL03 is now the global primary and PMSQL01 is the forwarder.

First change the Distributed AG to synchronous on **both PMSQL03 and PMSQL01**, wait for `SYNCHRONIZED`, then on PMSQL03 set:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
SET
(
    REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT = 1
);
GO
```

Demote the current global primary on PMSQL03:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
SET
(
    ROLE = SECONDARY
);
GO
```

Then connect to PMSQL01 and fail the Distributed AG back:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
FORCE_FAILOVER_ALLOW_DATA_LOSS;
GO
```

Now PMSQL01/AGAlpha becomes global primary again. On PMSQL03, which becomes the forwarder, clear the setting:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
SET
(
    REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT = 0
);
GO
```

Finally, if desired, change the inter-AG relationship back to `ASYNCHRONOUS_COMMIT` on both sides. 

Microsoft states that the same Distributed AG failover command is used for both failover to the forwarder and failback to the previous global-primary side. ([Microsoft Learn][1])

For our lab, the safe operational sequence to remember is:

```text
NORMAL

AGAlpha                     AGBeta
Global Primary ------------> Forwarder
PMSQL01                      PMSQL03


FAILOVER PREP

ASYNC
  ↓
SYNC
  ↓
Wait SYNCHRONIZED
  ↓
REQUIRED_SYNC_SECONDARIES = 1
  ↓
AGAlpha ROLE = SECONDARY
  ↓
Failover on PMSQL03


AFTER FAILOVER

AGAlpha                     AGBeta
Forwarder <---------------- Global Primary
PMSQL01                      PMSQL03


FAILBACK

Repeat the exact process in reverse.
```

**Avoid running `FORCE_FAILOVER_ALLOW_DATA_LOSS` by itself for a planned failover.** 

Microsoft supports that direct command for emergencies where data loss is acceptable, but the synchronization steps above are what make the SQL Server 2022+ planned procedure suitable when you want to avoid data loss. ([Microsoft Learn][1])

[1]: https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/configure-distributed-availability-groups?view=sql-server-ver17 "Configure a Distributed Availability Group - SQL Server Always On | Microsoft Learn"

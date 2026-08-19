## Overview of PMLABS Setup:

<img width="1536" height="1024" alt="ChatGPT Image Aug 18, 2026, 10_13_46 PM" src="https://github.com/user-attachments/assets/3df4607e-8906-4bde-8732-7ba0a1fdebac" />

Built a **multi-node SQL Server High Availability and Disaster Recovery lab** using Windows Server, Active Directory, Windows Server Failover Clustering, SQL Server Always On Availability Groups, and a Distributed Availability Group.

At a high level, the architecture looks like this:

```text
                         PMLABS.COM
                             |
                            PMDC
                       192.168.0.100
                     AD DS + DNS + Share
                             |
                   192.168.0.0/24 Network
                             |
          +------------------+------------------+
          |                                     |
       CLUSTER A                             CLUSTER B
        WSFC-A                                WSFC-B
          |                                     |
    +-----+-----+                         +-----+-----+
    |           |                         |           |
 PMSQL01     PMSQL02                   PMSQL03     PMSQL04
 .101          .102                       .103        .104
    |           |                         |           |
    +--AGAlpha--+                         +--AGBeta---+
          |                                     |
       America                               America
          |                                     |
          +---------- AGDistributed ------------+
```

The design gives you **two levels of availability**.

**Local High Availability:**

```text
AGAlpha
PMSQL01 <----> PMSQL02

AGBeta
PMSQL03 <----> PMSQL04
```

**Site/cluster-level Disaster Recovery:**

```text
AGAlpha
   |
   | Distributed Availability Group
   |
AGBeta
```

---

## Core Components

### Active Directory and DNS

`PMDC` at `192.168.0.100` provides the central identity and DNS infrastructure for the lab.

```text
Domain:  pmlabs.com
NetBIOS: PMLABS
DNS:     192.168.0.100
```

This gives all SQL and cluster nodes a common security boundary.

For example:

```text
PMLABS\pmadmin
PMLABS\sqlsvc
PMLABS\sqlagent
```

can be centrally managed rather than creating unrelated local accounts on every server.

---

### Windows Server Failover Clusters

You created two independent Windows clusters.

```text
WSFC-A
  |
  +-- PMSQL01
  +-- PMSQL02


WSFC-B
  |
  +-- PMSQL03
  +-- PMSQL04
```

This is important because the Distributed AG connects **two independent Availability Groups**, which can belong to separate WSFC clusters.

It is much closer to a real HA/DR architecture than putting all four SQL Servers into one cluster.

---

## AGAlpha

Your first local Availability Group is:

```text
                AGAlpha
                   |
        +----------+----------+
        |                     |
     PMSQL01               PMSQL02
     PRIMARY               SECONDARY
        |                     |
        +------ America ------+
```

Listener:

```text
AGAlpha-LST.pmlabs.com
192.168.0.121
```

Applications should normally connect through the listener instead of directly to:

```text
PMSQL01
```

That means the application connection target can remain stable even when the AG primary moves between nodes.

---

## AGBeta

Your second Availability Group is:

```text
                 AGBeta
                   |
        +----------+----------+
        |                     |
     PMSQL03               PMSQL04
     PRIMARY               SECONDARY
        |                     |
        +------ America ------+
```

Listener:

```text
AGBETA-LST.pmlabs.com
192.168.0.131
```

PMSQL03 also acts as the **forwarder** for the Distributed Availability Group.

---

# Distributed Availability Group

This is the most important architectural layer in your lab.

Instead of directly creating one four-node AG, you have:

```text
             AGDistributed
                   |
          +--------+--------+
          |                 |
       AGAlpha            AGBeta
          |                 |
      Cluster A          Cluster B
```

The database transaction flow is conceptually:

```text
                    America
                       |
                       v
                    PMSQL01
                       |
                       +----> PMSQL02
                       |
                       |
              Distributed AG
                       |
                       v
                    PMSQL03
                       |
                       +----> PMSQL04
```

So you effectively have copies of `America` across four SQL Servers.

---

# Why This Architecture Is Useful

The biggest advantage is that **HA and DR are separated into different layers**.

### Local HA

Suppose PMSQL01 fails.

You don't immediately need the second cluster.

Instead:

```text
Before:

PMSQL01  PRIMARY
PMSQL02  SECONDARY


After local failover:

PMSQL01  unavailable
PMSQL02  PRIMARY
```

AGAlpha continues operating inside Cluster A.

That is **High Availability**.

---

### Cluster/Site-Level DR

Now imagine a much larger problem:

```text
PMSQL01   DOWN
PMSQL02   DOWN

or

WSFC-A completely unavailable
```

You still have:

```text
WSFC-B

PMSQL03
PMSQL04

AGBeta
America
```

The Distributed AG provides the mechanism for moving the global primary role toward AGBeta according to your planned DR procedure.

That is **Disaster Recovery**.

---

# Failure Protection

Your architecture protects against several different failure scenarios.

| Failure                               | Protection                 |
| ------------------------------------- | -------------------------- |
| SQL Server service failure on PMSQL01 | AGAlpha                    |
| PMSQL01 VM failure                    | PMSQL02                    |
| PMSQL02 VM failure                    | PMSQL01                    |
| PMSQL03 VM failure                    | PMSQL04                    |
| PMSQL04 VM failure                    | PMSQL03                    |
| Cluster A problem                     | AGBeta / Distributed AG    |
| Cluster B problem                     | AGAlpha continues          |
| Planned SQL maintenance               | AG failover                |
| Windows patching                      | Rolling node maintenance   |
| SQL patching                          | Rolling node maintenance   |
| Primary AG/site outage                | Distributed AG DR strategy |

This is one reason the architecture is valuable for learning enterprise SQL Server operations.

---

# Synchronous vs Asynchronous Replication

Your architecture can use different replication strategies at the two levels.

### Inside each local AG

For nodes that are close together:

```text
PMSQL01 <-- SYNCHRONOUS --> PMSQL02

PMSQL03 <-- SYNCHRONOUS --> PMSQL04
```

The goal is local HA and, when healthy/synchronized, planned or configured automatic failover.

### Between the two AGs

Your Distributed AG starts with:

```text
AGAlpha ---- ASYNCHRONOUS ----> AGBeta
```

This is useful for DR because the second AG could eventually be in another network, datacenter, region, or location where network latency is higher.

It prevents the primary workload from having to wait for every transaction to reach the remote DR environment before committing.

The tradeoff is that asynchronous DR can have **potential data loss during an unplanned disaster** if the remote side has not received all log records.

---

# Listener Advantage

Without an AG listener, applications might connect directly to:

```text
PMSQL01
```

If PMSQL01 stops being primary, that creates a client-routing problem.

Instead, applications use:

```text
AGAlpha-LST.pmlabs.com
```

DNS:

```text
AGAlpha-LST.pmlabs.com
        |
        v
192.168.0.121
```

The listener belongs to the Availability Group rather than permanently belonging to PMSQL01.

That gives applications a stable SQL connection name.

The same concept exists for AGBeta:

```text
AGBETA-LST.pmlabs.com
192.168.0.131
```

---

# Dedicated SQL Service Accounts

You also separated administration from service execution.

```text
PMLABS\pmadmin
        |
        +---- Administration


PMLABS\sqlsvc
        |
        +---- SQL Database Engine


PMLABS\sqlagent
        |
        +---- SQL Server Agent
```

This is much better than running everything using Domain Administrator.

It helps with:

* least privilege
* auditing
* service isolation
* password/service management
* endpoint authentication
* network resource access
* troubleshooting

It also teaches the account model you are much more likely to see in enterprise SQL Server environments.

---

# Shared Backup Location

You introduced:

```text
\\PMDC\sharedbkps
```

This was useful for manually seeding the replicas:

```text
AGAlpha Primary
      |
      | FULL + LOG backup
      v
\\PMDC\sharedbkps
      |
      +----------> PMSQL02
      |
      +----------> PMSQL03
      |
      +----------> PMSQL04
```

This also exposed an important SQL Server administration concept:

**the SQL Server service account must be able to access the network backup location.**

Your interactive Windows administrator permissions alone are not sufficient for SQL Server backup/restore operations to a UNC path.

---

# Separate SQL and HADR Traffic Concepts

Another valuable part of this lab was learning the difference between:

```text
TCP 1433
SQL client connections
```

and:

```text
TCP 5022
Always On / HADR endpoint communication
```

For example:

```text
Test-NetConnection PMSQL02 -Port 5022
```

returning `True` does **not** prove that SSMS can connect to SQL Server.

You separately test:

```text
Test-NetConnection PMSQL02 -Port 1433
```

This distinction is extremely useful when troubleshooting Always On.

---

# Operational Advantage

The environment now allows you to practice much more than installation.

You can simulate:

```text
Normal operation
      ↓
Local AG failover
      ↓
Node failure
      ↓
Windows patching
      ↓
SQL patching
      ↓
Listener connectivity
      ↓
Replica synchronization problems
      ↓
Endpoint failures
      ↓
Network failures
      ↓
Cluster failures
      ↓
Distributed AG failover
      ↓
DR operation
      ↓
Failback
```

That makes the lab particularly useful because the next stage is **operations**, not installation.

---

# HA vs DR in Your Environment

A useful way to remember your design is:

```text
                 HIGH AVAILABILITY

        PMSQL01 <------------> PMSQL02
                    AGAlpha

        PMSQL03 <------------> PMSQL04
                    AGBeta


                 DISASTER RECOVERY

                    AGAlpha
                       |
                       |
                 AGDistributed
                       |
                       |
                     AGBeta
```

So:

**AGAlpha and AGBeta provide local HA.**

**AGDistributed provides the higher-level DR relationship between the two AGs.**

---

# Main Advantages of Your Lab

Your architecture gives you a practical environment for learning:

1. **Server-level HA** — losing one SQL node does not necessarily mean losing the SQL service.
2. **Database-level replication** — `America` is replicated using SQL Server transaction-log transport rather than shared database disks.
3. **Cluster-level DR** — the second AG can protect against losing the first AG/cluster.
4. **Stable client connectivity** — listeners abstract applications from individual SQL Server names.
5. **Independent clusters** — AGAlpha and AGBeta can be operated and maintained separately.
6. **Maintenance flexibility** — you can practice moving workloads before Windows/SQL maintenance.
7. **Realistic security** — domain admin, SQL Engine, SQL Agent, and administrative identities are separated.
8. **Real troubleshooting practice** — DNS, AD, WSFC, SQL ports, endpoints, service accounts, permissions, backups, LSN chains, listeners and HADR all interact.
9. **Future multi-site expansion** — the same logical pattern can be adapted to separate networks/sites rather than keeping everything on `192.168.0.0/24`.
10. **DR testing** — you can now practice planned Distributed AG failover and failback without using a production system.

The biggest architectural lesson from the setup is:

```text
        HA                           DR
         |                            |
Individual Server             Entire AG / Cluster
   Failure                       Failure
         |                            |
         v                            v
Local Availability           Distributed
     Group                 Availability Group
```

That separation of **local HA from broader DR** is the key reason your two-AG + Distributed AG design is much more useful for advanced SQL Server practice than simply building a basic two-node Availability Group.

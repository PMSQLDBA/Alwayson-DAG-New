# PMLABS Windows Server 2025 + WSFC + SQL Server Always On + Distributed AG Lab

> End-to-end implementation runbook for the VMware Workstation lab built in this conversation.

<img width="1536" height="1024" alt="ChatGPT Image Aug 18, 2026, 10_13_46 PM" src="https://github.com/user-attachments/assets/eb7c3cb5-8c4b-440a-9d82-e1f74dd4fccd" />

---

## 1. Scope

This README documents the complete lab build from an empty VMware Workstation environment through:

- Windows Server 2025 Evaluation VMs
- Active Directory Domain Services and DNS
- Domain administration and SQL service accounts
- Four SQL Server nodes
- Two independent Windows Server Failover Clusters
- Two SQL Server Always On Availability Groups
- One Distributed Availability Group
- Database: `America`
- Shared backup location: `\\PMDC\sharedbkps`

The final design is:

```text
                               PMLABS.COM
                                   |
                                  PMDC
                             192.168.0.100
                           AD DS + DNS + Share
                                   |
                         192.168.0.0/24 LAN
                                   |
             +---------------------+---------------------+
             |                                           |
           WSFC-A                                      WSFC-B
             |                                           |
      +------+------+                             +------+------+
      |             |                             |             |
   PMSQL01       PMSQL02                       PMSQL03       PMSQL04
192.168.0.101 192.168.0.102                 192.168.0.103 192.168.0.104
      |             |                             |             |
      +--- AGAlpha--+                             +--- AGBeta---+
             |                                           |
          America                                     America
             |                                           |
             +----------- AGDistributed -----------------+
```

---

# 2. Final Lab Values

## 2.1 Domain and network

| Item | Value |
|---|---|
| AD DNS Domain | `pmlabs.com` |
| NetBIOS Domain | `PMLABS` |
| Subnet | `192.168.0.0/24` |
| Subnet Mask | `255.255.255.0` |
| Gateway | `192.168.0.1` |
| DNS Server | `192.168.0.100` |
| Local admin workstation seen during lab | `192.168.0.73` |

## 2.2 Servers

| Server | IP | Purpose |
|---|---:|---|
| PMDC | `192.168.0.100` | AD DS, DNS, backup share |
| PMSQL01 | `192.168.0.101` | WSFC-A / AGAlpha primary |
| PMSQL02 | `192.168.0.102` | WSFC-A / AGAlpha secondary |
| PMSQL03 | `192.168.0.103` | WSFC-B / AGBeta primary / DAG forwarder |
| PMSQL04 | `192.168.0.104` | WSFC-B / AGBeta secondary |

## 2.3 Cluster and AG naming

| Component | Name | Lab IP |
|---|---|---:|
| Windows Cluster A | `WSFC-A` | `192.168.0.110`* |
| AGAlpha listener | `AGAlpha-LST.pmlabs.com` | `192.168.0.121` |
| Windows Cluster B | `WSFC-B` | `192.168.0.120`* |
| AGBeta listener | `AGBETA-LST.pmlabs.com` | `192.168.0.131` |
| Distributed AG | `AGDistributed` | N/A |
| Database | `America` | N/A |

`*` The WSFC cluster-management IPs above were the planned lab values. Verify the actual cluster IPs in Failover Cluster Manager or PowerShell before relying on them.

## 2.4 Accounts

| Account | Purpose |
|---|---|
| `PMLABS\Administrator` | Domain-level emergency/admin work |
| `PMLABS\pmadmin` | Day-to-day Windows/cluster/SQL administration |
| `PMLABS\sqlsvc` | SQL Server Database Engine service |
| `PMLABS\sqlagent` | SQL Server Agent service |
| `PMLABS\PMLAB-Server-Admins` | Member-server local administrator group |
| `PMLABS\PMLAB-SQL-Admins` | SQL Server administrative group |

---

# 3. VMware Workstation Design

For the flat, single-subnet lab, use one bridged NIC per VM.

For each VM:

1. Shut down the VM.
2. Open **VM > Settings > Network Adapter**.
3. Select **Bridged: Connected directly to the physical network**.
4. Enable:
   - Connected
   - Connect at power on
5. Ensure VMware `VMnet0` is bridged to the physical NIC that reaches `192.168.0.0/24`.

Recommended VM sizing for a lab:

```text
PMDC:
  2 vCPU
  4 GB RAM minimum
  60 GB disk

PMSQL01-04:
  2-4 vCPU
  4-8 GB RAM minimum
  60+ GB disk
```

If the host has limited memory, reduce SQL max server memory later rather than starving Windows.

---

# 4. Build PMDC

## 4.1 Configure static networking

Target:

```text
Host:    PMDC
IP:      192.168.0.100
Mask:    255.255.255.0
Gateway: 192.168.0.1
DNS:     192.168.0.100
```

Verify:

```powershell
ipconfig /all
Get-NetIPConfiguration
Get-DnsClientServerAddress -AddressFamily IPv4
```

Set PMDC DNS explicitly:

```powershell
Set-DnsClientServerAddress `
  -InterfaceAlias "Ethernet0" `
  -ServerAddresses 192.168.0.100
```

Avoid multiple active NICs with competing gateways/DNS servers in this simple lab.

---

# 5. Install AD DS and DNS

Run on PMDC in elevated PowerShell:

```powershell
Get-WindowsFeature AD-Domain-Services,DNS
```

Install:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

Verify:

```powershell
Get-WindowsFeature AD-Domain-Services,DNS
```

Expected:

```text
[X] Active Directory Domain Services
[X] DNS Server
```

---

# 6. Create the PMLABS Forest

Run:

```powershell
Import-Module ADDSDeployment
```

Create the forest:

```powershell
Install-ADDSForest `
-DomainName "pmlabs.com" `
-DomainNetbiosName "PMLABS" `
-InstallDNS:$true `
-CreateDnsDelegation:$false `
-DatabasePath "C:\Windows\NTDS" `
-LogPath "C:\Windows\NTDS" `
-SysvolPath "C:\Windows\SYSVOL" `
-Force:$true
```

Enter a strong DSRM password when prompted.

A DNS delegation warning is expected for this self-contained lab when there is no authoritative parent zone delegation.

If PMDC does not reboot automatically:

```powershell
Restart-Computer -Force
```

After reboot, sign in as:

```text
PMLABS\Administrator
```

---

# 7. Validate AD and DNS

Run on PMDC:

```powershell
Get-ADDomain
Get-ADForest
Get-ADDomainController
Get-Service NTDS,DNS
```

DNS tests:

```powershell
nslookup pmlabs.com
nslookup PMDC.pmlabs.com
nslookup -type=SRV _ldap._tcp.dc._msdcs.pmlabs.com
```

Health tests:

```powershell
dcdiag
dcdiag /test:dns
```

Do not continue to clustering until AD/DNS is healthy.

---

# 8. Create Administrative Groups and Accounts

## 8.1 Create `pmadmin`

```powershell
$Password = Read-Host "Enter password for pmadmin" -AsSecureString

New-ADUser `
  -Name "PM Admin" `
  -SamAccountName "pmadmin" `
  -UserPrincipalName "pmadmin@pmlabs.com" `
  -AccountPassword $Password `
  -Enabled $true
```

For a lab you may choose `PasswordNeverExpires`, but production policy should govern that setting.

## 8.2 Create server admin group

```powershell
New-ADGroup `
  -Name "PMLAB-Server-Admins" `
  -SamAccountName "PMLAB-Server-Admins" `
  -GroupScope Global `
  -GroupCategory Security

Add-ADGroupMember `
  -Identity "PMLAB-Server-Admins" `
  -Members "pmadmin"
```

## 8.3 Create SQL admin group

```powershell
New-ADGroup `
  -Name "PMLAB-SQL-Admins" `
  -SamAccountName "PMLAB-SQL-Admins" `
  -GroupScope Global `
  -GroupCategory Security

Add-ADGroupMember `
  -Identity "PMLAB-SQL-Admins" `
  -Members "pmadmin"
```

Do not make `sqlsvc` or `sqlagent` Domain Admins.

---

# 9. Create SQL Service Accounts

## 9.1 SQL Database Engine service

```powershell
$SqlSvcPassword = Read-Host "Enter password for SQL Server service account" -AsSecureString

New-ADUser `
  -Name "SQL Server Service" `
  -SamAccountName "sqlsvc" `
  -UserPrincipalName "sqlsvc@pmlabs.com" `
  -AccountPassword $SqlSvcPassword `
  -Enabled $true `
  -PasswordNeverExpires $true `
  -CannotChangePassword $true
```

## 9.2 SQL Agent service

```powershell
$SqlAgentPassword = Read-Host "Enter password for SQL Agent service account" -AsSecureString

New-ADUser `
  -Name "SQL Server Agent Service" `
  -SamAccountName "sqlagent" `
  -UserPrincipalName "sqlagent@pmlabs.com" `
  -AccountPassword $SqlAgentPassword `
  -Enabled $true `
  -PasswordNeverExpires $true `
  -CannotChangePassword $true
```

Verify both accounts:

```powershell
"sqlsvc","sqlagent" | ForEach-Object {
    Get-ADUser -Identity $_ -Properties PasswordNeverExpires,CannotChangePassword
} | Select-Object SamAccountName,Enabled,PasswordNeverExpires,CannotChangePassword
```

---

# 10. Configure PMSQL01-PMSQL04 Networking

Use:

```text
PMSQL01: 192.168.0.101
PMSQL02: 192.168.0.102
PMSQL03: 192.168.0.103
PMSQL04: 192.168.0.104

Mask:    255.255.255.0
Gateway: 192.168.0.1
DNS:     192.168.0.100
```

The critical setting is:

```text
Preferred DNS = 192.168.0.100
```

Do not use the VMware NAT DNS or a public resolver as the preferred DNS for domain members.

Verify each node:

```powershell
ipconfig /all
Resolve-DnsName PMDC.pmlabs.com
Test-NetConnection 192.168.0.100 -Port 53
```

---

# 11. Join SQL Nodes to `pmlabs.com`

Run on each SQL node, changing only the machine being operated on:

```powershell
Add-Computer `
-DomainName "pmlabs.com" `
-Credential "PMLABS\Administrator" `
-Restart
```

After reboot:

```powershell
Get-CimInstance Win32_ComputerSystem |
Select-Object Name,Domain,PartOfDomain

nltest /dsgetdc:pmlabs.com
```

Expected:

```text
Domain       : pmlabs.com
PartOfDomain : True
```

---

# 12. Grant Member-Server Administrative Access

On each of PMSQL01-PMSQL04:

```powershell
Add-LocalGroupMember `
  -Group "Administrators" `
  -Member "PMLABS\PMLAB-Server-Admins"
```

Verify:

```powershell
Get-LocalGroupMember Administrators
```

---

# 13. Install Failover Clustering

Run on PMSQL01-PMSQL04:

```powershell
Install-WindowsFeature Failover-Clustering -IncludeManagementTools
```

Verify:

```powershell
Get-WindowsFeature Failover-Clustering
```

---

# 14. Validate and Create WSFC-A

Validate:

```powershell
Test-Cluster -Node PMSQL01,PMSQL02
```

Review the validation report.

Create Cluster A if using the planned lab IP:

```powershell
New-Cluster `
  -Name "WSFC-A" `
  -Node PMSQL01,PMSQL02 `
  -StaticAddress 192.168.0.110
```

Verify:

```powershell
Get-Cluster -Name WSFC-A
Get-ClusterNode -Cluster WSFC-A
Get-ClusterQuorum -Cluster WSFC-A
```

Both nodes should be `Up`.

---

# 15. Validate and Create WSFC-B

Validate:

```powershell
Test-Cluster -Node PMSQL03,PMSQL04
```

Create:

```powershell
New-Cluster `
  -Name "WSFC-B" `
  -Node PMSQL03,PMSQL04 `
  -StaticAddress 192.168.0.120
```

Verify:

```powershell
Get-Cluster -Name WSFC-B
Get-ClusterNode -Cluster WSFC-B
Get-ClusterQuorum -Cluster WSFC-B
```

---

# 16. Configure Quorum

Both clusters have two nodes, so a witness is recommended for realistic failover testing.

Check current quorum:

```powershell
Get-ClusterQuorum -Cluster WSFC-A
Get-ClusterQuorum -Cluster WSFC-B
```

For a File Share Witness, create independent shares and grant the cluster computer object the required permissions. Do not place a cluster's file-share witness on one of that cluster's two member nodes.

---

# 17. Create Shared SQL Backup Folder on PMDC

Example share:

```text
\\PMDC\sharedbkps
```

Grant:

```text
PMLABS\sqlsvc
  Share: Change + Read
  NTFS:  Modify
```

Test from each SQL node:

```powershell
Test-Path "\\PMDC\sharedbkps"
```

Remember: SQL Server performs BACKUP/RESTORE under the SQL Database Engine service identity, not necessarily the interactive administrator account.

---

# 18. Install SQL Server on PMSQL01-PMSQL04

Use the same SQL Server version/build and patch level on all four nodes.

During Setup:

```text
SQL Server Database Engine:
  PMLABS\sqlsvc

SQL Server Agent:
  PMLABS\sqlagent
  Startup Type: Automatic

SQL administrators:
  PMLABS\PMLAB-SQL-Admins
```

Do not use `pmadmin` as the SQL service account.

After installation, validate:

```sql
SELECT
    @@SERVERNAME AS ServerName,
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('Edition') AS Edition;
GO
```

---

# 19. Standardize SQL TCP Connectivity

For a default instance lab, use TCP 1433 consistently.

In SQL Server Configuration Manager:

```text
SQL Server Network Configuration
  Protocols for MSSQLSERVER
    TCP/IP = Enabled

IPAll:
  TCP Dynamic Ports = blank
  TCP Port = 1433
```

Restart SQL Server.

Open the firewall on every SQL node:

```powershell
New-NetFirewallRule `
  -DisplayName "SQL Server TCP 1433" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 1433 `
  -Action Allow
```

Verify from another node:

```powershell
Test-NetConnection PMSQL02.pmlabs.com -Port 1433
```

---

# 20. Enable Always On Availability Groups

Enable Always On on all four SQL instances using SQL Server Configuration Manager, then restart SQL Server.

Verify:

```sql
SELECT
    @@SERVERNAME AS ServerName,
    SERVERPROPERTY('IsHadrEnabled') AS IsHadrEnabled;
GO
```

Expected:

```text
IsHadrEnabled = 1
```

---

# 21. Create / Validate HADR Endpoints

Use TCP 5022 on all four instances.

Check:

```sql
SELECT
    e.name,
    e.state_desc,
    t.port
FROM sys.database_mirroring_endpoints e
JOIN sys.tcp_endpoints t
    ON e.endpoint_id = t.endpoint_id;
GO
```

If no mirroring endpoint exists:

```sql
CREATE ENDPOINT [Hadr_endpoint]
STATE = STARTED
AS TCP
(
    LISTENER_PORT = 5022,
    LISTENER_IP = ALL
)
FOR DATABASE_MIRRORING
(
    ROLE = ALL,
    AUTHENTICATION = WINDOWS NEGOTIATE,
    ENCRYPTION = REQUIRED
);
GO
```

If it exists but is stopped:

```sql
ALTER ENDPOINT [Hadr_endpoint]
STATE = STARTED;
GO
```

Grant the SQL service identity endpoint access where required:

```sql
USE master;
GO

IF NOT EXISTS
(
    SELECT 1
    FROM sys.server_principals
    WHERE name = N'PMLABS\sqlsvc'
)
BEGIN
    CREATE LOGIN [PMLABS\sqlsvc] FROM WINDOWS;
END
GO

GRANT CONNECT ON ENDPOINT::[Hadr_endpoint]
TO [PMLABS\sqlsvc];
GO
```

Firewall:

```powershell
New-NetFirewallRule `
  -DisplayName "SQL AlwaysOn HADR TCP 5022" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 5022 `
  -Action Allow
```

Confirm listening:

```powershell
Get-NetTCPConnection -LocalPort 5022 -State Listen
```

Required connectivity:

```text
PMSQL01 -> PMSQL02 :5022 = True
PMSQL02 -> PMSQL01 :5022 = True

PMSQL03 -> PMSQL04 :5022 = True
PMSQL04 -> PMSQL03 :5022 = True

PMSQL01 -> PMSQL03 :5022 = True
PMSQL03 -> PMSQL01 :5022 = True
```

Example:

```powershell
Test-NetConnection PMSQL03.pmlabs.com -Port 5022
```

---

# 22. Prepare Database `America`

On PMSQL01:

```sql
SELECT
    name,
    state_desc,
    recovery_model_desc
FROM sys.databases
WHERE name = 'America';
GO
```

Required:

```text
America | ONLINE | FULL
```

If needed:

```sql
ALTER DATABASE [America]
SET RECOVERY FULL;
GO
```

---

# 23. Seed PMSQL02 for AGAlpha

On PMSQL01:

```sql
BACKUP DATABASE [America]
TO DISK = '\\PMDC\sharedbkps\America_AGAlpha_FULL.bak'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
GO

BACKUP LOG [America]
TO DISK = '\\PMDC\sharedbkps\America_AGAlpha_LOG.trn'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
GO
```

Verify backups:

```sql
RESTORE VERIFYONLY
FROM DISK = '\\PMDC\sharedbkps\America_AGAlpha_FULL.bak';
GO

RESTORE VERIFYONLY
FROM DISK = '\\PMDC\sharedbkps\America_AGAlpha_LOG.trn';
GO
```

On PMSQL02:

```sql
RESTORE DATABASE [America]
FROM DISK = '\\PMDC\sharedbkps\America_AGAlpha_FULL.bak'
WITH NORECOVERY, REPLACE, STATS = 10;
GO

RESTORE LOG [America]
FROM DISK = '\\PMDC\sharedbkps\America_AGAlpha_LOG.trn'
WITH NORECOVERY, STATS = 10;
GO
```

Verify:

```sql
SELECT name,state_desc,recovery_model_desc
FROM sys.databases
WHERE name='America';
GO
```

Expected:

```text
America | RESTORING | FULL
```

---

# 24. Create AGAlpha

Run on PMSQL01:

```sql
CREATE AVAILABILITY GROUP [AGAlpha]
FOR DATABASE [America]

REPLICA ON

N'PMSQL01' WITH
(
    ENDPOINT_URL = N'TCP://PMSQL01.pmlabs.com:5022',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = AUTOMATIC,
    SEEDING_MODE = MANUAL,
    SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
),

N'PMSQL02' WITH
(
    ENDPOINT_URL = N'TCP://PMSQL02.pmlabs.com:5022',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = AUTOMATIC,
    SEEDING_MODE = MANUAL,
    SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
);
GO
```

On PMSQL02:

```sql
ALTER AVAILABILITY GROUP [AGAlpha] JOIN;
GO

ALTER DATABASE [America]
SET HADR AVAILABILITY GROUP = [AGAlpha];
GO
```

If `ALTER AVAILABILITY GROUP ... JOIN` reports that a replica already exists, do not drop the AG automatically. Check the replica state first; in the lab this occurred because the secondary replica was already registered.

---

# 25. Validate AGAlpha

Run on PMSQL01:

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
JOIN sys.dm_hadr_availability_replica_states ars
    ON ar.replica_id = ars.replica_id
WHERE ag.name = 'AGAlpha';
GO
```

Database synchronization:

```sql
SELECT
    ar.replica_server_name,
    DB_NAME(drs.database_id) AS DatabaseName,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar
    ON drs.replica_id = ar.replica_id
WHERE DB_NAME(drs.database_id) = 'America';
GO
```

Target:

```text
PMSQL01 PRIMARY
PMSQL02 SECONDARY
America SYNCHRONIZED / HEALTHY
```

---

# 26. Create AGAlpha Listener

Actual lab listener:

```text
AGAlpha-LST.pmlabs.com
192.168.0.121
```

Create on current AGAlpha primary:

```sql
ALTER AVAILABILITY GROUP [AGAlpha]
ADD LISTENER N'AGAlpha-LST'
(
    WITH IP
    (
        (N'192.168.0.121', N'255.255.255.0')
    ),
    PORT = 1433
);
GO
```

Verify:

```powershell
Resolve-DnsName AGAlpha-LST.pmlabs.com
Test-NetConnection AGAlpha-LST.pmlabs.com -Port 1433
```

Expected DNS:

```text
AGAlpha-LST.pmlabs.com -> 192.168.0.121
```

---

# 27. Test Local AGAlpha Failover

Only perform planned failover when the secondary is synchronized.

Connect to PMSQL02:

```sql
ALTER AVAILABILITY GROUP [AGAlpha] FAILOVER;
GO
```

Validate roles, then fail back to PMSQL01 after synchronization:

```sql
ALTER AVAILABILITY GROUP [AGAlpha] FAILOVER;
GO
```

Desired starting point before Distributed AG work:

```text
PMSQL01 = AGAlpha PRIMARY
PMSQL02 = AGAlpha SECONDARY
```

---

# 28. Prepare PMSQL03 and PMSQL04 for the Distributed AG

For manual Distributed AG seeding, prepare both copies of `America` from the same backup chain from the AGAlpha global primary.

Example full/log set on the current AGAlpha primary:

```sql
BACKUP DATABASE [America]
TO DISK = '\\PMDC\sharedbkps\America_DAG_FULL.bak'
WITH COPY_ONLY, INIT, COMPRESSION, CHECKSUM, STATS = 10;
GO

BACKUP LOG [America]
TO DISK = '\\PMDC\sharedbkps\America_DAG_LOG.trn'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
GO
```

Restore to PMSQL03:

```sql
RESTORE DATABASE [America]
FROM DISK = '\\PMDC\sharedbkps\America_DAG_FULL.bak'
WITH NORECOVERY, REPLACE, STATS = 10;
GO

RESTORE LOG [America]
FROM DISK = '\\PMDC\sharedbkps\America_DAG_LOG.trn'
WITH NORECOVERY, STATS = 10;
GO
```

Restore the same chain to PMSQL04:

```sql
RESTORE DATABASE [America]
FROM DISK = '\\PMDC\sharedbkps\America_DAG_FULL.bak'
WITH NORECOVERY, REPLACE, STATS = 10;
GO

RESTORE LOG [America]
FROM DISK = '\\PMDC\sharedbkps\America_DAG_LOG.trn'
WITH NORECOVERY, STATS = 10;
GO
```

Both should remain:

```text
America = RESTORING
```

Do not run `WITH RECOVERY`.

---

# 29. Create Empty AGBeta

For the Distributed AG pattern, create the second local AG without initially specifying `America`.

Run on PMSQL03:

```sql
CREATE AVAILABILITY GROUP [AGBeta]
FOR
REPLICA ON

N'PMSQL03' WITH
(
    ENDPOINT_URL = N'TCP://PMSQL03.pmlabs.com:5022',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = MANUAL,
    SEEDING_MODE = MANUAL,
    SECONDARY_ROLE (ALLOW_CONNECTIONS = NO)
),

N'PMSQL04' WITH
(
    ENDPOINT_URL = N'TCP://PMSQL04.pmlabs.com:5022',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = MANUAL,
    SEEDING_MODE = MANUAL,
    SECONDARY_ROLE (ALLOW_CONNECTIONS = NO)
);
GO
```

On PMSQL04:

```sql
ALTER AVAILABILITY GROUP [AGBeta] JOIN;
GO
```

Do not join `America` to AGBeta yet.

---

# 30. Create AGBeta Listener

Actual lab listener:

```text
AGBETA-LST.pmlabs.com
192.168.0.131
```

Run on PMSQL03:

```sql
ALTER AVAILABILITY GROUP [AGBeta]
ADD LISTENER N'AGBETA-LST'
(
    WITH IP
    (
        (N'192.168.0.131', N'255.255.255.0')
    ),
    PORT = 1433
);
GO
```

Verify:

```powershell
Resolve-DnsName AGBETA-LST.pmlabs.com
Test-NetConnection AGBETA-LST.pmlabs.com -Port 1433
```

Expected:

```text
AGBETA-LST.pmlabs.com -> 192.168.0.131
```

---

# 31. Validate Distributed-AG Listener Endpoint Connectivity

This is important:

The AG listeners use TCP 1433 for normal SQL client traffic, but the `LISTENER_URL` inside a Distributed AG uses the **database mirroring endpoint port**, which is `5022` in this lab.

Test:

```powershell
Test-NetConnection AGAlpha-LST.pmlabs.com -Port 5022
Test-NetConnection AGBETA-LST.pmlabs.com -Port 5022
```

Run tests from both AG sides.

All must be:

```text
TcpTestSucceeded : True
```

---

# 32. Take a Final Log Backup Before DAG Join

Because transactions may have occurred after the first DAG backup set, take a final log backup from the current AGAlpha primary:

```sql
BACKUP LOG [America]
TO DISK = '\\PMDC\sharedbkps\America_DAG_FinalLog.trn'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
GO
```

Restore on PMSQL03:

```sql
RESTORE LOG [America]
FROM DISK = '\\PMDC\sharedbkps\America_DAG_FinalLog.trn'
WITH NORECOVERY, STATS = 10;
GO
```

Restore on PMSQL04:

```sql
RESTORE LOG [America]
FROM DISK = '\\PMDC\sharedbkps\America_DAG_FinalLog.trn'
WITH NORECOVERY, STATS = 10;
GO
```

If an LSN / backup-chain error occurs, stop and repair the restore chain before continuing.

---

# 33. Create `AGDistributed`

Run on the current AGAlpha primary:

```sql
CREATE AVAILABILITY GROUP [AGDistributed]
WITH (DISTRIBUTED)

AVAILABILITY GROUP ON

'AGAlpha' WITH
(
    LISTENER_URL = 'TCP://AGAlpha-LST.pmlabs.com:5022',
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
    FAILOVER_MODE = MANUAL,
    SEEDING_MODE = MANUAL
),

'AGBeta' WITH
(
    LISTENER_URL = 'TCP://AGBETA-LST.pmlabs.com:5022',
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
    FAILOVER_MODE = MANUAL,
    SEEDING_MODE = MANUAL
);
GO
```

Verify:

```sql
SELECT name,is_distributed
FROM sys.availability_groups
WHERE name IN ('AGAlpha','AGDistributed');
GO
```

Expected:

```text
AGAlpha        0
AGDistributed  1
```

---

# 34. Join the Distributed AG from PMSQL03

Run on PMSQL03:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
JOIN

AVAILABILITY GROUP ON

'AGAlpha' WITH
(
    LISTENER_URL = 'TCP://AGAlpha-LST.pmlabs.com:5022',
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
    FAILOVER_MODE = MANUAL,
    SEEDING_MODE = MANUAL
),

'AGBeta' WITH
(
    LISTENER_URL = 'TCP://AGBETA-LST.pmlabs.com:5022',
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
    FAILOVER_MODE = MANUAL,
    SEEDING_MODE = MANUAL
);
GO
```

---

# 35. Join `America` on the Forwarder

On PMSQL03:

```sql
ALTER DATABASE [America]
SET HADR AVAILABILITY GROUP = [AGDistributed];
GO
```

Important:

```text
PMSQL03 / forwarder:
America joins AGDistributed
```

---

# 36. Join `America` on PMSQL04

On PMSQL04:

```sql
ALTER DATABASE [America]
SET HADR AVAILABILITY GROUP = [AGBeta];
GO
```

Important distinction:

```text
PMSQL03 -> America -> AGDistributed
PMSQL04 -> America -> AGBeta
```

This is the manual-seeding flow for the second AG of a Distributed AG.

---

# 37. Validate AGBeta

On PMSQL03:

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
WHERE ag.name = 'AGBeta';
GO
```

Expected:

```text
PMSQL03 PRIMARY
PMSQL04 SECONDARY
```

Check `America`:

```sql
SELECT
    ar.replica_server_name,
    DB_NAME(drs.database_id) AS DatabaseName,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar
    ON drs.replica_id = ar.replica_id
WHERE DB_NAME(drs.database_id) = 'America';
GO
```

---

# 38. Validate the Distributed AG

On PMSQL01:

```sql
SELECT
    ag.name AS AGName,
    ag.is_distributed,
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

Expected logical state:

```text
AGAlpha = Global Primary AG
AGBeta  = Forwarder / Secondary AG
```

Final topology:

```text
                           AGDistributed
                                |
                  +-------------+-------------+
                  |                           |
               AGAlpha                     AGBeta
          Global Primary AG           Forwarder AG
                  |                           |
          +-------+-------+           +-------+-------+
          |               |           |               |
       PMSQL01         PMSQL02      PMSQL03         PMSQL04
       PRIMARY         SECONDARY     PRIMARY         SECONDARY
          |               |           |               |
          +-----SYNC------+           +------SYNC-----+
                    \                     /
                     \                   /
                          America
```

Recommended initial Distributed AG mode:

```text
ASYNCHRONOUS_COMMIT
MANUAL FAILOVER
```

Local AGs can remain synchronous.

---

# 39. SQL / HADR Validation Commands

## SQL service accounts

```sql
SELECT
    servicename,
    service_account,
    status_desc
FROM sys.dm_server_services;
GO
```

## HADR enabled

```sql
SELECT SERVERPROPERTY('IsHadrEnabled');
GO
```

## Endpoint

```sql
SELECT
    e.name,
    e.state_desc,
    t.port
FROM sys.database_mirroring_endpoints e
JOIN sys.tcp_endpoints t
    ON e.endpoint_id = t.endpoint_id;
GO
```

## AGs

```sql
SELECT
    name,
    is_distributed
FROM sys.availability_groups;
GO
```

## Replica state

```sql
SELECT
    ag.name,
    ar.replica_server_name,
    ars.role_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar
  ON ag.group_id = ar.group_id
LEFT JOIN sys.dm_hadr_availability_replica_states ars
  ON ar.replica_id = ars.replica_id;
GO
```

## Database state

```sql
SELECT
    DB_NAME(database_id) AS DatabaseName,
    synchronization_state_desc,
    synchronization_health_desc,
    database_state_desc,
    is_suspended
FROM sys.dm_hadr_database_replica_states;
GO
```

---

# 40. Troubleshooting Lessons From This Build

## 40.1 Server Manager Add Roles wizard hangs

Use PowerShell instead:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

## 40.2 PowerShell command pasted twice

If you see an invalid parameter such as:

```text
-IncludeManagementToolsInstall-WindowsFeature
```

the command was pasted twice on the same line. Re-run the command once.

## 40.3 DNS works but RDP name does not

Test IP first:

```powershell
ping 192.168.0.101
```

Then DNS:

```powershell
Resolve-DnsName PMSQL01.pmlabs.com
```

The computer running RDCMan must also be able to resolve `pmlabs.com`.

## 40.4 Ping succeeds but TCP 5022 fails

This means basic networking is fine, but the HADR endpoint/firewall/listener is not.

Check:

```powershell
Get-NetTCPConnection -LocalPort 5022 -State Listen
```

and:

```sql
SELECT
    e.name,
    e.state_desc,
    t.port
FROM sys.database_mirroring_endpoints e
JOIN sys.tcp_endpoints t
  ON e.endpoint_id = t.endpoint_id;
GO
```

Then confirm firewall rule.

## 40.5 SQL connection fails but 5022 works

TCP 5022 is HADR replication traffic.

Normal SQL client connectivity is typically:

```text
TCP 1433
```

Test separately:

```powershell
Test-NetConnection PMSQL02.pmlabs.com -Port 1433
```

## 40.6 Error 35250

Typical checks:

- HADR endpoint is `STARTED`
- TCP 5022 works in both directions
- SQL service identities are correct
- Endpoint CONNECT permission exists
- Database was restored `WITH NORECOVERY`
- Replica is connected before joining the database

Check SQL error log:

```sql
EXEC sys.xp_readerrorlog 0,1,N'35250';
GO

EXEC sys.xp_readerrorlog 0,1,N'endpoint';
GO
```

## 40.7 Error 41106: replica already exists

Do not immediately drop the availability group.

Check:

```sql
SELECT
    ag.name,
    ar.replica_server_name
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar
    ON ag.group_id = ar.group_id
WHERE ag.name = 'AGAlpha';
GO
```

If the replica already exists, fix connectivity and join only the database if required.

---

# 41. Planned Failover Testing

## 41.1 Local AGAlpha

Before failover, ensure PMSQL02 is `SYNCHRONIZED`.

On the current synchronized secondary:

```sql
ALTER AVAILABILITY GROUP [AGAlpha] FAILOVER;
GO
```

Verify listener still connects to the active primary.

## 41.2 Local AGBeta

After the Distributed AG is healthy and the local AGBeta replicas are synchronized, perform a planned local AGBeta failover following the same principle.

## 41.3 Distributed AG

Distributed AG failover is not the same as local AG failover. Use the current Microsoft procedure for your SQL Server major version and build.

For SQL Server 2022 and later, Microsoft documents additional steps to synchronize the distributed AG before invoking the only supported distributed-AG failover command:

```sql
ALTER AVAILABILITY GROUP [AGDistributed]
FORCE_FAILOVER_ALLOW_DATA_LOSS;
GO
```

Despite the command name, Microsoft provides a controlled procedure to synchronize first when zero data loss is required. Do not use this casually. Validate synchronization and follow the official failover sequence for the exact SQL Server version before testing.

---

# 42. Final Health Checklist

## AD / DNS

- [ ] PMDC = `192.168.0.100`
- [ ] `pmlabs.com` healthy
- [ ] DNS service running
- [ ] All nodes resolve PMDC
- [ ] All nodes use `192.168.0.100` as DNS
- [ ] `dcdiag` reviewed

## Domain Accounts

- [ ] `PMLABS\pmadmin`
- [ ] `PMLABS\sqlsvc`
- [ ] `PMLABS\sqlagent`
- [ ] `PMLAB-Server-Admins`
- [ ] `PMLAB-SQL-Admins`

## Windows Clusters

- [ ] WSFC-A: PMSQL01 + PMSQL02
- [ ] WSFC-B: PMSQL03 + PMSQL04
- [ ] Cluster validation reports reviewed
- [ ] Both clusters healthy
- [ ] Quorum/witness reviewed

## SQL

- [ ] Same SQL major version/build where intended
- [ ] SQL Engine uses `PMLABS\sqlsvc`
- [ ] SQL Agent uses `PMLABS\sqlagent`
- [ ] `PMLAB-SQL-Admins` has SQL sysadmin
- [ ] TCP 1433 accessible
- [ ] Always On enabled
- [ ] TCP 5022 accessible in all required directions
- [ ] HADR endpoints STARTED

## AGAlpha

- [ ] PMSQL01 primary
- [ ] PMSQL02 secondary
- [ ] America healthy/synchronized
- [ ] Listener `AGAlpha-LST.pmlabs.com`
- [ ] Listener IP `192.168.0.121`

## AGBeta

- [ ] PMSQL03 primary/forwarder
- [ ] PMSQL04 secondary
- [ ] America healthy/synchronized
- [ ] Listener `AGBETA-LST.pmlabs.com`
- [ ] Listener IP `192.168.0.131`

## Distributed AG

- [ ] `AGDistributed` exists
- [ ] AGAlpha is global primary AG
- [ ] AGBeta is forwarder AG
- [ ] Distributed connection healthy
- [ ] America is flowing to AGBeta
- [ ] Manual failover runbook prepared before testing

---

# 43. Reference Architecture Summary

```text
Local PC / RDCMan
192.168.0.73
       |
       | 192.168.0.0/24
       |
PMDC 192.168.0.100
AD DS / DNS / \\PMDC\sharedbkps
       |
       +----------------------------------------------------+
       |                                                    |
       |                                                    |
WSFC-A                                                   WSFC-B
       |                                                    |
PMSQL01 .101 ---------------------------------------- PMSQL03 .103
PMSQL02 .102 ---------------------------------------- PMSQL04 .104
       |                                                    |
    AGAlpha                                               AGBeta
       |                                                    |
AGAlpha-LST                                           AGBETA-LST
192.168.0.121                                        192.168.0.131
       |                                                    |
       +---------------- AGDistributed ---------------------+
                              |
                           America
```

---

# 44. Official Microsoft References

Use Microsoft Learn as the source of truth for version-specific behavior:

- Windows Server Failover Cluster creation:
  https://learn.microsoft.com/windows-server/failover-clustering/create-failover-cluster

- Failover Cluster PowerShell module:
  https://learn.microsoft.com/powershell/module/failoverclusters/?view=windowsserver2025-ps

- SQL Server Always On prerequisites:
  https://learn.microsoft.com/sql/database-engine/availability-groups/windows/prereqs-restrictions-recommendations-always-on-availability

- CREATE AVAILABILITY GROUP:
  https://learn.microsoft.com/sql/t-sql/statements/create-availability-group-transact-sql

- Configure a Distributed Availability Group:
  https://learn.microsoft.com/sql/database-engine/availability-groups/windows/configure-distributed-availability-groups

- Distributed Availability Groups overview:
  https://learn.microsoft.com/sql/database-engine/availability-groups/windows/distributed-availability-groups

- Always On configuration troubleshooting:
  https://learn.microsoft.com/sql/database-engine/availability-groups/windows/troubleshoot-always-on-availability-groups-configuration-sql-server

---

# 45. Important Lab Notes

1. This is a VMware practice environment, not a production design.
2. Reserve static IPs in your physical/router DHCP configuration so they are not handed to other devices.
3. Do not treat `PMLABS\sqlsvc` or `PMLABS\sqlagent` as administrator accounts.
4. Keep backups available until all AG/DAG synchronization and failover tests are complete.
5. Never recover a manually seeded secondary (`RESTORE ... WITH RECOVERY`) before the intended AG/DAG join.
6. Verify `TcpTestSucceeded : True` for all required SQL/HADR paths before troubleshooting SQL metadata.
7. For Distributed AG failover, follow the procedure for your exact SQL Server major version/build rather than using a generic failover command without synchronization checks.


# Workshop 2: Azure Files

This workshop uses an isolated Azure environment. A Windows Server VM simulates a source file server, a second Windows Server VM acts as the client, and an SMB Azure file share is exposed only through a private endpoint. No customer-network connectivity is required.

Procedures and portal labels were checked against Microsoft Learn on **26 August 2026**. Azure portal experiences can change; the facilitator should recheck the linked Learn articles shortly before delivery.

## Outcomes

By the end of the workshop, participants can:

- Explain the protocol, performance, redundancy, and identity choices for Azure Files.
- Validate an SMB share that is reachable only through a private endpoint.
- Migrate a measured data set with Robocopy and verify the result.
- Recover data from a share snapshot and an Azure Backup recovery point.
- Interpret Azure Files metrics, logs, alerts, capacity, and throttling signals.
- Diagnose DNS, TCP 445, credentials, network access, permissions, and service health.
- Decide when Azure File Sync is appropriate.

> This disposable lab mounts SMB with a storage account key. Microsoft recommends identity-based authentication for production. Microsoft Entra Kerberos is a design discussion here because it requires tenant-level configuration and supported client identities.

## Schedule

| Time | Activity |
|---|---|
| 09:00-10:00 | Foundations and design decisions |
| 10:00-11:00 | Lab 1: Validate a private SMB share |
| 11:00-12:00 | Lab 2: Migrate and validate data |
| 12:00-13:00 | Lunch |
| 13:00-13:20 | Azure File Sync decision exercise |
| 13:20-14:20 | Lab 3: Protect and recover files |
| 14:20-15:20 | Lab 4: Harden, monitor, and troubleshoot |
| 15:20-16:20 | Lab 5: Incident scenarios |
| 16:20-16:45 | Review and cleanup |

## Architecture

![Workshop 2 Azure Files architecture showing private SMB access, migration, backup, and monitoring](assets/workshop-2-azure-files-architecture.png)

## Assigned Environment

Replace `NN` with your two-digit participant number. Do not use another participant's subscription.

| Item | Assigned value |
|---|---|
| Participant | `pNN` |
| Subscription | Provided by facilitator |
| Region | `Sweden Central` |
| Resource group | `rg-lab-files-pNN` |
| Virtual network | `vnet-lab-files-pNN` |
| Subnet | `snet-workload` |
| Client VM | `vm-filesclient-pNN` |
| Source VM | `vm-fileserver-pNN` |
| Storage account | Provided by facilitator; begins `stfilespNN` |
| File share | `workshop` |
| Private endpoint | `pe-files-pNN` |
| Private DNS zone | `privatelink.file.core.windows.net` |
| Recovery Services vault | `rsv-lab-files-pNN` |
| Log Analytics workspace | `log-lab-files-pNN` |
| Action group | `ag-lab-files-pNN` |

## Sign In and Connect to the VMs

1. Open [Azure portal](https://portal.azure.com) in an InPrivate or Incognito window.
2. Sign in with the Workshop 2 account supplied by the facilitator and change the temporary password when prompted.
3. In the search box at the top of the portal, enter `Subscriptions`, and then select **Subscriptions** under **Services**.
4. In the subscription list, select the subscription provided by the facilitator. On its **Overview** page, confirm that the **Subscription ID** matches your assigned subscription.
5. In the portal search box, enter `Resource groups`, and then select **Resource groups** under **Services**.
6. If resource groups from several subscriptions are shown, select **Add filter**, choose **Subscription**, select only your assigned subscription, and then select **Apply**.
7. Select `rg-lab-files-pNN`. On the resource-group **Overview** page, locate these two virtual machines:
    - `vm-filesclient-pNN` - the client VM used first for SMB connectivity and recovery exercises.
    - `vm-fileserver-pNN` - the source VM used later for the Robocopy migration exercise.
8. Select `vm-filesclient-pNN`. On the VM **Overview** page, confirm that **Status** is **Running**. If it is stopped, select **Start** and wait until the status changes to **Running**.
9. Select **Connect > Bastion**.
10. For **Authentication Type**, select **VM Password**, enter the VM administrator credentials supplied by the facilitator, and select **Connect**.
11. Bastion Developer opens the Windows desktop in the browser and permits one VM connection at a time. When a later task requires `vm-fileserver-pNN`, disconnect from the client VM, return to `rg-lab-files-pNN`, open `vm-fileserver-pNN`, confirm that it is running, and repeat steps 9-10.

## Lab 1: Validate a Private SMB Share

**Time:** 60 minutes  
**Purpose:** Verify storage configuration, private endpoint routing, DNS, and SMB access before moving data.

### Task 1: Inspect the storage and share configuration

1. In the portal, search for **Storage accounts** and open your assigned account.
2. On **Overview**, record **Performance**, **Redundancy**, **Account kind**, and **Location**.
3. Under **Settings**, select **Configuration** and verify:
   - **Secure transfer required** is enabled.
   - **Minimum TLS version** is **Version 1.2** or later.
   - **Allow storage account key access** is enabled for this disposable key-based lab.
   - **Hierarchical namespace** is disabled.
4. Under **Data storage**, select **File shares**, then select `workshop`.
5. On the share **Overview**, record the protocol, access tier, and quota.
6. Return to the file-share list, select **Soft delete**, and confirm that soft delete is enabled with seven-day retention.

**Evidence:** Record the storage settings and explain why account-key access is an exception for this lab rather than the production recommendation.

### Task 2: Inspect the private endpoint and DNS path

1. In the storage account, select **Security + networking > Networking**.
2. Under **Public network access**, verify the account is disabled for public access.
3. Select **Private endpoint connections** and open `pe-files-pNN`.
4. Verify the connection state is **Approved** and the target subresource is `file`.
5. From the private endpoint, open **DNS configuration** and record its private IP address.
6. Return to the resource group and open the private DNS zone `privatelink.file.core.windows.net`.
7. Select **Recordsets** and confirm an A record maps the storage account name to the private endpoint IP.
8. Select **Virtual network links** and confirm only `vnet-lab-files-pNN` is linked.

### Task 3: Validate DNS and TCP 445

Connect to `vm-filesclient-pNN` through Bastion Developer. Open Windows PowerShell and run:

```powershell
$storageAccount = '<assigned-storage-account>'
$hostName = "$storageAccount.file.core.windows.net"
Resolve-DnsName $hostName
Test-NetConnection $hostName -Port 445
```

Confirm that the standard hostname resolves through a CNAME to `privatelink.file.core.windows.net`, the returned A record is the private endpoint address, and `TcpTestSucceeded` is `True`.

> Always mount the standard `file.core.windows.net` hostname. Do not mount the `privatelink` hostname directly.

### Task 4: Mount the share

1. In the portal, open the storage account and select **Security + networking > Access keys**.
2. Select **Show keys**, copy one key, and keep it out of notes and screenshots.
3. On the client VM, run:

```powershell
$storageAccount = '<assigned-storage-account>'
$shareName = 'workshop'
$key = Read-Host 'Paste storage account key' -AsSecureString
$credential = [pscredential]::new("Azure\$storageAccount", $key)
New-PSDrive -Name Z -PSProvider FileSystem `
    -Root "\\$storageAccount.file.core.windows.net\$shareName" `
    -Credential $credential -Persist
New-Item Z:\Department, Z:\Migration, Z:\Recovery -ItemType Directory -Force
Get-ChildItem Z:\
```

4. Repeat the mount on `vm-fileserver-pNN` after disconnecting the client Bastion session.

**Pass criteria:** Both VMs resolve the private endpoint, TCP 445 succeeds, and each can create and read a test file through `Z:`.

## Lab 2: Migrate and Validate Data

**Time:** 60 minutes  
**Purpose:** Perform a measured and verifiable migration from the simulated source server.

### Task 1: Inventory the source

Connect to `vm-fileserver-pNN`. The pre-stage script created nested paths, an empty folder, long names, text files, and a 5 MiB binary file under `C:\LabSource`.

```powershell
$source = 'C:\LabSource'
$sourceFiles = Get-ChildItem $source -File -Recurse
$sourceInventory = [pscustomobject]@{
    FileCount   = $sourceFiles.Count
    FolderCount = (Get-ChildItem $source -Directory -Recurse).Count
    TotalBytes  = ($sourceFiles | Measure-Object Length -Sum).Sum
    LargestFile = ($sourceFiles | Sort-Object Length -Descending | Select-Object -First 1).FullName
}
$sourceInventory | Format-List
```

Record the output. Identify open files or business processes that would require a cutover window in a real migration.

### Task 2: Run the initial copy

```powershell
$log = 'C:\LabLogs\robocopy-initial.log'
New-Item -ItemType Directory -Path (Split-Path $log) -Force | Out-Null
$stopwatch = [Diagnostics.Stopwatch]::StartNew()
robocopy C:\LabSource Z:\Migration /E /COPY:DAT /DCOPY:DAT /R:2 /W:2 /MT:8 /TEE /LOG:$log
$exitCode = $LASTEXITCODE
$stopwatch.Stop()
[pscustomobject]@{
    RobocopyExitCode = $exitCode
    ElapsedSeconds   = [math]::Round($stopwatch.Elapsed.TotalSeconds, 2)
}
```

Robocopy exit codes `0` through `7` are success or informational results. An exit code of `8` or higher indicates at least one failure.

### Task 3: Validate counts, bytes, and hashes

```powershell
$destination = 'Z:\Migration'
$destinationFiles = Get-ChildItem $destination -File -Recurse
$destinationInventory = [pscustomobject]@{
    FileCount   = $destinationFiles.Count
    FolderCount = (Get-ChildItem $destination -Directory -Recurse).Count
    TotalBytes  = ($destinationFiles | Measure-Object Length -Sum).Sum
}
$sourceInventory
$destinationInventory

$sourceHashes = Get-ChildItem C:\LabSource -File -Recurse | ForEach-Object {
    [pscustomobject]@{
        RelativePath = $_.FullName.Substring('C:\LabSource'.Length)
        Hash = (Get-FileHash $_.FullName -Algorithm SHA256).Hash
    }
}
$destinationHashes = Get-ChildItem Z:\Migration -File -Recurse | ForEach-Object {
    [pscustomobject]@{
        RelativePath = $_.FullName.Substring('Z:\Migration'.Length)
        Hash = (Get-FileHash $_.FullName -Algorithm SHA256).Hash
    }
}
Compare-Object $sourceHashes $destinationHashes -Property RelativePath, Hash
```

No `Compare-Object` output means the file paths and hashes match.

### Task 4: Run a delta copy

```powershell
Add-Content C:\LabSource\Finance\sample-003.txt 'Delta change'
Set-Content C:\LabSource\Finance\new-after-initial.txt 'New file'
Remove-Item C:\LabSource\HR\Policies\sample-002.txt -ErrorAction SilentlyContinue
robocopy C:\LabSource Z:\Migration /E /COPY:DAT /DCOPY:DAT /R:2 /W:2 /MT:8 /TEE /LOG:C:\LabLogs\robocopy-delta.log
"Robocopy exit code: $LASTEXITCODE"
```

Review the copied, skipped, mismatch, failed, and extra counts. The deleted source file remains at the destination because `/MIR` was deliberately not used.

**Pass criteria:** Initial counts, bytes, and hashes match; the delta run copies changed files; and the participant can explain how downtime, final delta, validation, and rollback fit into cutover.

## Azure File Sync Decision Exercise

**Time:** 20 minutes, facilitator-led. No deployment.

- Use Azure File Sync when Windows Servers remain and local caching, branch access, or low-downtime migration justifies the agent and sync lifecycle.
- Cloud tiering manages local capacity; it is not backup.
- Direct Azure Files access is simpler when file servers will be retired.
- Operating File Sync adds agent updates, server registration, sync health, files-not-syncing investigation, tiering-policy tuning, and recall planning.

Decision question: after the migration, do Windows file servers remain in the estate? If not, use a measured copy method such as Robocopy or AzCopy and avoid a permanent sync dependency.

## Lab 3: Protect and Recover Files

**Time:** 60 minutes  
**Purpose:** Compare self-managed snapshots with policy-managed snapshot-tier Azure Backup.

### Task 1: Create and use a share snapshot

1. On the client VM, create the recovery data:

```powershell
New-Item Z:\Recovery -ItemType Directory -Force | Out-Null
Set-Content Z:\Recovery\quarterly-plan.txt 'Quarterly plan version 1'
Set-Content Z:\Recovery\delete-me.txt 'Snapshot recovery sample'
```

2. In the portal, open the storage account, select **Data storage > File shares**, and open `workshop`.
3. Select **Operations > Snapshots**, select **+ Add snapshot**, add the comment `Before Lab 3 changes`, and select **OK**.
4. On the VM, change `quarterly-plan.txt` to version 2 and delete `delete-me.txt`.
5. Return to **Operations > Snapshots** and open the snapshot.
6. Browse to `Recovery`, select `quarterly-plan.txt`, and download the snapshot version. Save it under an alternate name so the live version remains unchanged.
7. The portal's **Restore > Overwrite original file** action replaces the live file. Use that option only when replacement is intended and approved.
8. In File Explorer, right-click `Z:\Recovery`, select **Properties > Previous Versions**, open the snapshot, and copy `delete-me.txt` back as `delete-me-restored.txt`.

### Task 2: Inspect Azure Backup protection

1. In the portal, search for and open `rsv-lab-files-pNN`.
2. Select **Backup**. The current Learn flow labels the datasource **Azure Files (Azure Storage)**.
3. Review the existing protected item and policy `pol-files-snapshot-lab`.
4. Confirm that the policy uses the **Snapshot** backup tier and seven-day retention. Snapshot tier is required here because the lab performs item-level recovery.
5. Select **Resiliency > Protected inventory > Protected items** and filter the datasource type to **Azure Storage (Azure Files)**.
6. Open the `workshop` protected item, select **Backup now**, set retention to seven days, and start the job.
7. Select **Resiliency > Monitoring + Reporting > Jobs** and wait for the backup job to complete.

### Task 3: Restore an individual file to an alternate location

1. Open **Resiliency > Protected inventory > Protected items** with datasource type **Azure Storage (Azure Files)**.
2. Select the `workshop` item and then select **File Recovery**.
3. Under **Restore Point**, select **Select**, choose a snapshot-tier recovery point, and select **OK**.
4. Under **Recovery Destination**, select **Alternate Location**.
5. Select the assigned storage account and `workshop` share; enter `BackupRestore` for **Folder Name**.
6. Select **Add File**, browse to `Recovery`, select the item to recover, and select **Select**.
7. For conflicts, select **Skip** unless the facilitator directs otherwise, then select **Restore**.
8. Monitor the restore under **Resiliency > Monitoring + Reporting > Jobs**.
9. On the client VM, compare the restored content and timestamps under `Z:\BackupRestore`.

**Pass criteria:** A deleted or changed item is recovered without unintentionally overwriting current data, and the participant identifies the recovery point and destination used.

## Lab 4: Harden, Monitor, and Troubleshoot

**Time:** 60 minutes  
**Purpose:** Review security controls and investigate the client, network, service, and capacity layers.

### Task 1: Review hardening controls

1. Open the storage account and select **Security + networking > Networking**. Confirm public network access is disabled and the private endpoint is approved.
2. Select **Settings > Configuration**. Confirm secure transfer and TLS 1.2 or later.
3. Select **Data storage > File shares > Soft delete** and record the retention period.
4. Select **Access control (IAM) > Role assignments** and identify who can read or manage the account.
5. Select **Security + networking > Access keys** and review the **Rotate and synchronize keys** guidance. Do not rotate a key until the facilitator authorizes the shared exercise.
6. Explain what must change in production: use identity-based authentication, apply least privilege, protect key operations, and inventory services that still require keys.

### Task 2: Analyze metrics

1. In the storage account, select **Monitoring > Metrics**.
2. Select **+ Add metric** and set **Metric Namespace** to **File**.
3. Review **Transactions**, **Availability**, **Success E2E Latency**, **Success Server Latency**, **Ingress**, **Egress**, and **File Capacity**.
4. For **Transactions**, select **Apply splitting** and split by **Response type** and **API name**. Look for throttling response types.
5. Compare Success E2E Latency with Success Server Latency. A large gap indicates delay outside the Azure Files service, such as the client or network path.
6. For this pay-as-you-go share, per-share dimensions can be unavailable. Record whether the chart is at account or share scope rather than assuming a **File Share** dimension exists.

### Task 3: Review diagnostics and alerts

1. In the storage account, select **Monitoring > Diagnostic settings**.
2. Select the **file** resource and confirm `diag-files-pNN` sends all logs and metrics to `log-lab-files-pNN`.
3. Select **Monitoring > Alerts > Alert rules** and open `alert-files-availability-pNN`.
4. Confirm the scope is the storage account's `file` service, the signal is **Availability**, aggregation is **Average**, operator is **Less than**, threshold is `99.9`, evaluation frequency is five minutes, and lookback is one hour.
5. Confirm the action group is `ag-lab-files-pNN`. It may intentionally have no notification receiver unless the facilitator supplied an email address.
6. Generate activity on the client VM:

```powershell
1..20 | ForEach-Object {
    Set-Content "Z:\Department\monitor-$_.txt" "monitor sample $_"
    Get-Content "Z:\Department\monitor-$_.txt" | Out-Null
}
```

7. Open `log-lab-files-pNN`, select **Logs**, set the time range to the last hour, and run:

```kusto
StorageFileLogs
| where TimeGenerated > ago(1h)
| summarize Operations=count() by Protocol, OperationName, StatusText
| order by Operations desc
```

Log ingestion is not immediate. If no rows appear, wait several minutes and rerun the query.

### Task 4: Use the troubleshooting ladder

For the facilitator-supplied symptom, test in this order:

1. **Name resolution:** Does the standard storage hostname resolve to the intended private IP?
2. **Transport:** Does `Test-NetConnection <account>.file.core.windows.net -Port 445` succeed?
3. **Credentials:** Is the drive using a current account key, and are stale Windows credentials cached?
4. **Network access:** Is the private endpoint approved, DNS zone linked, and public endpoint correctly restricted?
5. **Permissions:** Can the current SMB identity perform the requested file operation?
6. **Service state:** Are quota, throttling, availability, or latency metrics abnormal?

**Pass criteria:** Identify the failing layer, show the evidence, and propose a verification action before changing configuration.

## Lab 5: Incident Scenarios

**Time:** 60 minutes  
**Purpose:** Produce an evidence-based response to an ambiguous incident.

Use this incident-note format:

| Field | Required content |
|---|---|
| Impact | Users, paths, operations, and business effect |
| Time window | First observed time and last known-good time |
| Evidence | Commands, metrics, logs, jobs, or recovery points |
| Diagnosis | Failing layer and confidence level |
| Action | Containment and recovery decision |
| Validation | How correct service and data are proven |
| Prevention | Control, alert, process, or architecture change |

### Scenario A: Accidental deletion

- Determine deletion time and affected path.
- Choose Previous Versions, snapshot item restore, or full-share restore.
- Restore to an alternate path first and validate with the data owner.

### Scenario B: Suspected ransomware

- Stop further writes without destroying evidence.
- Identify the last known-good recovery point.
- Compare snapshot-tier recovery with vaulted backup isolation and immutability options.
- Recover to an isolated location and scan before cutover.

### Scenario C: Capacity pressure

- Confirm quota, used capacity, growth rate, and whether metrics are account or share scoped.
- Distinguish a share quota from storage-account or provisioned-capacity limits.
- Recommend an immediate action and an alert threshold.

### Scenario D: High latency

- Compare client-observed time, Success E2E Latency, and Success Server Latency.
- Check throttling, DNS, private endpoint routing, SMB transport, and workload IOPS profile.
- Decide whether the evidence justifies a tier, provisioned performance, local cache, or workload change.

Each participant presents one incident note and answers: what evidence would disprove your diagnosis?

## Cleanup

Do not delete the environment until the facilitator has captured validation evidence.

1. On each VM, run `Remove-PSDrive Z -Force` and remove cached credentials if any were added with `cmdkey`.
2. Confirm restore and migration evidence has been saved outside the disposable lab.
3. Stop here and notify the facilitator that your environment is ready for cleanup. Participants must not delete resource groups or backup resources directly.
4. The facilitator uses the workshop cleanup automation supplied separately by the workshop owner. Before execution, verify that its preview contains only the exact resource group `rg-lab-files-pNN` for this participant.
5. The facilitator confirms that the protected Azure Files item and registered storage container are removed before the Recovery Services vault, and then deletes only `rg-lab-files-pNN`.
6. Confirm that `rg-lab-files-pNN` is absent and that no other participant resource group was changed.
7. The facilitator deletes any locally stored participant credential file after all required first sign-ins are complete.

## Microsoft Learn References

- [Configure Azure Files network endpoints](https://learn.microsoft.com/azure/storage/files/storage-files-networking-endpoints)
- [Mount an SMB Azure file share on Windows](https://learn.microsoft.com/azure/storage/files/storage-how-to-use-files-windows)
- [Use Azure Files share snapshots](https://learn.microsoft.com/azure/storage/files/storage-snapshots-files)
- [Back up Azure Files in the Azure portal](https://learn.microsoft.com/azure/backup/backup-azure-files)
- [Restore Azure Files with Azure Backup](https://learn.microsoft.com/azure/backup/restore-afs)
- [Monitor Azure Files using Azure Monitor](https://learn.microsoft.com/azure/storage/files/storage-files-monitoring)
- [Create Azure Files monitoring alerts](https://learn.microsoft.com/azure/storage/files/files-monitoring-alerts)
- [Bastion Developer quickstart](https://learn.microsoft.com/azure/bastion/quickstart-developer-sku)
- [Plan an Azure File Sync deployment](https://learn.microsoft.com/azure/storage/file-sync/file-sync-planning)

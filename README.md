# Enable Cross-Region Backup Replication for EC2 using AWS Backup

## **1. EC2 Instance Setup (Primary/Source Region)**

1. **Launch an EC2 Instance**
    - **Instance type**: `t2.micro` (for testing purposes)
    - **Storage**: Use the default EBS root volume, or attach additional EBS volumes if you'd like to test backing up multiple volumes.
    - **Security Group**: Allow inbound SSH (port 22) to access the instance.
    - **Key Pair**: Select or create a key pair for SSH access.
2. **Prepare the Instance**
    - SSH into the instance using your key pair.
    - Create or upload test data (e.g., files, logs, dummy databases) to the instance's EBS volumes to simulate production data.
3. **Tag the EC2 Instance for Backup Plan Assignment**
    - In the EC2 console, go to the **Tags** section of the instance.
    - Add the following tag:
        - **Key**: `Backup`
        - **Value**: `cross-region-test`

> This tag will be used to automatically associate the instance with the backup plan in AWS Backup.
> 

---

## **2. Create Backup Vaults**

To enable cross-region replication, AWS Backup requires a **source vault** (in the primary region) and a **destination vault** (in the disaster recovery region).

### **A — Create Source Vault (Primary Region)**

1. Navigate to **AWS Backup → Backup vaults → Create backup vault**.
2. Configure:
    - **Name**: `ec2-primary-vault-1`
    - **Encryption**: Use the AWS managed key (default) or a customer-managed CMK, depending on your organization's security policies.
3. Click **Create**.

### **B — Create Destination Vault (Disaster Recovery Region)**

1. Switch to your **destination region** in the AWS Console.
2. Navigate to **AWS Backup → Backup vaults → Create backup vault**.
3. Configure:
    - **Name**: `ec2-dr-vault-1`
    - **Encryption**: Same as source vault – AWS managed or CMK.
4. Click **Create**.

> Using separate vaults ensures logical separation of backups across regions and improves recovery readiness.
> 

---

## **3. Create a Backup Plan**

1. Navigate to **AWS Backup → Backup plans → Create backup plan**.
2. Select **Build a new plan**.

### **Backup Plan Details**

- **Name**: `ec2-cross-region-plan`

### **Backup Rule Configuration**

- **Rule name**: `WeeklyEC2Backup-rule`
- **Backup vault**: `ec2-primary-vault-1` (source vault)
- **Backup frequency**: Weekly
- **Backup window**: Accept default or customize per organizational backup window
- **Lifecycle configuration**:
    - **Transition to cold storage after**: 14 days
    - **Expire after**: 35 days

> Transitioning backups to cold storage after 14 days helps reduce storage costs by up to 75%, as cold storage is significantly cheaper (~$0.0125 per GB-month vs. ~$0.05 per GB-month for warm storage).
> 
> 
> Expiring backups after 35 days ensures unnecessary data is removed, reducing storage sprawl and ongoing costs. These settings should be adjusted based on compliance and retention policies.
> 

---

### **Enable Cross-Region Copy**

- **Copy to destination**: Enable
- **Destination region**: Select the target region where the disaster recovery vault is located
- **Destination vault**: `ec2-dr-vault-1`
- **Copy lifecycle**:
    - Transition to cold storage: 12 days
    - Expire after: 35 days (same as source rule for consistency)

> Enabling cross-region copy ensures business continuity in the event of a regional outage. Backups stored in the DR region can be restored independently of the primary region.
> 

---

## **4. Assign Resources to the Backup Plan**

1. After the backup plan is created, click **Assign resources**.
2. Select **Assign by tag**.

### **Assignment Details**

- **Resource assignment name**: `ec2-cross-region`
- **Tag key**: `Backup`
- **Tag value**: `cross-region-test`
- **IAM Role**: Use the default role.

---

## **Summary Report**

- Why Enable Cross-Region Backup Replication?

| Benefit | Description |
| --- | --- |
| **Disaster Recovery** | Ensures EC2 data is recoverable in a separate AWS region during outages. |
| **Cost Optimization** | Lifecycle rules transition backups to cold storage, reducing costs by ~75%. |
| **Compliance & Retention** | Helps meet compliance by storing backups in geographically diverse regions. |
| **Automation & Scalability** | Automatically applies to resources with matching tags, scaling with demand. |

## Issue I Encountered & How I Resolved It

| **Issue** | **Cause** | **Resolution** |
| --- | --- | --- |
| **Backups not triggering as expected** | Missing tags on the EC2 instance | Verified the EC2 instance had the exact tag key `Backup` and value `cross-region-test`. Tagging must match the assignment criteria in the backup plan exactly. |
| **Manual or on-demand backup didn’t start immediately** | Misunderstanding of the Backup Start Window setting in the backup plan | The Backup Start Window defines how long AWS Backup can wait before starting a backup. For example, if the backup window begins at 2:00 AM and the start window is 1 hour, AWS may initiate the backup anytime between 2:00 AM and 3:00 AM. This is expected behavior, not a failure. If you want an immediate backup, use the "Create on-demand backup" option outside the scheduled backup plan. Also, monitor the job under Backup jobs to see the actual start time. |

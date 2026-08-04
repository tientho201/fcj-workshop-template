---
title: "Configure AWS Credentials"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.2.1 </b> "
---


### Preparation steps

**Step 1.1 - Sign in to the Console and select a Region**

Sign in to the [AWS Management Console](https://console.aws.amazon.com/) and select the **us-east-1 (N. Virginia)** Region in the top-right corner.

![Select Region in the top-right corner of the AWS Console](/images/5-Workshop/5.2-Prerequisite/image5.2.1-1.png)

**Step 1.2 - Enable MFA for root**

Go to **IAM → Dashboard → Security recommendations → Add MFA**, and use an authenticator app (Google Authenticator/Authy) to scan the QR code.

![Configure MFA for the root user](/images/5-Workshop/5.2-Prerequisite/image5.2.1-2.png)


**Step 1.3 - Add Policies using a JSON file**

Copy the information from **IAM permissions**. Then go to **IAM → Dashboard**. Next, go to **Left menu → Policies → click the Create policy button (top-right corner)**. You'll see 2 tabs: **Visual editor** and **JSON** — click the **JSON** tab. Delete all the sample code already in that box, then paste **(Ctrl+V)** the entire content you copied in step 1. Click **Next**. On the next screen, there's a Policy name field — this is the only field you need to fill in yourself, for example type: ``RAGProjectDeployPolicy``.
Click **Create policy**.

![Create policies](/images/5-Workshop/5.2-Prerequisite/image5.2.1-3a.png)
![Create policies](/images/5-Workshop/5.2-Prerequisite/image5.2.1-3b.png)
![Create policies](/images/5-Workshop/5.2-Prerequisite/image5.2.1-3c.png)

→ Once this step is done, the policy has been created with all 8 permission groups included (you don't need to add anything else).

**Step 1.4 - Attach the policy to a Group**

Go to **IAM → IAM User groups → Create group** (for example `rag-developers`). **Permissions tab → Add permissions → Attach policies**. Search for the name you set in step 7 **(RAGProjectDeployPolicy) → tick to select → Add permissions**.

![Create an IAM Group and attach the policy](/images/5-Workshop/5.2-Prerequisite/image5.2.1-4.png)

**Step 1.5 - Create an IAM User and add it to the group**

Go to **IAM → IAM Users → Create user**, add it to the group, enable Console access + require MFA.

![Create an IAM User and add it to the group](/images/5-Workshop/5.2-Prerequisite/image5.2.1-5a.png)

![Create an IAM User and add it to the group](/images/5-Workshop/5.2-Prerequisite/image5.2.1-5b.png)

![Create an IAM User and add it to the group](/images/5-Workshop/5.2-Prerequisite/image5.2.1-5c.png)

![Create an IAM User and add it to the group](/images/5-Workshop/5.2-Prerequisite/image5.2.1-5d.png)

**Step 1.6 — Create an Access Key and configure the AWS CLI**

Create an Access key for the IAM User **(Security credentials → Create access key)**

![Configure AWS with the access key](/images/5-Workshop/5.2-Prerequisite/image5.2.1-6a.png)

![Configure AWS with the access key](/images/5-Workshop/5.2-Prerequisite/image5.2.1-6b.png)

![Configure AWS with the access key](/images/5-Workshop/5.2-Prerequisite/image5.2.1-6c.png)

![Configure AWS with the access key](/images/5-Workshop/5.2-Prerequisite/image5.2.1-6d.png)

**Step 1.7 — Configure AWS Budgets**

Go to **Billing and Cost Management → Budgets → Create budget**, choose the **Cost budget** type, set a monthly threshold, alerts at 50%/80%/100%, and enter the whole team's email.

![Create a cost alert Budget](/images/5-Workshop/5.2-Prerequisite/image5.2.1-7a.png)

![Create a cost alert Budget](/images/5-Workshop/5.2-Prerequisite/image5.2.1-7b.png)

![Create a cost alert Budget](/images/5-Workshop/5.2-Prerequisite/image5.2.1-7c.png)


#### Next content

- [Configure HCP Terraform](../5.2.2-Configure-HCP-Terraform/_index.md)
- [Prepare Terraform Code](../5.2.3-Prepare-Terraform-Code/_index.md)

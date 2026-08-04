---
title: "Configure HCP Terraform"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.2.2 </b> "
---

### Create an Organization for the project

**Step 2.1 — Create an account and Organization**

Go to [https://app.terraform.io/app](https://app.terraform.io/app) and sign in. If you don't have an account yet, choose Sign up to create a new one. On the main screen, choose Create organization to start creating an organization for the project (for example `RAG-on-AWS`).

![Create an Organization on HCP Terraform](/images/5-Workshop/5.2-Prerequisite/image5.2.2-1a.png)

![Create an Organization on HCP Terraform](/images/5-Workshop/5.2-Prerequisite/image5.2.2-1b.png)

**Step 2.2 — Create a Workspace**

Create a new **Workspace**, choose either the **CLI-driven** workflow (Terraform runs locally, HCP Terraform only stores the state) or the **Version control workflow** (runs automatically when code is pushed to Git), depending on your team's decision. Here I'll choose **CLI-Driven**. Then click **Create** to finish.

![Create a Workspace on HCP Terraform](/images/5-Workshop/5.2-Prerequisite/image5.2.2-2a.png)

![Create a Workspace on HCP Terraform](/images/5-Workshop/5.2-Prerequisite/image5.2.2-2b.png)

**Step 2.3 — Configure AWS credentials environment variables**

In the Workspace you just created, go to **Variables**, and add 2 environment variables (marked as **Sensitive**): `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` of the IAM User created in the previous section.

![Add AWS credentials to Workspace Variables](/images/5-Workshop/5.2-Prerequisite/image5.2.2-3a.png)

![Add AWS credentials to Workspace Variables](/images/5-Workshop/5.2-Prerequisite/image5.2.2-3b.png)

**Step 2.4 — Connect local Terraform CLI to HCP Terraform**

Run `terraform login` to authenticate the token, then declare the `cloud` block in your Terraform configuration file:

```hcl
terraform {
  cloud {
    organization = "RAG-on-AWS"
    workspaces {
      name = "RAG-app"
    }
  }
}
```

Run `terraform init` to confirm a successful connection to HCP Terraform.

Run `terraform apply` to start the first run for this workspace.

#### Next content

- [Configure AWS Credentials](../5.2.1-Configure-AWS-Credentials/_index.md)
- [Prepare Terraform Code](../5.2.3-Prepare-Terraform-Code/_index.md)

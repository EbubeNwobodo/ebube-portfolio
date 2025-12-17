---
title: "From ClickOps to IaC: Automating Infrastructure with Terraform"
date: 2025-12-16
summary: "How moving from manual portal clicks to Terraform modules reduced environment provisioning time by 30%."
tags: ["Terraform", "IaC", "Azure", "DevOps", "Automation"]
cover:
  image: "images/intro-terraform-workflow.png" # Standard TF workflow image
  alt: "Terraform Workflow"
  caption: "Write -> Plan -> Apply"
---

## The "ClickOps" Trap

We have all been there. You need a new development environment, so you log into the Azure Portal or AWS Console. You click "Create Resource," select your region, configure networking, and 20 minutes later, you have a VM.

It feels productive, but it's a trap.

Manual provisioning (affectionately called "ClickOps") is:
* **Unrepeatable:** Can you guarantee you clicked the exact same checkboxes for the Staging environment?
* **Un-versioned:** Who changed the firewall rule last Tuesday? There is no git commit history for a UI click.
* **Slow:** Scaling out to 50 servers manually is a nightmare.

## The Shift to Infrastructure as Code (IaC)

In my recent role at **Snapnet**, I led the adoption of Terraform to replace these manual workflows. The goal was simple: treat infrastructure exactly like software.

By defining our cloud resources in code, we achieved a **30% reduction in environment provisioning times**. But more importantly, we eliminated "configuration drift"—the silent killer where Dev and Prod environments slowly become different over time.

## Key Concept: The Power of Modules

The biggest mistake beginners make with Terraform is writing one giant `main.tf` file. To truly scale, you need **Modules**. Modules allow you to create reusable components (like a standard "Web Server" or "VNet") that can be deployed repeatedly with different parameters.

### Real-World Example: A Standard Azure Resource Group

Instead of defining tags and locations every time, I use a reusable module.

**The Module (`modules/resource_group/main.tf`):**
```hcl
resource "azurerm_resource_group" "rg" {
  name     = var.name
  location = var.location

  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = "Ebube Nwobodo"
  }
}
```

**Using the Module (`main.tf`):**
```hcl
module "dev_network_rg" {
  source      = "./modules/resource_group"
  name        = "rg-network-dev-eus"
  location    = "East US"
  environment = "Development"
}

module "prod_network_rg" {
  source      = "./modules/resource_group"
  name        = "rg-network-prod-eus"
  location    = "East US"
  environment = "Production"
}
```

### Managing State Safely
One challenge with Terraform is the State File (`terraform.tfstate`). This file tracks the mapping between your code and the real world.

Never store your state file in Git. It contains sensitive data (like keys) and creates locking issues if two people run `terraform apply` at the same time.

Instead, I always configure a **Remote Backend** using Azure Storage Account or AWS S3 with DynamoDB locking:
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

## Conclusion
Moving to Terraform wasn't just about saving time; it was about confidence. Now, when we need to spin up a Disaster Recovery (DR) region, it isn't a week-long project of reading documentation and clicking buttons. It's a single command:

`terraform apply -var="region=westus2"`
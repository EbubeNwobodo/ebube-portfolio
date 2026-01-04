---
title: "Architecting Multi-Region Disaster Recovery for Tier-1 Workloads"
date: 2026-01-04
summary: "Designing an Active-Active cloud architecture using Azure Site Recovery and Traffic Manager to cut RTO from 6 hours to under 1 hour."
tags: ["Azure", "Disaster Recovery", "Terraform", "Networking", "Resilience"]
weight: 5
cover:
  image: "images/disaster-recovery-smb-azure-site-recovery.png" # Optional placeholder
  alt: "Multi-Region DR Architecture"
  caption: "Active-Active Routing with Automatic Failover"
---

## The Problem
For a critical Tier-1 application, the existing Disaster Recovery (DR) strategy was insufficient. The Recovery Time Objective (RTO) stood at **6 hours**, meaning a regional outage could cripple operations for nearly a full business day.

Furthermore, provisioning a recovery environment was a manual, error-prone process that took up to **4 days** to complete, making regular DR drills nearly impossible.

## The Solution
I architected a **Multi-Region Active-Active** solution on Azure to ensure continuous availability. By leveraging **Azure Site Recovery (ASR)** for replication and **Azure Traffic Manager** for global routing, we transformed a fragile system into a resilient fortress.



### Key Architecture Decisions
1.  **Global Routing:** Implemented **Azure Traffic Manager** to distribute traffic across regions. In the event of a primary region failure, traffic is automatically rerouted to the secondary region.
2.  **Data Resilience:** Enabled **Geo-Redundant Storage (GRS)** to ensure data durability across geographic boundaries.
3.  **Infrastructure as Code:** Redesigned the Azure Landing Zone using **Terraform** to ensure the DR region was an exact, version-controlled replica of production.

## Technical Implementation

### 1. Automated Failover Routing
The core of this solution was the Traffic Manager profile. I configured it with "Priority" routing to handle failover logic automatically.

### 2. Infrastructure Consistency
To solve the "4-day provisioning" problem, I used Terraform modules to define the network topology (Hub-and-Spoke) and compute resources. This allowed us to spin up the DR landing zone in hours, not days.

### Code Snippet: Terraform for Traffic Manager
*This configuration defines the global entry point that manages traffic between the Primary (East US) and Secondary (West US) regions.*

```hcl
resource "azurerm_traffic_manager_profile" "tm_profile" {
  name                   = "global-app-routing"
  resource_group_name    = azurerm_resource_group.rg.name
  traffic_routing_method = "Priority"

  dns_config {
    relative_name = "app-global"
    ttl           = 60
  }

  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = "/health"
    interval_in_seconds          = 30
    timeout_in_seconds           = 10
    tolerated_number_of_failures = 3
  }
}

# Primary Endpoint (Active)
resource "azurerm_traffic_manager_azure_endpoint" "primary" {
  name               = "primary-region-endpoint"
  profile_id         = azurerm_traffic_manager_profile.tm_profile.id
  target_resource_id = azurerm_public_ip.primary_pip.id
  priority           = 1
}

# Secondary Endpoint (Passive/DR)
resource "azurerm_traffic_manager_azure_endpoint" "secondary" {
  name               = "secondary-region-endpoint"
  profile_id         = azurerm_traffic_manager_profile.tm_profile.id
  target_resource_id = azurerm_public_ip.secondary_pip.id
  priority           = 2
}
```
## Business Impact
1. **RTO Slashed by 83%:** Reduced recovery time from 6 hours to <1 hour for critical workloads.

2. **Drill Readiness:** Reduced the time required to provision a DR environment from 4 days to a few hours, enabling frequent, low-stress recovery drills.

3. **Business Continuity:** Achieved alignment with corporate compliance goals for data availability and resilience.
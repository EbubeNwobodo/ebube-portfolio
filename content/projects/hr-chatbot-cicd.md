---
title: "Deploying a Standalone AI Service for HR Automation"
date: 2025-06-01
summary: "Architecting a multi-tenant AI chatbot as a standalone microservice to enhance an existing HR software ecosystem."
tags: ["GitHub Actions", "Kubernetes", "Azure", "Microservices", "AI Integration", "Python"]
weight: 4
cover:
  image: "images/icons8-github-500.png" # Placeholder
  alt: "CI/CD Pipeline Visualization"
  caption: "Automated workflow from Code Push to Production"
---

## The Problem
Our core HR solution software needed an intelligent upgrade. While the platform managed records effectively, HR teams were still overwhelmed by repetitive employee inquiries about organizational policies. 

We needed to introduce an **AI-powered feature** to handle these queries autonomously. However, integrating this directly into the monolithic legacy codebase was risky. The challenge was to build and deploy this feature as a **standalone service** that could scale independently without disrupting the main application.

## The Solution
I engineered a decoupled CI/CD pipeline for this new AI microservice. By treating the chatbot as a standalone entity, we ensured that updates to the AI logic (which happen frequently) could be deployed without redeploying the entire HR suite.

### Architecture Overview
* **Microservice Design:** The AI feature runs as an isolated container, communicating with the main HR platform via APIs.
* **Staging (ACI):** Uses **Azure Container Instances** for rapid testing of new AI models and logic.
* **Production (AKS):** Deploys to **Azure Kubernetes Service** as a scalable service within the larger cluster ecosystem.

## Technical Implementation

### 1. Automated Testing & Linting
Because this service handles sensitive employee queries, code quality is non-negotiable. Every push triggers an integration test job using `flake8` for syntax and `pytest` for API validation.

### 2. Dual-Target Deployment Strategy
The pipeline intelligently routes deployments based on the target environment to optimize for both cost (Staging) and reliability (Production).

* **Staging:** Tears down and rebuilds a lightweight Linux container (4 vCPU, 8GB RAM).
* **Production:** Performs a rolling update on the Kubernetes cluster, ensuring the AI service remains available to users during upgrades.

### Code Snippet: Conditional Deployment Logic
*This workflow demonstrates how the pipeline distinguishes between the test environment and the live production cluster.*

```yaml
      - name: Set Deployment Variables
        id: set_vars
        run: |
          if [[ "${{ inputs.environment }}" == "production" ]]; then
            echo "RESOURCE_GROUP=HCM-Production" >> $GITHUB_ENV
            echo "DEPLOYMENT_TYPE=AKS" >> $GITHUB_ENV
            echo "NAMESPACE=hcmatrix-prod" >> $GITHUB_ENV
          else # Staging (development)
            echo "RESOURCE_GROUP=HCM-Staging" >> $GITHUB_ENV
            echo "DEPLOYMENT_TYPE=ACI" >> $GITHUB_ENV
            echo "ACI_CONTAINER_NAME=hcmatrix-staging-container" >> $GITHUB_ENV
          fi

      - name: Deploy to ACI (Staging Only)
        if: env.DEPLOYMENT_TYPE == 'ACI'
        run: |
          echo "Creating new container instance with image tag: ${{ env.IMAGE_TAG }}..."
          az container create --resource-group ${{ env.RESOURCE_GROUP }} \
            --name ${{ env.ACI_CONTAINER_NAME }} \
            --image ${{ env.IMAGE_PULL_NAME }}:${{ env.IMAGE_TAG }} \
            --dns-name-label $DNS_NAME \
            --cpu 4 --memory 8
```

## Business Impact
1. **Accelerated Feedback Loop:** Developers get instant feedback on their code quality via the automated linting stage.
2. **Reduced HR Workload:** The stable deployment of the chatbot ensures it is always available to answer employee questions, freeing up HR staff.
3. **Safe Releases:** By separating Staging (ACI) and Production (AKS), we ensure that bugs are caught in the cheap, transient environment before they ever reach the live user base.

Tech Stack: `GitHub Actions, Azure AKS, Azure ACI, Docker, Python, Flake8`.
# Beatit-AI — Model Deployment Pipeline (AWS SageMaker + CodePipeline)

> **Part of the [Beatit-AI Churn Prediction System](https://github.com/chetnapriyadarshini-iiit)** — a production MLOps system for predicting music streaming subscriber churn on AWS.

This repository implements the **SageMaker Endpoint Deployment Pipeline** — automatically detecting newly approved models in the SageMaker Model Registry and deploying them to real-time inference endpoints via a staged CodePipeline with a manual promotion gate between staging and production.

---

## Table of Contents

- [Overview](#overview)
- [Deployment Architecture](#deployment-architecture)
- [Staging → Production Promotion Flow](#staging--production-promotion-flow)
- [Repository Structure](#repository-structure)
- [Technologies Used](#technologies-used)
- [Setup and Configuration](#setup-and-configuration)
- [Related Repositories](#related-repositories)
- [Contact](#contact)

---

## Overview

When the training pipeline registers and approves a new model version in the SageMaker Model Registry, this deployment pipeline is automatically triggered. It provisions two SageMaker real-time inference endpoints — staging and production — using CloudFormation infrastructure-as-code, with a manual approval gate between the two environments ensuring human oversight before production promotion.

---

## Deployment Architecture

```
SageMaker Model Registry
(New model approved)
         │
         ▼
  AWS CodePipeline triggered
         │
         ▼
┌────────────────────────┐
│  Build Stage           │  buildspec.yml + build.py
│  · Fetch latest        │  Gets approved ModelPackage ARN
│    approved model ARN  │  Exports staging-config.json
│  · Package CloudFormation  prod-config.json
└──────────┬─────────────┘
           │
           ▼
┌────────────────────────┐
│  Staging Deployment    │  endpoint-config-template.yml
│  · Deploy staging      │  CloudFormation stack
│    endpoint            │
│  · Run integration     │  test/test.py
│    tests               │
└──────────┬─────────────┘
           │  ✅ Tests pass
           ▼
  ┌─────────────────┐
  │  Manual Approval │  AWS CodePipeline Console
  │  Gate            │  Human review required
  └────────┬─────────┘
           │  ✅ Approved
           ▼
┌────────────────────────┐
│  Production Deployment │  prod-config.json
│  · Deploy production   │  CloudFormation stack
│    endpoint            │
│  · Live inference      │
│    available           │
└────────────────────────┘
```

---

## Staging → Production Promotion Flow

1. Model approved in SageMaker Model Registry → CodePipeline triggered automatically
2. `build.py` fetches the latest approved `ModelPackage` ARN and injects it into configuration files
3. CloudFormation deploys a **staging endpoint** with the configuration from `staging-config.json`
4. Integration tests in `test/test.py` invoke the staging endpoint and validate response format and latency
5. CodePipeline pauses at a **manual approval step** — navigate to the CodePipeline console to approve
6. CloudFormation deploys the **production endpoint** with `prod-config.json` configuration
7. Production endpoint is live; model monitor pipeline is triggered automatically

---

## Repository Structure

```
beatit-ai-model-deploy/
├── build.py                        # Fetches approved model ARN; exports configs
├── buildspec.yml                   # CodeBuild instructions for the build stage
├── endpoint-config-template.yml    # CloudFormation template for SageMaker endpoints
├── staging-config.json             # Staging endpoint: instance type, count
├── prod-config.json                # Production endpoint: instance type, count
└── test/
    ├── buildspec.yml               # CodeBuild instructions for test stage
    └── test.py                     # Integration tests: invoke staging endpoint
```

---

## Technologies Used

| Service / Tool | Purpose |
|---|---|
| **AWS CodePipeline** | CI/CD orchestration for the deployment workflow |
| **AWS CodeBuild** | Build and test execution environments |
| **AWS CloudFormation** | Infrastructure-as-code for SageMaker endpoint provisioning |
| **AWS SageMaker Endpoints** | Real-time inference serving |
| **SageMaker Model Registry** | Source of approved model packages |
| **Python** | `build.py` model ARN resolution; `test.py` integration tests |

---

## Setup and Configuration

Instance type and count for each environment are configured in the JSON files:

```json
// staging-config.json
{
  "Parameters": {
    "StagingInstanceType": "ml.m5.large",
    "StagingInstanceCount": "1"
  }
}

// prod-config.json
{
  "Parameters": {
    "ProdInstanceType": "ml.m5.xlarge",
    "ProdInstanceCount": "2"
  }
}
```

To complete the staging → production promotion, navigate to the CodePipeline console, locate the manual approval action, review the staging test results, and click **Approve**.

---

## Related Repositories

| Repository | Role |
|---|---|
| [beatit-ai-glue-redshift-tables](https://github.com/chetnapriyadarshini-iiit/beatit-ai-glue-redshift-tables) | Upstream data pipeline producing model features |
| [beatit-ai-model-train](https://github.com/chetnapriyadarshini-iiit/beatit-ai-model-train) | Produces the approved model deployed by this pipeline |
| [beatit_ai_common_utilites](https://github.com/chetnapriyadarshini-iiit/beatit_ai_common_utilites) | Shared utilities |
| [beatit-ai-model-monitor](https://github.com/chetnapriyadarshini-iiit/beatit-ai-model-monitor) | Automatically triggered after production endpoint goes InService |

---

## Contact

Created by [@chetnapriyadarshini](https://github.com/chetnapriyadarshini) — feel free to reach out with questions or suggestions.

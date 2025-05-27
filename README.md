# MQ Queue Management

Automated IBM MQ queue management using GitOps principles. This repository manages MQ queue definitions as code, with automatic deployment to Kubernetes-hosted IBM MQ instances via GitHub Actions.

## 📋 Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Queue Definition Standards](#queue-definition-standards)
- [GitHub Actions Workflow](#github-actions-workflow)
- [Getting Started](#getting-started)
- [Usage Examples](#usage-examples)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## 🎯 Overview

This repository implements Infrastructure as Code (IaC) for IBM MQ queue management:

- **Version Control**: All queue definitions stored in Git
- **Automated Deployment**: Changes automatically applied via GitHub Actions
- **Environment Separation**: Separate configurations for dev/test/prod
- **Audit Trail**: Complete history of all queue changes
- **Idempotent Operations**: Safe to run multiple times

## 📁 Repository Structure

```
mq-queue-management/
├── .github/
│   └── workflows/
│       └── manage-queues.yml    # GitHub Actions workflow
├── mqsc/
│   ├── dev/                     # Development environment
│   │   ├── app1-queues.mqsc    # Application 1 queues
│   │   └── app2-queues.mqsc    # Application 2 queues
│   ├── test/                    # Test environment
│   │   └── (queue definitions)
│   └── prod/                    # Production environment
│       └── (queue definitions)
└── README.md                    # This file
```

## 📝 Queue Definition Standards

### MQSC File Format

Each `.mqsc` file contains queue definitions in IBM MQ Script Commands format:

```sql
*******************************************************************
* Application: Order Processing System
* Environment: Development
* Last Modified: 2024-01-15
*******************************************************************

DEFINE QLOCAL('APP.ORDER.REQUEST') +
       DESCR('Incoming order requests') +
       DEFPSIST(YES) +              * Persistent messages
       MAXDEPTH(10000) +             * Maximum 10k messages
       MAXMSGL(4194304) +            * Max message size 4MB
       SHARE +                       * Shareable queue
       DEFSOPT(SHARED) +            * Default share option
       REPLACE                       * Update if exists

DEFINE QLOCAL('APP.ORDER.DLQ') +
       DESCR('Order processing dead letter queue') +
       DEFPSIST(YES) +
       MAXDEPTH(100000) +
       REPLACE
```

### Naming Conventions

- **Application Queues**: `APP.<SYSTEM>.<PURPOSE>` (e.g., `APP.ORDER.REQUEST`)
- **System Queues**: `SYSTEM.<PURPOSE>` (e.g., `SYSTEM.AUDIT.LOG`)
- **Dead Letter Queues**: `<PREFIX>.DLQ` (e.g., `APP.ORDER.DLQ`)
- **Backout Queues**: `<QNAME>.BACKOUT` (e.g., `APP.PAYMENT.BACKOUT`)

## 🚀 GitHub Actions Workflow

### Automatic Triggers

The workflow runs automatically when:
- Changes are pushed to `.mqsc` files in the `main` branch
- Manual trigger via GitHub Actions UI

### Workflow Steps

1. **Configure AWS Credentials** - Authenticates with AWS
2. **Update Kubeconfig** - Connects to EKS cluster
3. **Get MQ Pod** - Identifies the running MQ instance
4. **Verify Queue Manager** - Ensures MQ is ready
5. **Apply Queue Definitions** - Applies all MQSC files for the environment
6. **Verify Queues** - Lists created/updated queues
7. **Export Definitions** - Backs up current queue state

### Manual Workflow Execution

1. Go to the **Actions** tab in GitHub
2. Select **MQ Queue Management**
3. Click **Run workflow**
4. Select environment (dev/test/prod)
5. Click **Run workflow**

## 🏁 Getting Started

### Prerequisites

1. **AWS Secrets** - Configure in GitHub repository settings:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_REGION`
   - `EKS_CLUSTER_NAME`

2. **IBM MQ** - Running in Kubernetes with:
   - Namespace: `ibm-mq-ns`
   - Queue Manager: `secureapphelm`

### Initial Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/your-org/mq-queue-management.git
   cd mq-queue-management
   ```

2. Create queue definitions:
   ```bash
   # Create a new queue file
   touch mqsc/dev/my-app-queues.mqsc
   ```

3. Add queue definitions (see examples below)

4. Commit and push:
   ```bash
   git add mqsc/dev/my-app-queues.mqsc
   git commit -m "Add queues for my application"
   git push origin main
   ```

## 💡 Usage Examples

### Adding a New Queue

```sql
* File: mqsc/dev/notification-queues.mqsc
DEFINE QLOCAL('APP.NOTIFICATION.EMAIL') +
       DESCR('Email notification queue') +
       DEFPSIST(YES) +
       MAXDEPTH(50000) +
       MAXMSGL(1048576) +
       REPLACE
```

### Updating Queue Properties

```sql
* Increase max depth for Black Friday traffic
DEFINE QLOCAL('APP.ORDER.REQUEST') +
       DESCR('Order queue - increased capacity') +
       MAXDEPTH(50000) +       * Increased from 10000
       REPLACE
```

### Adding Triggered Queues

```sql
DEFINE QLOCAL('APP.INVENTORY.UPDATE') +
       DESCR('Inventory updates with trigger') +
       DEFPSIST(YES) +
       TRIGGER +
       TRIGTYPE(FIRST) +
       TRIGDATA('START_INVENTORY_PROCESSOR') +
       REPLACE
```

### Setting Up Dead Letter Queue

```sql
DEFINE QLOCAL('APP.PAYMENT.REQUEST') +
       DESCR('Payment processing') +
       DEFPSIST(YES) +
       BOTHRESH(3) +                    * Backout threshold
       BOQNAME('APP.PAYMENT.BACKOUT') + * Backout queue
       REPLACE

DEFINE QLOCAL('APP.PAYMENT.BACKOUT') +
       DESCR('Failed payment transactions') +
       DEFPSIST(YES) +
       REPLACE
```

## ✅ Best Practices

### 1. **Always Use REPLACE**
```sql
DEFINE QLOCAL('QUEUE.NAME') +
       ... +
       REPLACE  * Always include this
```

### 2. **Add Descriptive Comments**
```sql
*******************************************************************
* Purpose: Handle real-time order processing
* Created: 2024-01-15
* Modified: 2024-01-20 - Increased depth for peak season
*******************************************************************
```

### 3. **Set Appropriate Limits**
- `MAXDEPTH`: Based on expected volume
- `MAXMSGL`: Based on message size requirements
- `BOTHRESH`: For poison message handling

### 4. **Use Consistent Naming**
- Follow naming conventions
- Include environment in descriptions
- Use meaningful queue names

### 5. **Test in Lower Environments First**
```bash
# Deploy to dev first
git push origin main  # Triggers dev deployment

# After testing, copy to test/prod
cp mqsc/dev/app-queues.mqsc mqsc/test/
cp mqsc/dev/app-queues.mqsc mqsc/prod/
```

## 🔧 Troubleshooting

### Common Issues

#### 1. **Queue Already Exists Error**
```
AMQ8150E: IBM MQ queue already exists
```
**Solution**: Add `REPLACE` keyword to queue definition

#### 2. **Invalid MQSC Syntax**
```
AMQ8405E: Syntax error detected
```
**Solution**: Check for:
- Missing `+` on continuation lines
- Unmatched quotes in descriptions
- Invalid parameter values

#### 3. **Workflow Fails to Find MQ Pod**
**Solution**: Verify:
- Correct namespace: `ibm-mq-ns`
- MQ pod is running: `kubectl get pods -n ibm-mq-ns`
- AWS credentials are correct

### Viewing Queue Status

SSH into the cluster and run:
```bash
# List all queues
kubectl exec -it <mq-pod> -n ibm-mq-ns -- \
  runmqsc secureapphelm <<< "DISPLAY QLOCAL(*)"

# Check specific queue
kubectl exec -it <mq-pod> -n ibm-mq-ns -- \
  runmqsc secureapphelm <<< "DISPLAY QLOCAL('APP.ORDER.REQUEST') ALL"
```

### Workflow Logs

1. Go to **Actions** tab
2. Click on the workflow run
3. Click on failed step to see detailed logs
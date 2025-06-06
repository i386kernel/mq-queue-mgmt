# MQ Queue Management - Job Based Workflow

This GitHub Actions workflow automates the deployment and management of IBM MQ queue configurations across multiple environments using Kubernetes Jobs on AWS EKS.

## Overview

The workflow provides a robust, automated solution for applying MQ Script Command (MQSC) files to IBM MQ queue managers running in Kubernetes. It includes validation, deployment, verification, and cleanup capabilities with support for multiple environments.

## Features

- **Multi-Environment Support**: Deploy to dev, test, and prod environments
- **Validation**: Syntax validation of MQSC files before deployment
- **Dry Run Mode**: Test configurations without applying changes
- **Configuration Tracking**: Hash-based change detection and deployment records
- **Automatic Cleanup**: Removes old ConfigMaps and Jobs
- **Comprehensive Logging**: Detailed logs for troubleshooting
- **Rollback Safety**: Maintains deployment history for rollback scenarios

## Prerequisites

### GitHub Secrets Required

Configure the following secrets in your GitHub repository:

| Secret Name | Description |
|-------------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS Access Key ID for EKS access |
| `AWS_SECRET_ACCESS_KEY` | AWS Secret Access Key |
| `AWS_REGION` | AWS region where EKS cluster is located |
| `EKS_CLUSTER_NAME` | Name of your EKS cluster |

### Repository Structure

```
your-repo/
├── .github/
│   └── workflows/
│       └── mq-queue-management.yml
├── mqsc/
│   ├── dev/
│   │   ├── app-queues.mqsc
│   │   └── queue-manager-config.mqsc
│   ├── test/
│   │   ├── app-queues.mqsc
│   │   └── queue-manager-config.mqsc
│   └── prod/
│       ├── app-queues.mqsc
│       └── queue-manager-config.mqsc
└── k8s/
    └── jobs/
        └── (optional job configurations)
```

### Kubernetes Requirements

- IBM MQ deployed on Kubernetes with Helm
- Queue manager accessible within the cluster
- Persistent Volume Claim for MQ data
- Appropriate RBAC permissions for job creation

## Configuration

### Environment Variables

Update these variables in the workflow file to match your setup:

```yaml
env:
  MQ_NAMESPACE: ibm-mq-ns        # Kubernetes namespace for MQ resources
  QMGR_NAME: secureapphelm       # Name of your queue manager
```

### MQSC File Structure

Each environment should have its own directory under `mqsc/` containing `.mqsc` files:

```bash
# Example MQSC file content - Application Queues
DEFINE QLOCAL('MY.FIRST.QUEUE') +
       DESCR('My first queue via GitOps') +
       DEFPSIST(YES) +
       MAXDEPTH(5000) +
       REPLACE

DEFINE QLOCAL('MY.SECOND.QUEUE') +
       DESCR('My SECOND queue via GitOps') +
       DEFPSIST(YES) +
       MAXDEPTH(5000) +
       REPLACE

DEFINE QLOCAL('MY.THIRD.QUEUE') +
       DESCR('My THIRD queue via GitOps') +
       DEFPSIST(YES) +
       MAXDEPTH(5000) +
       REPLACE

# Additional MQSC commands examples
ALTER QMGR MAXMSGL(104857600)
```

## Usage

### Automatic Triggers

The workflow automatically runs when:
- Changes are pushed to the `main` branch
- Files in `mqsc/**/*.mqsc` are modified
- Files in `k8s/jobs/**/*.yaml` are modified

### Manual Execution

1. Go to **Actions** tab in your GitHub repository
2. Select **MQ Queue Management - Job Based** workflow
3. Click **Run workflow**
4. Configure parameters:
   - **Environment**: Choose dev, test, or prod
   - **Dry run**: Enable to validate without applying changes

## Workflow Process

### Stage 1: Validation (`validate-configs`)

1. **Checkout Code**: Retrieves the latest repository content
2. **Validate MQSC Syntax**: 
   - Checks if environment directory exists
   - Verifies presence of `.mqsc` files
   - Validates basic MQSC command syntax
3. **Generate Job Identifier**: Creates unique job name with timestamp
4. **Generate Config Hash**: Creates 8-character hash for change tracking

### Stage 2: Deployment (`deploy-queues`)

1. **Setup Tools**: Installs AWS CLI and kubectl
2. **Configure AWS**: Sets up credentials and kubeconfig for EKS
3. **Create ConfigMap**: Stores MQSC files in Kubernetes ConfigMap
4. **Deploy Job**: Creates and applies Kubernetes Job manifest
5. **Wait for Completion**: Monitors job execution (max 10 minutes)
6. **Show Logs**: Displays job logs for debugging
7. **Verify Configuration**: Connects to queue manager for verification
8. **Cleanup**: Removes old ConfigMaps (keeps 5 most recent)
9. **Record Deployment**: Creates deployment record for tracking

## Job Specification

The Kubernetes Job created by this workflow:

- **Image**: `ibmcom/mq:latest`
- **Restart Policy**: Never
- **Backoff Limit**: 2 retries
- **TTL**: 300 seconds (auto-cleanup after 5 minutes)
- **Volumes**: 
  - ConfigMap with MQSC files
  - Persistent volume for MQ data access

## Monitoring and Troubleshooting

### Viewing Deployment Status

```bash
# List recent jobs
kubectl get jobs -n ibm-mq-ns -l app.kubernetes.io/name=mq-config-job

# Check job logs
kubectl logs job/<job-name> -n ibm-mq-ns

# View deployment records
kubectl get configmaps -n ibm-mq-ns -l app.kubernetes.io/name=deployment-record
```

### Common Issues

| Issue | Solution |
|-------|----------|
| Job fails to start | Check RBAC permissions and namespace access |
| Queue manager not ready | Verify MQ deployment status and PVC mounting |
| MQSC syntax errors | Use dry-run mode to validate before deployment |
| ConfigMap creation fails | Check namespace permissions and resource quotas |

### Debugging Steps

1. **Check GitHub Actions logs** for detailed error messages
2. **Verify MQSC file syntax** using dry-run mode
3. **Confirm queue manager status** in Kubernetes
4. **Review job logs** using kubectl commands above
5. **Validate AWS/EKS connectivity** and permissions

## Security Considerations

- **Secrets Management**: All sensitive data stored in GitHub Secrets
- **RBAC**: Jobs run with minimal required permissions
- **Network Isolation**: Jobs connect to MQ within cluster network
- **Audit Trail**: Complete deployment history maintained in ConfigMaps

## Customization

### Extending the Workflow

To customize for your environment:

1. **Modify environment variables** in the workflow file
2. **Adjust job specifications** (resources, timeouts, etc.)
3. **Add custom validation steps** in the validation job
4. **Implement custom notification** for deployment results
5. **Add environment-specific configurations**

### Adding New Environments

1. Create new directory under `mqsc/your-env/`
2. Add environment option to workflow dispatch inputs
3. Update any environment-specific logic if needed

## Best Practices

- **Test in lower environments** before production deployment
- **Keep MQSC files focused** (e.g., `app-queues.mqsc` for application queues, `queue-manager-config.mqsc` for QM settings)
- **Use meaningful MQSC file names** that reflect their purpose
- **Include REPLACE parameter** in DEFINE statements to handle updates safely
- **Review deployment logs** regularly for early issue detection
- **Maintain consistent naming conventions** across environments
- **Use dry-run mode** for validation during development

## Support

For issues related to:
- **IBM MQ**: Consult IBM MQ documentation
- **Kubernetes**: Check cluster and RBAC configurations  
- **AWS EKS**: Verify cluster connectivity and permissions
- **Workflow**: Review GitHub Actions logs and job outputs

## Contributing

1. Fork the repository
2. Create a feature branch
3. Test changes in development environment
4. Submit pull request with clear description
5. Ensure all checks pass before merging
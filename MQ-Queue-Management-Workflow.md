# GitHub Actions Workflow: MQ Queue Management - Line by Line Explanation

## Workflow Header and Triggers

Sets the workflow name that appears in GitHub Actions UI.
```yaml
name: MQ Queue Management - Job Based
```

Triggers workflow on pushes to `main` branch, but only when files in `mqsc/` directories (MQ script files) or `k8s/jobs/` directories (Kubernetes job YAML files) are changed.
```yaml
on:
  push:
    branches: [main]
    paths:
      - 'mqsc/**/*.mqsc'
      - 'k8s/jobs/**/*.yaml'
```

Enables manual workflow execution with an environment dropdown (dev/test/prod).
```yaml
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - test
          - prod
```

Adds optional dry-run parameter to validate configurations without applying them.
```yaml
      dry_run:
        description: 'Dry run (validate only)'
        required: false
        default: false
        type: boolean
```

## Environment Variables

Sets global environment variables for the MQ namespace and queue manager name.
```yaml
env:
  MQ_NAMESPACE: ibm-mq-ns
  QMGR_NAME: secureapphelm
```

## Job 1: Configuration Validation

Defines first job to validate configurations, running on Ubuntu with two outputs for downstream jobs.
```yaml
jobs:
  validate-configs:
    runs-on: ubuntu-latest
    outputs:
      config-hash: ${{ steps.hash.outputs.hash }}
      job-name: ${{ steps.jobname.outputs.name }}
```

Checks out the repository code.
```yaml
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
```

Starts MQSC validation step, setting environment variable (defaults to 'dev' if not provided).
```yaml
      - name: Validate MQSC syntax
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
```

Checks if the environment-specific MQSC directory exists, exits with error if not found.
```yaml
          # Validate MQSC files exist and have basic syntax
          if [ ! -d "./mqsc/${ENV}" ]; then
            echo "Error: No MQSC directory found for environment ${ENV}"
            exit 1
          fi
```

Verifies that .mqsc files exist in the environment directory.
```yaml
          # Check for MQSC files
          if [ -z "$(find ./mqsc/${ENV} -name '*.mqsc' -type f)" ]; then
            echo "Error: No .mqsc files found in ./mqsc/${ENV}"
            exit 1
          fi
```

Loops through each MQSC file and checks for valid MQ commands (DEFINE, ALTER, DELETE, DISPLAY, REPLACE).
```yaml
          # Basic syntax validation (check for common MQSC commands)
          for file in ./mqsc/${ENV}/*.mqsc; do
            echo "Validating $(basename $file)..."
            # Check for valid MQSC command structure
            if ! grep -E "^(DEFINE|ALTER|DELETE|DISPLAY|REPLACE)" "$file" > /dev/null; then
              echo "Warning: $file may not contain valid MQSC commands"
            fi
          done
```

Creates unique job name using environment, timestamp, and GitHub run number.
```yaml
      - name: Generate unique job identifier
        id: jobname
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
          TIMESTAMP=$(date +%Y%m%d-%H%M%S)
          JOB_NAME="mq-config-${ENV}-${TIMESTAMP}-${{ github.run_number }}"
          echo "name=$JOB_NAME" >> $GITHUB_OUTPUT
          echo "Generated job name: $JOB_NAME"
```

Generates 8-character hash of all MQSC files to track configuration changes.
```yaml
      - name: Generate config hash
        id: hash
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
          HASH=$(find ./mqsc/${ENV} -name '*.mqsc' -type f -exec sha256sum {} \; | sort | sha256sum | cut -d' ' -f1 | cut -c1-8)
          echo "hash=$HASH" >> $GITHUB_OUTPUT
          echo "Configuration hash: $HASH"
```

## Job 2: Queue Deployment

Second job that depends on validation, only runs if not in dry-run mode.
```yaml
  deploy-queues:
    needs: validate-configs
    runs-on: ubuntu-latest
    if: github.event.inputs.dry_run != 'true'
```

Check if this config was already deployed. Helps in tracking history, rollback mapping. 
```yaml
      - name: Check for configuration changes
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
          CURRENT_HASH="${{ needs.validate-configs.outputs.config-hash }}"
          
          # Get the last deployed hash for this environment
          LAST_DEPLOYED_HASH=$(kubectl get configmaps -n ${{ env.MQ_NAMESPACE }} \
            -l app.kubernetes.io/name=deployment-record,app.kubernetes.io/environment=${ENV} \
            --sort-by=.metadata.creationTimestamp \
            -o jsonpath='{.items[-1].data.config-hash}' 2>/dev/null || echo "")
          
          echo "Current config hash: $CURRENT_HASH"
          echo "Last deployed hash: $LAST_DEPLOYED_HASH"
          
          if [ "$CURRENT_HASH" = "$LAST_DEPLOYED_HASH" ] && [ -n "$LAST_DEPLOYED_HASH" ]; then
            echo "Configuration unchanged for $ENV environment, skipping deployment"
            exit 0
          else
            echo "Configuration changed, proceeding with deployment"
          fi
```

Installs AWS CLI if not already present.
```yaml
      - name: Setup AWS and kubectl
        run: |
          # Install AWS CLI
          if ! command -v aws &> /dev/null; then
            echo "Installing AWS CLI..."
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip -q awscliv2.zip
            sudo ./aws/install
          fi
```

Installs kubectl if not already present.
```yaml
          # Install kubectl
          if ! command -v kubectl &> /dev/null; then
            echo "Installing kubectl..."
            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
            chmod +x kubectl
            sudo mv kubectl /usr/local/bin/
          fi
```

Configures AWS credentials and updates kubeconfig for EKS cluster access.
```yaml
      - name: Configure AWS
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_REGION }}
        run: |
          aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
          aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
          aws configure set region $AWS_DEFAULT_REGION
          
          aws sts get-caller-identity
          aws eks update-kubeconfig --name ${{ secrets.EKS_CLUSTER_NAME }} --region ${{ secrets.AWS_REGION }}
```

Creates Kubernetes ConfigMap containing all MQSC files for the specific environment.
```yaml
      - name: Create ConfigMap for MQSC files
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
          CONFIG_HASH="${{ needs.validate-configs.outputs.config-hash }}"
          JOB_NAME="${{ needs.validate-configs.outputs.job-name }}"
          
          # Create ConfigMap from MQSC files
          kubectl create configmap ${JOB_NAME}-mqsc \
            --from-file=./mqsc/${ENV}/ \
            --namespace=${{ env.MQ_NAMESPACE }} \
            --dry-run=client -o yaml | kubectl apply -f -
```

Adds standardized labels to the ConfigMap for organization and cleanup.
```yaml
          # Label the ConfigMap
          kubectl label configmap ${JOB_NAME}-mqsc \
            --namespace=${{ env.MQ_NAMESPACE }} \
            app.kubernetes.io/name=mq-config-job \
            app.kubernetes.io/environment=${ENV} \
            app.kubernetes.io/version=${CONFIG_HASH} \
            app.kubernetes.io/managed-by=github-actions
```

Begins creating the Kubernetes Job manifest file.
```yaml
      - name: Create and Apply MQ Configuration Job
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
          JOB_NAME="${{ needs.validate-configs.outputs.job-name }}"
          
          # Create the Job manifest
          cat > /tmp/mq-config-job.yaml << EOF
```

Defines Job metadata with labels and annotations tracking deployment details.
```yaml
          apiVersion: batch/v1
          kind: Job
          metadata:
            name: ${JOB_NAME}
            namespace: ${{ env.MQ_NAMESPACE }}
            labels:
              app.kubernetes.io/name: mq-config-job
              app.kubernetes.io/environment: ${ENV}
              app.kubernetes.io/managed-by: github-actions
              app.kubernetes.io/version: ${{ needs.validate-configs.outputs.config-hash }}
            annotations:
              github.com/repository: ${{ github.repository }}
              github.com/commit: ${{ github.sha }}
              github.com/actor: ${{ github.actor }}
              github.com/run-id: ${{ github.run_id }}
```

Job specification with 5-minute cleanup timer, 2 retry attempts, and pod labels.
```yaml
          spec:
            ttlSecondsAfterFinished: 300  # Clean up job after 5 minutes
            backoffLimit: 2
            template:
              metadata:
                labels:
                  app.kubernetes.io/name: mq-config-job
                  app.kubernetes.io/environment: ${ENV}
```

Pod specification with IBM MQ container image and bash command.
```yaml
              spec:
                restartPolicy: Never
                # serviceAccountName: mq-config-job-sa  # Optional - uses default SA if omitted
                containers:
                - name: mq-config
                  image: ibmcom/mq:latest  # Use same image as your queue manager
                  command: ["/bin/bash"]
```

Bash script arguments with error handling and logging setup.
```yaml
                  args:
                    - -c
                    - |
                      set -e
                      echo "Starting MQ configuration job..."
                      echo "Queue Manager: ${{ env.QMGR_NAME }}"
                      echo "Environment: ${ENV}"
```

Waits for queue manager to be available before proceeding.
```yaml
                      # Wait for queue manager to be available
                      echo "Waiting for queue manager to be ready..."
                      until echo "DISPLAY QMGR" | runmqsc ${{ env.QMGR_NAME }} > /dev/null 2>&1; do
                        echo "Queue manager not ready, waiting 10 seconds..."
                        sleep 10
                      done
                      echo "Queue manager is ready!"
```

Executes each MQSC file against the queue manager.
```yaml
                      # Apply all MQSC files
                      for mqsc_file in /mqsc-config/*.mqsc; do
                        if [ -f "\$mqsc_file" ]; then
                          echo "Applying \$(basename \$mqsc_file)..."
                          if runmqsc ${{ env.QMGR_NAME }} < "\$mqsc_file"; then
                            echo "Successfully applied \$(basename \$mqsc_file)"
                          else
                            echo "Failed to apply \$(basename \$mqsc_file)"
                            exit 1
                          fi
                        fi
                      done
```

Verifies the configuration by displaying all local queues.
```yaml
                      # Verify configuration
                      echo "Verifying queue configuration..."
                      echo "DISPLAY QLOCAL(*)" | runmqsc ${{ env.QMGR_NAME }}
                      
                      echo "MQ configuration job completed successfully!"
```

Sets required environment variables for the MQ container.
```yaml
                  env:
                  - name: LICENSE
                    value: accept
                  - name: MQ_QMGR_NAME
                    value: ${{ env.QMGR_NAME }}
```

Mounts ConfigMap with MQSC files and MQ data volume.
```yaml
                  volumeMounts:
                  - name: mqsc-config
                    mountPath: /mqsc-config
                    readOnly: true
                  # Connect to the same MQ network
                  - name: mq-data
                    mountPath: /mnt/mqm
                    readOnly: true
```

Defines volumes and applies the Job to Kubernetes.
```yaml
                volumes:
                - name: mqsc-config
                  configMap:
                    name: ${JOB_NAME}-mqsc
                - name: mq-data
                  persistentVolumeClaim:
                    claimName: ${{ env.QMGR_NAME }}-pvc  # Adjust to match your PVC name
          EOF
          
          # Apply the job
          kubectl apply -f /tmp/mq-config-job.yaml
```

Waits up to 10 minutes for the Job to complete.
```yaml
      - name: Wait for Job Completion
        run: |
          JOB_NAME="${{ needs.validate-configs.outputs.job-name }}"
          
          echo "Waiting for job ${JOB_NAME} to complete..."
          
          # Wait for job to complete (max 10 minutes)
          kubectl wait --for=condition=Complete job/${JOB_NAME} \
            --namespace=${{ env.MQ_NAMESPACE }} \
            --timeout=600s
```

Checks final job status and exits with error if unsuccessful.
```yaml
          # Check job status
          JOB_STATUS=$(kubectl get job ${JOB_NAME} -n ${{ env.MQ_NAMESPACE }} -o jsonpath='{.status.conditions[0].type}')
          
          if [ "$JOB_STATUS" = "Complete" ]; then
            echo "Job completed successfully!"
          else
            echo "Job failed or did not complete"
            kubectl describe job ${JOB_NAME} -n ${{ env.MQ_NAMESPACE }}
            exit 1
          fi
```

Always shows job logs and status for debugging, regardless of success/failure.
```yaml
      - name: Show Job Logs
        if: always()
        run: |
          JOB_NAME="${{ needs.validate-configs.outputs.job-name }}"
          
          echo "=== Job Logs ==="
          kubectl logs job/${JOB_NAME} -n ${{ env.MQ_NAMESPACE }} || echo "No logs available"
          
          echo "=== Job Status ==="
          kubectl describe job ${JOB_NAME} -n ${{ env.MQ_NAMESPACE }} || echo "Job not found"
```

Verifies deployment by connecting directly to the queue manager pod and listing queues.
```yaml
      - name: Verify Queue Configuration
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
          
          # Get the queue manager pod for verification
          QM_POD=$(kubectl get pods -n ${{ env.MQ_NAMESPACE }} \
            -l app.kubernetes.io/name=ibm-mq,app.kubernetes.io/instance=${{ env.QMGR_NAME }} \
            -o jsonpath='{.items[0].metadata.name}')
          
          if [ -n "$QM_POD" ]; then
            echo "Verifying configuration on queue manager pod: $QM_POD"
            
            # Simple verification - list queues
            kubectl exec $QM_POD -n ${{ env.MQ_NAMESPACE }} -- \
              bash -c "echo 'DISPLAY QLOCAL(*)' | runmqsc ${{ env.QMGR_NAME }}" || echo "Verification failed"
          else
            echo "Warning: Could not find queue manager pod for verification"
          fi
```

Cleans up old ConfigMaps, keeping only the 5 most recent ones per environment.
```yaml
      - name: Cleanup ConfigMaps
        if: always()
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
          
          # Keep only the 5 most recent job ConfigMaps
          kubectl get configmaps -n ${{ env.MQ_NAMESPACE }} \
            -l app.kubernetes.io/name=mq-config-job,app.kubernetes.io/environment=${ENV} \
            --sort-by=.metadata.creationTimestamp \
            -o name | head -n -5 | xargs -r kubectl delete -n ${{ env.MQ_NAMESPACE }} || echo "No old ConfigMaps to clean up"
```

Creates a deployment record ConfigMap with metadata about successful deployments.
```yaml
      - name: Record Deployment
        if: success()
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"
          CONFIG_HASH="${{ needs.validate-configs.outputs.config-hash }}"
          JOB_NAME="${{ needs.validate-configs.outputs.job-name }}"
          
          # Create a deployment record
          kubectl create configmap deployment-record-$(date +%Y%m%d-%H%M%S) \
            --from-literal=environment=${ENV} \
            --from-literal=config-hash=${CONFIG_HASH} \
            --from-literal=git-commit=${{ github.sha }} \
            --from-literal=deployed-by=${{ github.actor }} \
            --from-literal=job-name=${JOB_NAME} \
            --namespace=${{ env.MQ_NAMESPACE }} \
            --dry-run=client -o yaml | kubectl apply -f -
```

## Workflow Summary

This workflow automates IBM MQ queue configuration deployment through a two-stage process:

1. **Validation Stage**: Validates MQSC files, generates unique identifiers, and creates configuration hashes
2. **Deployment Stage**: Creates Kubernetes Jobs that apply MQSC configurations to running MQ queue managers

Key features include environment-specific deployments, dry-run capability, automatic cleanup, deployment tracking, and comprehensive logging for troubleshooting.
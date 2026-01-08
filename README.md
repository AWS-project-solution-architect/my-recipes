# My Recipes - Multi-Region Kubernetes Deployment

This repository contains Helm charts and ArgoCD ApplicationSet configuration for deploying the My Recipes application across multiple AWS regions with External Secrets integration.

## Architecture

- **Multi-Region Deployment**: Supports deployment to `eu-central-1` and `us-east-1`
- **External Secrets**: All sensitive data managed via AWS Secrets Manager
- **ArgoCD ApplicationSet**: Automated deployment to multiple clusters
- **Route 53**: Latency-based routing across regions

## Prerequisites

1. **EKS Clusters** in target regions with:
   - AWS Load Balancer Controller installed
   - Labels: `region=eu-central-1` or `region=us-east-1`

2. **External Secrets Operator** installed on each cluster:
   ```bash
   helm repo add external-secrets https://charts.external-secrets.io
   helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace
   ```

3. **IAM Role for Service Account (IRSA)** configured for External Secrets Operator with permissions to access AWS Secrets Manager

4. **ArgoCD** installed with cluster credentials configured

## Setup Instructions

### Step 1: Create AWS Secrets

Create secrets in AWS Secrets Manager for each region. Use the provided script:

```bash
# Set your region
export AWS_REGION=eu-central-1

# Create all required secrets
aws secretsmanager create-secret \
  --name my-recipes/oauth/google-client-id \
  --secret-string "YOUR_GOOGLE_CLIENT_ID" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/oauth/google-client-secret \
  --secret-string "YOUR_GOOGLE_CLIENT_SECRET" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/oauth/azure-client-id \
  --secret-string "YOUR_AZURE_CLIENT_ID" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/oauth/azure-client-secret \
  --secret-string "YOUR_AZURE_CLIENT_SECRET" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/oauth/azure-tenant-id \
  --secret-string "consumers" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/email/smtp-user \
  --secret-string "your-email@gmail.com" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/email/smtp-password \
  --secret-string "YOUR_SMTP_APP_PASSWORD" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/s3/access-key \
  --secret-string "YOUR_AWS_ACCESS_KEY" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/s3/secret-key \
  --secret-string "YOUR_AWS_SECRET_KEY" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/admin/username \
  --secret-string "admin" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/admin/email \
  --secret-string "admin@example.com" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/admin/password \
  --secret-string "CHANGE_ME_STRONG_PASSWORD" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/admin/first-name \
  --secret-string "Admin" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name my-recipes/admin/last-name \
  --secret-string "User" \
  --region $AWS_REGION

# Database password
aws secretsmanager create-secret \
  --name my-recipes/db/postgres-password \
  --secret-string "YOUR_STRONG_DB_PASSWORD" \
  --region $AWS_REGION
```

**Repeat for `us-east-1` region.**

### Step 2: Configure IRSA for External Secrets Operator

Create an IAM policy for Secrets Manager access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:YOUR_ACCOUNT_ID:secret:my-recipes/*"
    }
  ]
}
```

Associate the policy with the External Secrets Operator service account:

```bash
eksctl create iamserviceaccount \
  --name external-secrets \
  --namespace external-secrets-system \
  --cluster YOUR_CLUSTER_NAME \
  --region eu-central-1 \
  --attach-policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/ExternalSecretsPolicy \
  --approve
```

### Step 3: Update ApplicationSet

Edit [applicationset.yaml](applicationset.yaml) and update the repository URL:

```yaml
source:
  repoURL: https://github.com/YOUR-ORG/my-recipes.git
```

### Step 4: Deploy ApplicationSet

Apply the ApplicationSet to your ArgoCD installation:

```bash
kubectl apply -f applicationset.yaml -n argocd
```

ArgoCD will automatically:
- Detect all clusters labeled with `region=eu-central-1` or `region=us-east-1`
- Deploy the application to each cluster
- Use region-specific values from `envs/` directory

### Step 5: Configure Route 53

After deployment, get the ALB DNS names from each region:

```bash
kubectl get ingress -n my-apps
```

Create Route 53 latency-based records:

1. **Backend API** (`api-my-recipes.aws-dvirlabs.com`):
   - Record 1: Alias to EU ALB, Region: eu-central-1, Latency routing
   - Record 2: Alias to US ALB, Region: us-east-1, Latency routing

2. **Frontend** (`my-recipes.aws-dvirlabs.com`):
   - Record 1: Alias to EU ALB, Region: eu-central-1, Latency routing
   - Record 2: Alias to US ALB, Region: us-east-1, Latency routing

## Repository Structure

```
my-recipes/
├── applicationset.yaml          # ArgoCD ApplicationSet for multi-cluster deployment
├── Chart.yaml                   # Helm chart metadata
├── values.yaml                  # Base values (no secrets)
├── envs/
│   ├── eu-central-1.yaml       # EU-specific configuration
│   └── us-east-1.yaml          # US-specific configuration
└── templates/
    ├── app-secrets.yaml         # ExternalSecret for application secrets
    ├── db-secret.yaml           # ExternalSecret for database password
    ├── backend-deployment.yaml
    ├── frontend-deployment.yaml
    ├── ingress.yaml
    └── ...
```

## Key Features

### External Secrets Integration

All sensitive data is stored in AWS Secrets Manager and synchronized to Kubernetes secrets via External Secrets Operator:

- OAuth credentials (Google, Azure)
- Email SMTP credentials
- S3 access keys
- Admin user credentials
- Database passwords

### Multi-Region Support

The application is deployed identically to multiple regions with region-specific:
- ECR repository URLs
- S3 bucket names
- ALB endpoints

### Automated Sync

ArgoCD automatically:
- Syncs changes from Git
- Prunes deleted resources
- Self-heals configuration drift

## Monitoring

Check application status in ArgoCD:

```bash
argocd app list
argocd app get my-recipes-eu-central-1
argocd app get my-recipes-us-east-1
```

## Troubleshooting

### External Secrets not syncing

Check the SecretStore and ExternalSecret status:

```bash
kubectl get secretstore -n my-apps
kubectl get externalsecret -n my-apps
kubectl describe externalsecret app-secrets -n my-apps
```

### ArgoCD not detecting clusters

Verify cluster labels:

```bash
kubectl get clusters -n argocd -o yaml
```

Ensure clusters have the `region` label set correctly.

## Security Notes

- Never commit sensitive data to Git
- All secrets are managed in AWS Secrets Manager
- Use IRSA for secure AWS access without static credentials
- Rotate secrets regularly in AWS Secrets Manager
- External Secrets will automatically sync updated values

## Support

For issues or questions, please open an issue in this repository.

# GitHub Actions Deployment

This repository contains two deployment workflows:

1. **deploy.yml** - Deploy static site to AWS S3 + CloudFront (for frontend only)
2. **deploy-eks.yml** - Deploy full Next.js app to AWS EKS (for frontend + API)

## Workflow Selection

Choose based on your needs:
- **S3 + CloudFront**: Static site hosting, lower cost, no API routes
- **EKS**: Full Next.js with API routes, higher cost, scalable

---

# 1. S3 + CloudFront Deployment (deploy.yml)

Deploy static site to AWS S3 and invalidate CloudFront cache with separate development and production environments.

## Configuration

### Hard-coded Values (in deploy.yml)

The following values are configured directly in `.github/workflows/deploy.yml`:

**Global Configuration:**
- `AI_PROVIDER`: anthropic
- `AI_MODEL`: claude-sonnet-4-5-20250929
- `AWS_ACCESS_KEY_ID`: YOUR_AWS_ACCESS_KEY_ID (update this)
- `AWS_REGION`: us-east-1

**Development Environment** (`release/dev` branch):
- `DEV_S3_BUCKET_NAME`: aiot-drawio-dev
- `DEV_CLOUDFRONT_DISTRIBUTION_ID`: E1GZOU957522YT

**Production Environment** (`release/production` branch):
- `PROD_S3_BUCKET_NAME`: aiot-drawio
- `PROD_CLOUDFRONT_DISTRIBUTION_ID`: E2FMKKXA4ZH70Y

**IMPORTANT:** Update the `AWS_ACCESS_KEY_ID` value in line 16 of the workflow file with your actual AWS access key ID.

### Required GitHub Secrets

You only need to configure these secrets in your GitHub repository:

1. Go to your repository on GitHub
2. Navigate to Settings > Secrets and variables > Actions
3. Add the following secrets:

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `AWS_SECRET_ACCESS_KEY` | AWS secret access key | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `ANTHROPIC_API_KEY` | Anthropic API key for Claude AI | `sk-ant-...` |

## IAM Permissions Required

The AWS user/access key needs the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateInvalidation"
      ],
      "Resource": "arn:aws:cloudfront::your-account-id:distribution/your-distribution-id"
    }
  ]
}
```

## Workflow Triggers

The deployment workflow runs:
- Automatically on push to the `release/dev` branch → deploys to **development** environment
- Automatically on push to the `release/production` branch → deploys to **production** environment
- Manually via GitHub Actions UI (workflow_dispatch) → uses the branch you select

## Deployment Flow

```
release/dev branch → Dev S3 Bucket → Dev CloudFront
release/production branch → Prod S3 Bucket → Prod CloudFront
```

## Build Configuration

The project is configured for static export (`output: 'export'` in `next.config.ts`), which generates static HTML/CSS/JS files suitable for S3 hosting.

## Cache Strategy

- Static assets (JS, CSS, images): cached for 1 year
- HTML, JSON, XML, TXT files: no cache, must revalidate

---

# 2. EKS Deployment (deploy-eks.yml)

Deploy full Next.js application with API routes to AWS EKS (Elastic Kubernetes Service).

## Configuration

### Workflow Structure

The workflow follows the same pattern as your existing EKS deployments:
- Branch-based environment selection
- Docker image build and push to ECR
- Kubernetes deployment update
- Automatic rollout restart

### Branches and Environments

| Branch | Environment |
|--------|-------------|
| `release/dev` | Development |
| `release/production` | Production |

### Required GitHub Secrets

**AWS Credentials:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

**Kubernetes Configuration (per environment):**

Development:
- `KUBE_DEV_HOST` - EKS API endpoint
- `KUBE_DEV_CERTIFICATE` - Base64 encoded CA cert

Production:
- `KUBE_PROD_HOST` - EKS API endpoint
- `KUBE_PROD_CERTIFICATE` - Base64 encoded CA cert

### ECR Repository

Create ECR repository before first deployment:

```bash
aws ecr create-repository \
  --repository-name aiot-drawio-nextjs \
  --region ap-southeast-1
```


## Deployment Flow

```
┌──────────────────────────────────────────┐
│  Push to release/dev or release/production │
└───────────────────┬──────────────────────┘
                    │
                    ▼
           ┌────────────────┐
           │ Checkout code  │
           └────────┬───────┘
                    │
                    ▼
      ┌─────────────────────────┐
      │ Build Docker image      │
      │ (Next.js standalone)    │
      └───────────┬─────────────┘
                  │
                  ▼
      ┌─────────────────────────┐
      │ Push to ECR with tags:  │
      │ - dev/production        │
      │ - <git-sha>             │
      │ - latest                │
      └───────────┬─────────────┘
                  │
                  ▼
      ┌─────────────────────────┐
      │ Get EKS token           │
      └───────────┬─────────────┘
                  │
                  ▼
      ┌─────────────────────────┐
      │ Update K8s deployment   │
      │ image (kubectl patch)   │
      └───────────┬─────────────┘
                  │
                  ▼
      ┌─────────────────────────┐
      │ Rollout restart         │
      │ deployment              │
      └─────────────────────────┘
```

## Docker Build

The Dockerfile uses multi-stage builds:
1. **deps** - Install dependencies
2. **builder** - Build Next.js app (standalone mode)
3. **runner** - Production image with minimal footprint

## Environment Variables

Environment variables are configured in Kubernetes secrets and deployment manifests:
- `NODE_ENV=production`
- `AI_PROVIDER=anthropic`
- `AI_MODEL=claude-sonnet-4-5-20250929`
- `ANTHROPIC_API_KEY` (from K8s secret)

## ECR Image Tags

Each build pushes three tags to ECR:
- **Environment tag** (`dev` or `production`) - Latest for that environment
- **Git SHA tag** (`abc1234`) - Specific version
- **Latest tag** (`latest`) - Most recent build

## Kubernetes Deployment Configuration

The workflow expects these Kubernetes resources to exist:
- **Namespace**: `aiot-drawio`
- **Deployment**: `{env}-aiot-drawio-deploy` (e.g., `dev-aiot-drawio-deploy`)

### Expected Deployment Structure

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-aiot-drawio-deploy  # or prod-aiot-drawio-deploy
  namespace: aiot-drawio
spec:
  replicas: 2
  selector:
    matchLabels:
      app: aiot-drawio
  template:
    spec:
      containers:
        - name: nextjs
          image: <ECR_REGISTRY>/aiot-drawio-nextjs:dev
          ports:
            - containerPort: 3000
```

The workflow will automatically update the image in `spec.template.spec.containers[0].image`

## Monitoring

**Build Status:**
- Check GitHub Actions tab for workflow runs

**Kubernetes Status:**
```bash
# Check deployment
kubectl get deployment -n aiot-drawio

# View pods
kubectl get pods -n aiot-drawio

# Check logs
kubectl logs -f deployment/dev-aiot-drawio-deploy -n aiot-drawio
```

## Cost Comparison

| Deployment Type | Monthly Cost (est.) | Pros | Cons |
|----------------|---------------------|------|------|
| S3 + CloudFront | $10-50 | Low cost, simple | No API routes |
| EKS | $150-500+ | Full features, scalable | Higher cost, complex |

---

# Additional Resources

- [Next.js Standalone Mode](https://nextjs.org/docs/advanced-features/output-file-tracing)
- [AWS ECR Documentation](https://docs.aws.amazon.com/ecr/)
- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)

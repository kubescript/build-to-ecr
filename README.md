# Build Docker Image to AWS ECR

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Build%20to%20ECR-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=github)](https://github.com/marketplace/actions/build-docker-image-to-aws-ecr)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A GitHub Action to build and push Docker images to Amazon Elastic Container Registry (ECR) using AWS OIDC authentication (no credentials needed).

## Features

- **Secure Authentication**: Uses AWS OIDC (OpenID Connect) - no need to store AWS credentials
- **BuildKit Caching**: Fast builds with GitHub Actions cache integration
- **Multi-Stage Builds**: Support for Docker multi-stage builds with target selection
- **Flexible Configuration**: Customizable build context, Dockerfile path, and build arguments
- **Smart Tagging**: Automatically tags images with `latest`, the specified tag, and major version (e.g., `v1`, `v2`)
- **Zero Setup**: Works out of the box with sensible defaults

## Automatic Tagging

This action automatically creates multiple tags for your Docker image:

1. **latest** - Always points to the most recent build
2. **Specified tag** - The exact tag you provide (e.g., `v1.2.3`, `sha-abc123`)
3. **Major version tag** - Automatically extracted from semver tags (e.g., `v1.2.3` → `v1`)

### Tag Examples

| Input Tag | Generated Tags |
|-----------|----------------|
| `v1.2.3` | `latest`, `v1.2.3`, `v1` |
| `v2.0.0` | `latest`, `v2.0.0`, `v2` |
| `1.5.2` | `latest`, `1.5.2`, `1` |
| `sha-abc123` | `latest`, `sha-abc123` (no major tag) |
| `main` | `latest`, `main` (no major tag) |

**Note**: Major version tags are only created for semver-compatible tags (e.g., `v1.2.3` or `1.2.3`). Non-semver tags will only get `latest` and the specified tag.

## Prerequisites

Before using this action, you need to configure:

### 1. AWS IAM Role with OIDC Trust Policy

Create an IAM role with a trust relationship for GitHub Actions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

Attach the following policy to the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2. ECR Repository

Create an ECR repository to store your Docker images:

```bash
aws ecr create-repository --repository-name my-app --region us-east-1
```

### 3. GitHub Actions Permissions

Your workflow must have these permissions:

```yaml
permissions:
  id-token: write   # Required for OIDC authentication
  contents: read    # Required to checkout code
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `aws-creds-role-to-assume` | IAM role ARN to assume via OIDC | Yes | - |
| `aws-region` | AWS region where ECR is located | Yes | - |
| `ecr-url` | ECR registry URL (e.g., `123456789.dkr.ecr.us-east-1.amazonaws.com`) | Yes | - |
| `ecr-repo` | ECR repository name | Yes | - |
| `docker-tag` | Docker image tag (e.g., `v1.0.0`, `sha-abc123`) | Yes | - |
| `docker-build-args` | Docker build arguments (multiline supported) | No | `""` |
| `docker-build-target` | Multi-stage build target stage name | No | `""` |
| `docker-build-context` | Build context directory | No | `.` |
| `docker-file-path` | Path to Dockerfile | No | `Dockerfile` |
| `cache-from` | BuildKit cache source configuration | No | `type=gha` |
| `cache-to` | BuildKit cache destination configuration | No | `type=gha,mode=max` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `image-digest` | Docker image digest (SHA256) | `sha256:abc123...` |
| `image-tag` | Full image tag pushed to ECR | `123.dkr.ecr.us-east-1.amazonaws.com/app:v1.0.0` |
| `major-tag` | Major version tag (if semver) | `v1` |

## Usage

### Basic Example

```yaml
name: Build and Push to ECR

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Build and push to ECR
        uses: kubescript/build-to-ecr@v1
        with:
          aws-creds-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          ecr-url: 123456789012.dkr.ecr.us-east-1.amazonaws.com
          ecr-repo: my-app
          docker-tag: ${{ github.sha }}
```

### Semantic Versioning with Auto Major Tags

When you push a semver tag, the action automatically creates major version tags:

```yaml
name: Build Release

on:
  push:
    tags:
      - 'v*'

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Extract version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Build and push to ECR
        id: build
        uses: kubescript/build-to-ecr@v1
        with:
          aws-creds-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          ecr-url: 123456789012.dkr.ecr.us-east-1.amazonaws.com
          ecr-repo: my-app
          docker-tag: ${{ steps.version.outputs.VERSION }}

      - name: Show created tags
        run: |
          echo "Image pushed with tags:"
          echo "- latest"
          echo "- ${{ steps.version.outputs.VERSION }}"
          echo "- ${{ steps.build.outputs.major-tag }}"
```

**Example**: Pushing tag `v1.2.3` will create:
- `my-app:latest`
- `my-app:v1.2.3`
- `my-app:v1`

### Advanced Example with Build Args

```yaml
name: Build with Build Arguments

on:
  push:
    tags:
      - 'v*'

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Extract version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Build and push to ECR
        uses: kubescript/build-to-ecr@v1
        with:
          aws-creds-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          ecr-url: 123456789012.dkr.ecr.us-east-1.amazonaws.com
          ecr-repo: my-app
          docker-tag: ${{ steps.version.outputs.VERSION }}
          docker-build-args: |
            VERSION=${{ steps.version.outputs.VERSION }}
            BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
            COMMIT_SHA=${{ github.sha }}
```

### Multi-Stage Build Example

```yaml
name: Multi-Stage Build

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build production image
        uses: kubescript/build-to-ecr@v1
        with:
          aws-creds-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          ecr-url: 123456789012.dkr.ecr.us-east-1.amazonaws.com
          ecr-repo: my-app
          docker-tag: ${{ github.sha }}
          docker-build-target: production
          docker-file-path: ./docker/Dockerfile
          docker-build-context: .
```

### Monorepo Example

```yaml
name: Build Microservice

on:
  push:
    paths:
      - 'services/api/**'

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build API service
        uses: kubescript/build-to-ecr@v1
        with:
          aws-creds-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          ecr-url: 123456789012.dkr.ecr.us-east-1.amazonaws.com
          ecr-repo: api-service
          docker-tag: ${{ github.sha }}
          docker-build-context: ./services/api
          docker-file-path: ./services/api/Dockerfile
```

### Using Outputs

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-digest: ${{ steps.build.outputs.image-digest }}
      image-tag: ${{ steps.build.outputs.image-tag }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build and push to ECR
        id: build
        uses: kubescript/build-to-ecr@v1
        with:
          aws-creds-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          ecr-url: 123456789012.dkr.ecr.us-east-1.amazonaws.com
          ecr-repo: my-app
          docker-tag: ${{ github.sha }}

      - name: Print image info
        run: |
          echo "Image Digest: ${{ steps.build.outputs.image-digest }}"
          echo "Image Tag: ${{ steps.build.outputs.image-tag }}"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy image
        run: |
          echo "Deploying image: ${{ needs.build.outputs.image-tag }}"
          echo "With digest: ${{ needs.build.outputs.image-digest }}"
```

## Troubleshooting

### Error: "No basic auth credentials"

**Problem**: Authentication to ECR failed.

**Solution**: Verify your IAM role has the correct trust policy and ECR permissions. Ensure the `ecr-url` matches your AWS account and region.

### Error: "could not find a Dockerfile"

**Problem**: Dockerfile not found at the specified path.

**Solution**: Check that `docker-file-path` is correct relative to `docker-build-context`. For example:
- Context: `./services/api`
- Dockerfile: `./services/api/Dockerfile`
- Set `docker-file-path: Dockerfile` (relative to context)

### Slow builds

**Problem**: Builds are taking too long.

**Solution**:
- Ensure cache is working: check that `cache-from` and `cache-to` are configured
- Optimize your Dockerfile layer ordering (put frequently changing layers last)
- Use multi-stage builds to reduce final image size

### Error: "Permission denied" for OIDC

**Problem**: Cannot assume IAM role via OIDC.

**Solution**:
1. Verify OIDC provider is configured in AWS IAM
2. Check the trust policy condition matches your repository
3. Ensure workflow has `id-token: write` permission

### Build args not working

**Problem**: Build arguments are not being passed to Docker.

**Solution**: Use multiline format:
```yaml
docker-build-args: |
  ARG1=value1
  ARG2=value2
```

## Acknowledgments

This action uses the following third-party GitHub Actions:

- **[docker/setup-buildx-action](https://github.com/docker/setup-buildx-action)** - Set up Docker Buildx
- **[aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)** - Configure AWS credentials using OIDC
- **[docker/login-action](https://github.com/docker/login-action)** - Login to container registries
- **[docker/build-push-action](https://github.com/docker/build-push-action)** - Build and push Docker images

Thank you to all the maintainers and contributors of these projects!

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Support

- Report issues: [GitHub Issues](https://github.com/kubescript/build-to-ecr/issues)
- Documentation: [GitHub Marketplace](https://github.com/marketplace/actions/build-docker-image-to-aws-ecr)

---

Made with ❤️ by [KubeScript](https://github.com/kubescript)

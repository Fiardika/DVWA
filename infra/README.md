# Security Scan Pipeline using AWS Native

Pipeline security scanning menggunakan native AWS services untuk Tech Talks UG Malang.

## Architecture

```
Source (GitHub) → SAST (CodeGuru) → Build (Docker/ECR) → Container Scan (Inspector)
```

## Files

| File | Deskripsi |
|------|-----------|
| `buildspec.yml` | Build & push Docker image ke ECR |
| `buildspec-sast.yml` | SAST scan dengan CodeGuru Security |
| `buildspec-inspector.yml` | Container scan dengan Amazon Inspector |
| `pipeline.json` | Template CodePipeline (4 stages) |

## AWS Services yang Digunakan

1. **CodePipeline** - Orchestrator pipeline
2. **CodeBuild** - Build Docker image + jalankan scan
3. **CodeGuru Security** - Static Application Security Testing (SAST)
4. **Amazon Inspector** - Container vulnerability scanning
5. **ECR** - Container registry

## Setup Manual

### 1. Buat ECR Repository

```bash
aws ecr create-repository --repository-name demo-ug-malang --region ap-southeast-1
```

### 2. Enable Amazon Inspector

```bash
aws inspector2 enable --resource-types ECR
```

### 3. Buat S3 Bucket untuk Artifacts

```bash
aws s3 mb s3://security-scan-pipeline-artifacts-<ACCOUNT_ID> --region ap-southeast-1
```

### 4. Buat CodeBuild Projects

Buat 3 CodeBuild project di console:

| Project Name | Buildspec |
|--------------|-----------|
| `dvwa-sast-scan` | `infra/buildspec-sast.yml` |
| `dvwa-build` | `infra/buildspec.yml` |
| `dvwa-inspector-scan` | `infra/buildspec-inspector.yml` |

**Environment:**
- Managed image: `aws/codebuild/amazonlinux2-x86_64-standard:5.0`
- Privileged: ✅ (untuk Docker)
- Environment variable: `AWS_ACCOUNT_ID` = Account ID kamu

### 5. Buat CodePipeline

Update placeholder di `pipeline.json`:
- `<PIPELINE_ROLE_ARN>` - IAM Role untuk CodePipeline
- `<ARTIFACT_BUCKET_NAME>` - S3 bucket artifacts
- `<CODESTAR_CONNECTION_ARN>` - Connection ke GitHub
- `<GITHUB_OWNER/REPO_NAME>` - Repository GitHub

## IAM Permissions

### CodeBuild Role

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
    },
    {
      "Effect": "Allow",
      "Action": [
        "inspector2:ListFindings"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codeguru-security:CreateUploadUrl",
        "codeguru-security:CreateScan",
        "codeguru-security:GetScan",
        "codeguru-security:GetFindings"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::security-scan-pipeline-artifacts-*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

## Demo Flow

1. Push code ke GitHub → trigger pipeline
2. **Stage SAST**: CodeGuru scan source code → lihat findings di console
3. **Stage Build**: Build Docker image → push ke ECR
4. **Stage Container Scan**: Inspector scan image → lihat vulnerabilities

## Output yang Bisa Ditunjukkan

- CodeGuru Security findings (SQL Injection, XSS, dll)
- Inspector findings (CVE di base image, dependencies)
- Pipeline execution history

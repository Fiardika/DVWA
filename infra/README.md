# Security Scan Pipeline using AWS Native

Pipeline security scanning menggunakan native AWS services untuk Tech Talks UG Malang.

**Fokus**: Security scanning saja, tidak sampai deployment.

## Architecture

```
Source (GitHub) → SAST (Inspector SBOM) → Build (Docker/ECR) → Container Scan (Inspector)
```

## Files

| File | Deskripsi |
|------|-----------|
| `buildspec.yml` | Build & push Docker image ke ECR |
| `buildspec-sast.yml` | SAST scan dengan Amazon Inspector SBOM Generator |
| `buildspec-inspector.yml` | Container scan dengan Amazon Inspector |
| `pipeline.json` | Template CodePipeline (4 stages) |

## AWS Services yang Digunakan

1. **CodePipeline** - Orchestrator pipeline
2. **CodeBuild** - Build Docker image + jalankan scan
3. **Amazon Inspector** - SAST (via SBOM Generator) + Container vulnerability scanning
4. **ECR** - Container registry
5. **S3** - Artifact storage

## Setup Manual

### 1. Buat ECR Repository

```bash
aws ecr create-repository \
  --repository-name demo-ug-malang \
  --region ap-southeast-1
```

### 2. Enable Amazon Inspector

```bash
# Enable Inspector untuk ECR scanning
aws inspector2 enable \
  --resource-types ECR \
  --region ap-southeast-1
```

Inspector akan otomatis scan setiap image yang di-push ke ECR.

**Note tentang Inspector SBOM Generator:**
- Tool CLI `inspector-sbomgen` di-download langsung di buildspec
- Tidak perlu enable service tambahan
- Menggunakan `inspector-scan:ScanSbom` API untuk analisis vulnerabilities

### 3. Buat S3 Bucket untuk Artifacts

```bash
aws s3 mb s3://aws-ug-demo-artifacts \
  --region ap-southeast-1
```

Bucket ini untuk simpan artifacts antar stage di pipeline.

### 4. Buat IAM Roles

#### a. CodeBuild Service Role

Buat role dengan trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codebuild.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Attach policy berikut (buat inline policy):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAccess",
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
      "Sid": "InspectorAccess",
      "Effect": "Allow",
      "Action": [
        "inspector2:ListFindings"
      ],
      "Resource": "*"
    },
    {
      "Sid": "InspectorSBOMAccess",
      "Effect": "Allow",
      "Action": [
        "inspector-scan:ScanSbom"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::aws-ug-demo-artifacts/*"
    },
    {
      "Sid": "CloudWatchLogs",
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

**Penjelasan permissions:**
- `ECRAccess` - Login, push/pull Docker images
- `InspectorAccess` - Baca hasil scan vulnerabilities dari ECR image
- `InspectorSBOMAccess` - Scan SBOM untuk SAST (source code analysis)
- `S3Access` - Simpan/ambil artifacts
- `CloudWatchLogs` - Kirim logs build

#### b. CodePipeline Service Role

Buat role dengan trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codepipeline.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Attach policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CodeBuildAccess",
      "Effect": "Allow",
      "Action": [
        "codebuild:StartBuild",
        "codebuild:BatchGetBuilds"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::aws-ug-demo-artifacts",
        "arn:aws:s3:::aws-ug-demo-artifacts/*"
      ]
    },
    {
      "Sid": "CodeStarConnection",
      "Effect": "Allow",
      "Action": [
        "codestar-connections:UseConnection"
      ],
      "Resource": "*"
    }
  ]
}
```

**Penjelasan permissions:**
- `CodeBuildAccess` - Trigger CodeBuild projects
- `S3Access` - Simpan/ambil artifacts antar stage
- `CodeStarConnection` - Akses GitHub via CodeStar Connection

### 5. Buat CodeStar Connection ke GitHub

```bash
# Buat connection (akan pending sampai di-authorize)
aws codestar-connections create-connection \
  --provider-type GitHub \
  --connection-name github-connection \
  --region ap-southeast-1
```

Setelah itu:
1. Buka AWS Console → Developer Tools → Connections
2. Klik connection yang baru dibuat
3. Klik "Update pending connection"
4. Authorize akses ke GitHub
5. Copy ARN connection-nya

### 6. Buat CodeBuild Projects

Buat 3 CodeBuild project via console atau CLI:

#### Project 1: SAST Scan (Inspector SBOM)

```bash
aws codebuild create-project \
  --name dvwa-sast-scan \
  --source type=CODEPIPELINE \
  --artifacts type=CODEPIPELINE \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:5.0,computeType=BUILD_GENERAL1_SMALL,privilegedMode=false \
  --service-role <CODEBUILD_ROLE_ARN> \
  --region ap-southeast-1
```

Buildspec: `infra/buildspec-sast.yml`

#### Project 2: Docker Build

```bash
aws codebuild create-project \
  --name dvwa-build \
  --source type=CODEPIPELINE \
  --artifacts type=CODEPIPELINE \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:5.0,computeType=BUILD_GENERAL1_SMALL,privilegedMode=true,environmentVariables='[{"name":"AWS_ACCOUNT_ID","value":"<ACCOUNT_ID>"}]' \
  --service-role <CODEBUILD_ROLE_ARN> \
  --region ap-southeast-1
```

Buildspec: `infra/buildspec.yml`

#### Project 3: Inspector Container Scan

```bash
aws codebuild create-project \
  --name dvwa-inspector-scan \
  --source type=CODEPIPELINE \
  --artifacts type=CODEPIPELINE \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:5.0,computeType=BUILD_GENERAL1_SMALL,privilegedMode=true,environmentVariables='[{"name":"AWS_ACCOUNT_ID","value":"<ACCOUNT_ID>"}]' \
  --service-role <CODEBUILD_ROLE_ARN> \
  --region ap-southeast-1
```

Buildspec: `infra/buildspec-inspector.yml`

**Penjelasan Environment Settings:**

- **Managed image**: `aws/codebuild/amazonlinux2-x86_64-standard:5.0`
  - Image Docker yang sudah include tools: Docker, AWS CLI, git, dll
  - Versi 5.0 adalah yang terbaru dan stable
  
- **Privileged mode**: `true` (untuk project build & inspector)
  - Diperlukan untuk build Docker image (Docker-in-Docker)
  - SAST scan tidak perlu karena hanya scan source code
  
- **Environment variable**: `AWS_ACCOUNT_ID`
  - Dipakai di buildspec untuk construct ECR URI
  - Format: `<ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/demo-ug-malang`

- **Compute type**: `BUILD_GENERAL1_SMALL`
  - 3 GB memory, 2 vCPUs
  - Cukup untuk demo, bisa upgrade kalau perlu

### 7. Buat CodePipeline via Console

1. **Buka AWS Console** → CodePipeline → Create pipeline

2. **Pipeline settings**
   - Pipeline name: `security-scan-pipeline`
   - Service role: Pilih role yang dibuat di step 4b
   - Artifact store: Custom location
   - Bucket: `aws-ug-demo-artifacts`
   - Klik **Next**

3. **Add source stage**
   - Source provider: **GitHub (Version 2)**
   - Connection: Pilih connection yang dibuat di step 5
     - Kalau belum ada, klik "Connect to GitHub" dan authorize
   - Repository name: Pilih repo DVWA kamu
   - Branch name: `main`
   - Output artifact format: **CodePipeline default**
   - Klik **Next**

4. **Add build stage - SAST**
   - Build provider: **AWS CodeBuild**
   - Region: `ap-southeast-1`
   - Project name: `dvwa-sast-scan`
   - Build type: **Single build**
   - Klik **Next**

5. **Skip deploy stage**
   - Klik **Skip deploy stage**
   - Confirm skip

6. **Review**
   - Review semua settings
   - Klik **Create pipeline**

7. **Tambah stage Build & Container Scan**
   
   Setelah pipeline dibuat, edit untuk tambah 2 stage lagi:
   
   a. Klik **Edit** di pipeline
   
   b. Klik **Add stage** setelah stage "Build" (SAST)
      - Stage name: `DockerBuild`
      - Klik **Add stage**
   
   c. Di stage "DockerBuild", klik **Add action group**
      - Action name: `Docker-Build`
      - Action provider: **AWS CodeBuild**
      - Region: `ap-southeast-1`
      - Input artifacts: `SourceArtifact`
      - Project name: `dvwa-build`
      - Output artifacts: `BuildArtifact`
      - Klik **Done**
   
   d. Klik **Add stage** setelah "DockerBuild"
      - Stage name: `ContainerScan`
      - Klik **Add stage**
   
   e. Di stage "ContainerScan", klik **Add action group**
      - Action name: `Inspector-Scan`
      - Action provider: **AWS CodeBuild**
      - Region: `ap-southeast-1`
      - Input artifacts: `SourceArtifact`
      - Project name: `dvwa-inspector-scan`
      - Output artifacts: `InspectorArtifact`
      - Klik **Done**
   
   f. Klik **Save** di atas
   
   g. Confirm save

**Final pipeline structure:**
```
Source (GitHub) → SAST (Inspector SBOM) → DockerBuild → ContainerScan (Inspector)
```

## Demo Flow

1. **Push code** ke GitHub → otomatis trigger pipeline
2. **Stage 1 - Source**: Ambil code dari GitHub
3. **Stage 2 - SAST**: Inspector SBOM scan source code
   - Generate SBOM dari dependencies
   - Deteksi: Vulnerable packages, outdated libraries
   - Output: `sbom.json`, `inspector-sast-findings.json`
4. **Stage 3 - Build**: Build Docker image → push ke ECR
   - Image otomatis di-scan oleh Inspector
5. **Stage 4 - Container Scan**: Ambil hasil scan Inspector
   - Deteksi: CVE di base image, vulnerable packages
   - Lihat findings di Inspector console

## Output untuk Demo

**Yang bisa ditunjukkan:**

1. **CodePipeline Console**
   - Visual flow 4 stages
   - Execution history
   - Duration tiap stage

2. **CodeBuild Logs - SAST Stage**
   - SBOM generation output
   - Vulnerability findings dari Inspector Scan API
   - Severity breakdown: Critical, High, Medium, Low

3. **Amazon Inspector Console**
   - CVE findings di container image
   - CVSS scores
   - Affected packages & fix available

4. **Artifacts**
   - `sbom.json` - Software Bill of Materials
   - `inspector-sast-findings.json` - SAST scan results
   - `inspector-findings.json` - Container scan results

## Perbedaan SAST vs Container Scan

| Aspek | SAST (Inspector SBOM) | Container Scan |
|-------|----------------------|----------------|
| **Target** | Source code dependencies | Container image |
| **Timing** | Sebelum build | Setelah push ke ECR |
| **Deteksi** | Vulnerable libraries di code | CVE di OS packages + app |
| **Tool** | inspector-sbomgen + ScanSbom API | Inspector auto-scan ECR |

## Tips Demo

- Jalankan pipeline dari awal untuk show full flow
- Highlight perbedaan findings antara SAST vs Container scan
- Tunjukkan SBOM output untuk explain software composition
- Show severity levels & affected packages
- Explain kenapa DVWA vulnerable by design

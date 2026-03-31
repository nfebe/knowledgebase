# Packer Setup for runs-on in ubinfra

## Overview

The `ubinfra` repository uses **HashiCorp Packer** to build custom Amazon Machine Images (AMIs) for GitHub Actions runners via the **runs-on** service. These pre-built AMIs include all necessary development tools, significantly reducing CI build times.

**Location**: `ubinfra/utils/runs-on-ami/`

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AMI Build Pipeline                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GitHub Actions (rebuild-ami.yml)                               │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐    ┌─────────────────┐    ┌───────────────┐   │
│  │ Packer Init │───▶│ Packer Build    │───▶│ Custom AMI    │   │
│  └─────────────┘    │                 │    │ (ub-runner-*) │   │
│                     │ - Launches EC2  │    └───────────────┘   │
│                     │ - Runs scripts  │            │           │
│                     │ - Creates AMI   │            ▼           │
│                     └─────────────────┘    ┌───────────────┐   │
│                                            │ runs-on uses  │   │
│                                            │ AMI for jobs  │   │
│                                            └───────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Key Files

| File | Purpose |
|------|---------|
| `ub-runner.pkr.hcl` | Linux AMI Packer configuration |
| `ub-runner-windows.pkr.hcl` | Windows AMI Packer configuration |
| `scripts/provision.sh` | Linux provisioning script |
| `scripts/provision-windows.ps1` | Windows provisioning script |
| `scripts/user_data.sh` | Linux EC2 user data |
| `scripts/user_data_windows.ps1` | Windows EC2 user data (enables WinRM) |

## Linux AMI Configuration

### Base Image
- **Source**: runs-on official AMI (`runs-on-v2.2-ubuntu22-full-x64-*`)
- **Owner**: `135269210855` (runs-on AWS account)
- **Build Instance**: `t3.medium`

### Pre-installed Tools
| Category | Tools |
|----------|-------|
| Python | 3.10, 3.11, 3.12, 3.13 + rye |
| Node.js | 20 LTS |
| Java | OpenJDK 18 |
| Rust | Latest (via rustup) |
| Docker | Docker CE + Compose plugin |
| AWS | AWS CLI v2 |
| Browsers | Playwright (Chromium, Firefox, WebKit) |
| Build | gcc, make, patchelf, graphviz |

### Build Command
```bash
cd utils/runs-on-ami
packer init ub-runner.pkr.hcl
packer build \
  -var "subnet_id=subnet-xxxxxxxx" \
  -var "region=eu-central-1" \
  ub-runner.pkr.hcl
```

## Windows AMI Configuration

### Base Image
- **Source**: Amazon Windows Server 2022 (`Windows_Server-2022-English-Full-Base-*`)
- **Owner**: `amazon`
- **Build Instance**: `t3.large`
- **Communication**: WinRM (not SSH)

### Pre-installed Tools
| Category | Tools |
|----------|-------|
| Python | 3.10, 3.11, 3.12, 3.13 (via Chocolatey) |
| Node.js | 20 LTS |
| Java | OpenJDK 18 |
| Rust | Latest |
| Docker | Docker CLI |
| AWS | AWS CLI |
| Browsers | Playwright |
| Build | Visual Studio 2022 Build Tools, MSVC |
| Package Manager | Chocolatey, rye |

### Build Command
```bash
cd utils/runs-on-ami
packer init ub-runner-windows.pkr.hcl
packer build \
  -var "subnet_id=subnet-xxxxxxxx" \
  -var "region=eu-central-1" \
  ub-runner-windows.pkr.hcl
```

## Using runs-on in Workflows

### Step 1: Add runs-on.yml to your repo

Create `.github/runs-on.yml`:

```yaml
images:
  ub-runner:
    platform: linux
    arch: x64
    owner: "595319716151"
    name: "ub-runner-linux-*"
  ub-runner-windows:
    platform: windows
    arch: x64
    owner: "595319716151"
    name: "ub-runner-windows-*"
```

### Step 2: Reference in workflows

**Linux (custom AMI):**
```yaml
jobs:
  build:
    runs-on: runs-on=${{ github.run_id }}/runner=2cpu-linux-x64/image=ub-runner
    steps:
      - uses: actions/checkout@v4
```

**Linux with S3 caching:**
```yaml
jobs:
  build:
    runs-on: runs-on=${{ github.run_id }}/runner=2cpu-linux-x64/image=ub-runner/extras=s3-cache
    steps:
      - uses: runs-on/action@v2
      - uses: actions/checkout@v4
```

**Windows (custom AMI):**
```yaml
jobs:
  build:
    runs-on: runs-on=${{ github.run_id }}/image=ub-runner-windows/family=m7i
```

## Runner Sizes and Costs

### Linux
| Size | Instance | Cost/min | Use Case |
|------|----------|----------|----------|
| 1cpu | m7a.medium | $0.0008 | Linting, type checking, docs |
| 2cpu | m7i.large | $0.0011 | Tests, builds |
| 4cpu | m7i.xlarge | $0.0019 | Parallel workloads |

### Windows
| Size | Instance | Cost/min | Use Case |
|------|----------|----------|----------|
| 2cpu | m7i.large | ~$0.002 | Standard builds |
| 4cpu | m7i.xlarge | ~$0.004 | Parallel workloads |

### Boot Times
| Image | Boot Time | Notes |
|-------|-----------|-------|
| ub-runner (Linux) | ~30s | Fast boot with pre-installed tools |
| ub-runner-windows | 2-3 min | Custom AMI with pre-installed tools |
| windows22-full-x64 | 2-3 min | runs-on provided image |
| windows22-base-x64 | ~1 min | Minimal software |

## Automated Maintenance

### Scheduled Rebuilds
AMIs are automatically rebuilt via GitHub Actions:
- **Schedule**: 1st and 15th of each month at 2am UTC
- **Workflow**: `.github/workflows/rebuild-ami.yml`
- **Cleanup**: Keeps last 3 AMIs, deregisters older ones

### Manual Rebuild
Trigger via workflow dispatch:
```
GitHub UI → Actions → Rebuild runs-on AMI → Run workflow
  - region: eu-central-1 (default)
  - platform: linux | windows | both
```

### Why Rebuild?
GitHub Actions stops sending jobs to runners with agents older than 30 days. Regular rebuilds ensure compatibility.

## AWS Permissions Required

To build AMIs, the AWS credentials need:
- `ec2:RunInstances`, `ec2:TerminateInstances`
- `ec2:CreateImage`, `ec2:DescribeImages`, `ec2:DeregisterImage`
- `ec2:DescribeSubnets`, `ec2:CreateTags`
- `ec2:DescribeSecurityGroups`, `ec2:CreateSecurityGroup`, `ec2:DeleteSecurityGroup`
- `ec2:AuthorizeSecurityGroupIngress`, `ec2:DescribeKeyPairs`

## Troubleshooting

### Common Issues

**Packer build timeout:**
- Windows builds can take 30-45 minutes due to VS Build Tools
- Increase timeout in workflow if needed

**WinRM connection fails (Windows):**
- Check security group allows inbound WinRM (5985/5986)
- Verify user_data_windows.ps1 executed properly

**AMI not found in runs-on:**
- Verify AMI owner ID matches in `.github/runs-on.yml`
- Check AMI is in the correct region

**Playwright install fails:**
- May need to update Playwright version in provision scripts
- Check for dependency changes in Playwright releases

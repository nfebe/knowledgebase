# Packer / runs-on Quick Reference

## Build Commands

```bash
# Linux AMI
cd ubinfra/utils/runs-on-ami
packer init ub-runner.pkr.hcl
packer build -var "subnet_id=subnet-xxx" -var "region=eu-central-1" ub-runner.pkr.hcl

# Windows AMI
packer init ub-runner-windows.pkr.hcl
packer build -var "subnet_id=subnet-xxx" -var "region=eu-central-1" ub-runner-windows.pkr.hcl
```

## Workflow runs-on Strings

```yaml
# Linux - choose runner size
runs-on: runs-on=${{ github.run_id }}/runner=1cpu-linux-x64/image=ub-runner  # Lint/docs
runs-on: runs-on=${{ github.run_id }}/runner=2cpu-linux-x64/image=ub-runner  # Tests
runs-on: runs-on=${{ github.run_id }}/runner=4cpu-linux-x64/image=ub-runner  # Heavy

# Linux with S3 cache
runs-on: runs-on=${{ github.run_id }}/runner=2cpu-linux-x64/image=ub-runner/extras=s3-cache

# Windows
runs-on: runs-on=${{ github.run_id }}/image=ub-runner-windows/family=m7i
```

## runs-on.yml Template

```yaml
# .github/runs-on.yml
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

## Source Files Reference

| Description | Path |
|-------------|------|
| Linux Packer config | `ubinfra/utils/runs-on-ami/ub-runner.pkr.hcl` |
| Windows Packer config | `ubinfra/utils/runs-on-ami/ub-runner-windows.pkr.hcl` |
| Linux provision script | `ubinfra/utils/runs-on-ami/scripts/provision.sh` |
| Windows provision script | `ubinfra/utils/runs-on-ami/scripts/provision-windows.ps1` |
| Rebuild workflow | `ubinfra/.github/workflows/rebuild-ami.yml` |
| Original README | `ubinfra/utils/runs-on-ami/README.md` |

## Cost Quick Reference

| Runner | Cost/min | Monthly (8hr/day) |
|--------|----------|-------------------|
| 1cpu Linux | $0.0008 | ~$10 |
| 2cpu Linux | $0.0011 | ~$13 |
| 4cpu Linux | $0.0019 | ~$23 |
| 2cpu Windows | $0.002 | ~$24 |
| 4cpu Windows | $0.004 | ~$48 |

## Adding New Tools to AMI

1. Edit the appropriate provision script:
   - Linux: `scripts/provision.sh`
   - Windows: `scripts/provision-windows.ps1`

2. Trigger rebuild via GitHub Actions or run locally

3. Wait for AMI to propagate (~5-10 min after build completes)

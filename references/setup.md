# Installation & Configuration

## Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [CLI v2 Features](#cli-v2-features)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Installation

AWS CLI v2 is a self-contained bundle with its own Python runtime, eliminating dependency conflicts.

### Windows

**MSI Installer (Recommended):**
```powershell
# Download and install (admin required)
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Silent install
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi /qn

# Update existing installation
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi /qn REINSTALL=ALL REINSTALLMODE=omus
```

**Winget:**
```powershell
winget install -e --id Amazon.AWSCLI
winget upgrade Amazon.AWSCLI
```

**Chocolatey:**
```powershell
choco install awscli
choco upgrade awscli
```

### macOS

**PKG Installer (All Users):**
```bash
# Download and install
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Verify
which aws
aws --version
```

**Homebrew:**
```bash
brew install awscli
brew upgrade awscli
```

### Linux

**Bundled Installer (Recommended):**
```bash
# x86_64
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# ARM64 (Graviton)
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Update existing installation
sudo ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update

# Custom install location (no sudo)
./aws/install --install-dir ~/aws-cli --bin-dir ~/bin
```

**Snap (Auto-updating):**
```bash
sudo snap install aws-cli --classic
```

**Docker:**
```bash
docker run --rm -it amazon/aws-cli --version
docker run --rm -it -v ~/.aws:/root/.aws amazon/aws-cli s3 ls
```

### Verify Installation
```bash
aws --version
# aws-cli/2.x.x Python/3.x.x Linux/x86_64 source/x86_64.amzn.2
```

## Configuration

### Interactive Setup

```bash
# Basic configuration (access keys)
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-west-2
# Default output format [None]: json

# Named profile
aws configure --profile production

# IAM Identity Center (SSO) - Recommended for organizations
aws configure sso
# SSO session name: my-sso
# SSO start URL: https://my-sso-portal.awsapps.com/start
# SSO region: us-east-1
# SSO registration scopes: sso:account:access

# View current configuration
aws configure list
aws configure list --profile production
```

### Configuration Files

**~/.aws/credentials** (access keys only):
```ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[production]
aws_access_key_id = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
```

**~/.aws/config** (settings and role configurations):
```ini
[default]
region = us-west-2
output = json
cli_pager =

[profile production]
region = us-east-1
output = text

[profile cross-account]
role_arn = arn:aws:iam::123456789012:role/AdminRole
source_profile = default
role_session_name = MySession
duration_seconds = 3600

[profile mfa-protected]
role_arn = arn:aws:iam::123456789012:role/MFARole
source_profile = default
mfa_serial = arn:aws:iam::111111111111:mfa/myuser

[profile sso-dev]
sso_session = my-sso
sso_account_id = 123456789012
sso_role_name = PowerUserAccess
region = us-east-1

[sso-session my-sso]
sso_start_url = https://my-sso-portal.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access
```

### Environment Variables

| Variable | Description | Example |
|:---------|:------------|:--------|
| `AWS_ACCESS_KEY_ID` | Access key | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | Secret key | `wJalrXUtnFEMI/...` |
| `AWS_SESSION_TOKEN` | Session token (temp creds) | From STS |
| `AWS_PROFILE` | Named profile to use | `production` |
| `AWS_REGION` | AWS region | `us-west-2` |
| `AWS_DEFAULT_REGION` | Fallback region | `us-east-1` |
| `AWS_DEFAULT_OUTPUT` | Output format | `json`, `text`, `table` |
| `AWS_PAGER` | Pager program | `""` (disable) |
| `AWS_CONFIG_FILE` | Config file path | `~/.aws/config` |
| `AWS_SHARED_CREDENTIALS_FILE` | Credentials path | `~/.aws/credentials` |
| `AWS_CA_BUNDLE` | CA certificate bundle | `/path/to/cert.pem` |
| `AWS_RETRY_MODE` | Retry behavior | `standard`, `adaptive` |
| `AWS_MAX_ATTEMPTS` | Max retry attempts | `5` |

### Credential Precedence

1. Command-line options (`--profile`)
2. Environment variables (`AWS_ACCESS_KEY_ID`, etc.)
3. Web identity token file (EKS IRSA)
4. SSO credentials (`sso_*` config)
5. Credentials file (`~/.aws/credentials`)
6. Config file credential_process
7. Container credentials (ECS task role)
8. Instance metadata (EC2 instance profile)

## CLI v2 Features

### Auto-Prompt Mode
Interactive parameter completion for discovering commands:
```bash
# Enable for single command
aws dynamodb --cli-auto-prompt

# Enable globally
aws configure set cli_auto_prompt on

# Disable paging during auto-prompt
aws configure set cli_pager ""
```

### Output Formats
```bash
aws ec2 describe-instances --output json    # Default, machine-parseable
aws ec2 describe-instances --output text    # Tab-separated (grep/awk)
aws ec2 describe-instances --output table   # Human-readable ASCII
aws ec2 describe-instances --output yaml    # YAML (v2 only)
aws ec2 describe-instances --output yaml-stream  # Streaming YAML
```

### SSO Integration
```bash
# Configure SSO
aws configure sso

# Login (opens browser)
aws sso login --profile sso-dev

# Logout
aws sso logout

# List available accounts/roles
aws sso list-accounts
aws sso list-account-roles --account-id 123456789012
```

### Wizards
Interactive configuration for complex setups:
```bash
aws configure wizard    # General configuration wizard
```

## Troubleshooting

### Common Issues

| Issue | Symptom | Solution |
|:------|:--------|:---------|
| **Command not found** | `aws: command not found` | Add to PATH: `/usr/local/bin` (Linux/Mac), `C:\Program Files\Amazon\AWSCLIV2\` (Windows) |
| **Invalid credentials** | `InvalidClientTokenId` | Run `aws configure` or check `~/.aws/credentials` |
| **Expired SSO token** | `Token has expired` | Run `aws sso login --profile <name>` |
| **Clock skew** | `SignatureDoesNotMatch` | Sync system clock: `sudo ntpdate pool.ntp.org` |
| **SSL certificate error** | `SSL: CERTIFICATE_VERIFY_FAILED` | Set `AWS_CA_BUNDLE` or update CA certs |
| **Permission denied** | `AccessDenied` | Check IAM policy with `aws sts get-caller-identity` |
| **Region not set** | `You must specify a region` | Set `AWS_REGION` or use `--region` |
| **Profile not found** | `Profile not found` | Check `~/.aws/config` syntax, use `[profile name]` |
| **MFA required** | `AccessDenied` with MFA | Use `--serial-number` and `--token-code` with assume-role |
| **Throttling** | `Throttling` or `Rate exceeded` | Add retry logic, use exponential backoff |

### Debugging
```bash
# Enable debug output
aws s3 ls --debug

# Debug specific request
aws s3 ls --debug 2>&1 | grep -i "x-amz"

# Check configured values
aws configure list

# Verify identity
aws sts get-caller-identity

# Test specific profile
aws sts get-caller-identity --profile production
```

### Credential Troubleshooting
```bash
# Check which credentials are being used
aws configure list
# Shows: profile, access_key, secret_key, region (and source)

# Verify identity
aws sts get-caller-identity
# Returns: Account, UserId, Arn

# Clear cached SSO credentials
rm -rf ~/.aws/sso/cache/*

# Check environment overrides
env | grep AWS_
```

### Network Issues
```bash
# Test connectivity
aws s3 ls --debug 2>&1 | grep -i "establishing"

# Use proxy
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080

# Disable SSL verification (not recommended for production)
aws s3 ls --no-verify-ssl

# Custom CA bundle
export AWS_CA_BUNDLE=/path/to/ca-bundle.crt
```

### Version Conflicts (v1 vs v2)
```bash
# Check installed version
aws --version

# Check path
which aws
type aws

# Remove old v1 (pip-installed)
pip uninstall awscli

# Ensure v2 is in PATH first
export PATH="/usr/local/bin:$PATH"
```

## Best Practices

| Practice | Description |
|:---------|:------------|
| **Use SSO** | Prefer `aws configure sso` over long-lived access keys |
| **Named profiles** | Use profiles for different accounts/roles |
| **Disable pager** | Set `AWS_PAGER=""` in scripts |
| **Explicit regions** | Always specify region in automation |
| **MFA for sensitive ops** | Configure MFA on roles for destructive actions |
| **Credential rotation** | Rotate access keys every 90 days |
| **Use roles** | Prefer instance profiles/task roles over embedded keys |

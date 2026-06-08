# Skandha Deploy — TAMS Tools on EC2 Ubuntu

Deploy guide for `time-addressable-media-store-tools` on an EC2 Ubuntu instance.

**Assumes**: The [time-addressable-media-store](https://github.com/awslabs/time-addressable-media-store) (TAMS API) is already deployed and its CloudFormation stack is running. You need the TAMS API stack name for this deployment.

## Architecture

- **Backend**: Python 3.14 Lambda functions, Step Functions, SQS, EventBridge — deployed to AWS via SAM (serverless, not running on EC2)
- **Frontend**: React 19 + TypeScript SPA built with Vite — served via Nginx on EC2
- **Auth**: Uses Cognito from the already-deployed TAMS API stack (default) or external OIDC
- **EC2 Role**: Build machine for SAM + web server for the frontend

---

## 1. EC2 Instance Requirements

| Spec | Recommendation |
|------|---------------|
| AMI | Ubuntu 22.04 or 24.04 LTS |
| Instance Type | t3.medium (minimum) — SAM builds need memory |
| Storage | 30 GB gp3 (minimum) |
| Security Group | Inbound: SSH (22), HTTP (80), HTTPS (443) |
| IAM Role | Instance profile with permissions for CloudFormation, Lambda, S3, IAM, EventBridge, SQS, SSM, Cognito, Secrets Manager, Step Functions, MediaConvert, CloudWatch, DynamoDB |

---

## 2. Initial Server Setup

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.1 Install Core Dependencies

```bash
sudo apt install -y git curl unzip wget software-properties-common

# Python 3.14 (matches Lambda runtime)
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install -y python3.14 python3.14-venv python3.14-dev python3-pip
python3.14 --version
```

### 2.2 Install Node.js 20 LTS

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version && npm --version
```

### 2.3 Install Docker (required for SAM builds)

```bash
sudo apt install -y docker.io
sudo systemctl enable docker && sudo systemctl start docker
sudo usermod -aG docker $USER
# Log out and back in for group change
exit
```

After reconnecting:
```bash
docker --version
```

### 2.4 Install AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
rm -rf aws awscliv2.zip
aws --version
```

### 2.5 Install AWS SAM CLI

```bash
curl -L "https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip" -o "sam.zip"
unzip sam.zip -d sam-installation && sudo ./sam-installation/install
rm -rf sam.zip sam-installation
sam --version
```

### 2.6 Install Nginx

```bash
sudo apt install -y nginx
sudo systemctl enable nginx && sudo systemctl start nginx
```

---

## 3. Configure AWS Credentials

Use the same region where your TAMS API stack is deployed.

```bash
aws configure
# OR verify instance profile:
aws sts get-caller-identity
```

Confirm you can see the existing TAMS API stack:
```bash
aws cloudformation describe-stacks --stack-name <YOUR_TAMS_API_STACK_NAME> --query "Stacks[0].StackStatus"
```

---

## 4. Clone the Tools Repository

```bash
cd /home/ubuntu
git clone <your-tams-tools-repo-url> tams-tools
cd tams-tools
```

---

## 5. Deploy the Backend (SAM)

The backend creates Lambda functions, Step Functions, SQS queues, EventBridge connections, Cognito identity pool, MediaConvert resources, and SSM parameters. It references your existing TAMS API stack via CloudFormation cross-stack imports.

```bash
cd /home/ubuntu/tams-tools/backend
sam build --use-container
```

### 5.1 First Deploy (Guided)

```bash

```

| Parameter | Value |
|-----------|-------|
| Stack Name | `tams-tools` (your choice) |
| AWS Region | Same as your TAMS API stack |
| **ApiStackName** | **Your existing TAMS API CloudFormation stack name** |
| OidcUrl | Leave blank (uses Cognito from TAMS API stack) |
| OidcClientId | Leave blank |
| OAuthClientId | Leave blank |
| OAuthClientSecret | Leave blank |
| OAuthAuthorizationEndpoint | Leave blank |
| DeployHlsApi | `No` (or `Yes` for HLS video player) |
| DeployIngestHls | `No` (or `Yes` for HLS ingestion) |
| DeployIngestFfmpeg | `No` (or `Yes` for FFmpeg transcoding) |
| DeployReplication | `No` (or `Yes` for content replication) |
| DeployLoopRecorder | `No` (or `Yes` for auto-segment cleanup) |
| Confirm changes before deploy | `y` |
| Allow SAM CLI IAM role creation | `y` |
| Save arguments to samconfig.toml | `y` |

> All auth parameters default to using Cognito from your already-deployed TAMS API stack. Only fill them in if you're using an external OIDC/OAuth provider.

### 5.2 Subsequent Deploys

```bash
sam build --use-container
sam deploy --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND
```

### 5.3 Enable Optional Components Later

```bash
sam deploy --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides DeployHlsApi=Yes DeployIngestHls=Yes DeployIngestFfmpeg=Yes DeployReplication=Yes DeployLoopRecorder=Yes
```

---

## 6. Build and Deploy the Frontend

### 6.1 Install Dependencies

```bash
cd /home/ubuntu/tams-tools/frontend
npm ci
```

### 6.2 Generate Environment Variables

The `envLocal` script reads `samconfig.toml` and queries both CloudFormation stacks (TAMS API + TAMS Tools) to auto-populate `.env.local`:

```bash
npm run envLocal
```

If the script fails, create `.env.local` manually:

```bash
cp .env.template .env.local
```

Get output values from both stacks:
```bash
# Replace with your actual stack names
aws cloudformation describe-stacks --stack-name <TAMS_API_STACK> --query "Stacks[0].Outputs" --output table
aws cloudformation describe-stacks --stack-name <TAMS_TOOLS_STACK> --query "Stacks[0].Outputs" --output table
```

Fill `.env.local` — the required values:

```env
VITE_APP_TAMS_API_ENDPOINT=<ApiEndpoint from TAMS API stack>
VITE_APP_OIDC_AUTHORITY=<OidcAuthority from TAMS Tools stack>
VITE_APP_OIDC_CLIENT_ID=<OidcClientId from TAMS Tools stack>
VITE_APP_OIDC_REDIRECT_URI=http://YOUR_EC2_PUBLIC_IP
VITE_APP_AWS_IDENTITY_POOL_ID=<IdentityPoolId from TAMS Tools stack>
VITE_APP_OMAKASE_EXPORT_EVENT_BUS=<OmakaseExportEventBus from TAMS Tools stack>
VITE_APP_OMAKASE_EXPORT_EVENT_PARAMETER=<OmakaseExportEventParameter from TAMS Tools stack>
VITE_APP_TAMS_AUTH_CONNECTION_ARN=<TamsAuthConnectionArn from TAMS Tools stack>
VITE_APP_AWS_MEDIACONVERT_ROLE_ARN=<MediaConvertRoleArn from TAMS Tools stack>
VITE_APP_AWS_MEDIACONVERT_BUCKET=<MediaConvertBucket from TAMS Tools stack>
```

Optional component variables (uncomment in `.env.local` if you deployed them):
```env
# VITE_APP_AWS_HLS_FUNCTION_URL=<HlsFunctionUrl>           # DeployHlsApi=Yes
# VITE_APP_AWS_INGEST_CREATE_NEW_FLOW_ARN=<IngestCreateNewFlowArn>  # DeployIngestHls or DeployIngestFfmpeg=Yes
# VITE_APP_AWS_HLS_INGEST_ENDPOINT=<HlsIngestEndpoint>     # DeployIngestHls=Yes
# VITE_APP_AWS_HLS_INGEST_ARN=<HlsIngestArn>               # DeployIngestHls=Yes
# VITE_APP_AWS_FFMPEG_ENDPOINT=<FfmpegEndpoint>             # DeployIngestFfmpeg=Yes
# VITE_APP_AWS_FFMPEG_COMMANDS_PARAMETER=<FfmpegCommandsParameter>  # DeployIngestFfmpeg=Yes
# VITE_APP_AWS_FFMPEG_BATCH_ARN=<FfmpegBatchArn>            # DeployIngestFfmpeg=Yes
# VITE_APP_AWS_FFMPEG_EXPORT_ARN=<FfmpegExportArn>          # DeployIngestFfmpeg=Yes
# VITE_APP_AWS_REPLICATION_CONNECTIONS_PARAMETER=<ReplicationConnectionsParameter>  # DeployReplication=Yes
# VITE_APP_AWS_REPLICATION_BATCH_ARN=<ReplicationBatchArn>  # DeployReplication=Yes
# VITE_APP_AWS_REPLICATION_CREATE_RULE_ARN=<ReplicationCreateRuleArn>  # DeployReplication=Yes
# VITE_APP_AWS_REPLICATION_DELETE_RULE_ARN=<ReplicationDeleteRuleArn>  # DeployReplication=Yes
# VITE_APP_AWS_LOOP_RECORDER_ARN=<LoopRecorderArn>          # DeployLoopRecorder=Yes
```

### 6.3 Build

```bash
npm run build
```

Output goes to `frontend/dist/`.

---

## 7. Configure Nginx

```bash
sudo tee /etc/nginx/sites-available/tams-tools > /dev/null <<'EOF'
server {
    listen 80;
    server_name _;

    root /home/ubuntu/tams-tools/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
}
EOF

sudo ln -sf /etc/nginx/sites-available/tams-tools /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

---

## 8. Update Cognito Callback URLs

The TAMS Tools stack creates a Cognito App Client with `http://localhost:5173` as the default callback. You need to add your EC2 URL:

```bash
TAMS_API_STACK="your-tams-api-stack-name"
TAMS_TOOLS_STACK="your-tams-tools-stack-name"

USER_POOL_ID=$(aws cloudformation describe-stacks --stack-name $TAMS_API_STACK \
  --query "Stacks[0].Outputs[?OutputKey=='UserPoolId'].OutputValue" --output text)

CLIENT_ID=$(aws cloudformation describe-stacks --stack-name $TAMS_TOOLS_STACK \
  --query "Stacks[0].Outputs[?OutputKey=='OidcClientId'].OutputValue" --output text)

aws cognito-idp update-user-pool-client \
  --user-pool-id $USER_POOL_ID \
  --client-id $CLIENT_ID \
  --callback-urls "http://localhost:5173" "http://YOUR_EC2_PUBLIC_IP" \
  --logout-urls "http://localhost:5173" "http://YOUR_EC2_PUBLIC_IP" \
  --allowed-o-auth-flows code \
  --allowed-o-auth-scopes openid email tams-api/admin tams-api/read tams-api/write tams-api/delete \
  --supported-identity-providers COGNITO \
  --allowed-o-auth-flows-user-pool-client \
  --explicit-auth-flows ALLOW_REFRESH_TOKEN_AUTH ALLOW_USER_SRP_AUTH
```

---

## 9. (Optional) HTTPS with Let's Encrypt

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

After enabling HTTPS:
1. Update `VITE_APP_OIDC_REDIRECT_URI` in `.env.local` to `https://your-domain.com`
2. Update Cognito callback URLs (step 8) to `https://`
3. Rebuild: `cd /home/ubuntu/tams-tools/frontend && npm run build`

---

## 10. Create a Cognito User

If you haven't already created a user in the TAMS API's Cognito User Pool:

```bash
aws cognito-idp admin-create-user \
  --user-pool-id $USER_POOL_ID \
  --username your-email@example.com \
  --user-attributes Name=email,Value=your-email@example.com Name=email_verified,Value=true \
  --temporary-password "TempPass123!"
```

---

## 11. Verify

1. Open `http://YOUR_EC2_PUBLIC_IP` in a browser
2. Login with Cognito credentials
3. You should see the TAMS Tools UI with your TAMS store data
4. Verify MediaConvert jobs view and export modal work

---

## 12. Maintenance

```bash
# Redeploy backend
cd /home/ubuntu/tams-tools/backend
git pull && sam build --use-container && sam deploy --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND

# Rebuild frontend
cd /home/ubuntu/tams-tools/frontend
git pull && npm ci && npm run build

# Check stack status
aws cloudformation describe-stacks --stack-name tams-tools --query "Stacks[0].StackStatus"

# Delete everything
cd /home/ubuntu/tams-tools/backend && sam delete
```

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `sam build` fails | Ensure Docker is running: `sudo systemctl start docker` |
| `sam build` out of memory | Add swap: `sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` |
| `sam deploy` fails with cross-stack import error | Verify your TAMS API stack name is correct and the stack is in `CREATE_COMPLETE` or `UPDATE_COMPLETE` state |
| Frontend blank page | Check browser console. Verify `.env.local` values match CloudFormation outputs. |
| OIDC redirect error | Cognito callback URLs must include your EC2 address (step 8) |
| `npm run envLocal` fails | Create `.env.local` manually from CloudFormation outputs (step 6.2) |
| 502 Bad Gateway | `sudo nginx -t` and `sudo tail -f /var/log/nginx/error.log` |
| Permission denied on dist/ | `sudo chown -R ubuntu:ubuntu /home/ubuntu/tams-tools/frontend/dist` |

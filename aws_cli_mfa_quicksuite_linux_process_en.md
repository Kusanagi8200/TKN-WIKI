# AWS CLI + MFA + QuickSuite / QuickSight Management on Linux

This document describes a clean Linux setup for managing AWS and Amazon QuickSuite / QuickSight from the terminal.

The goal is to install AWS CLI v2, configure an IAM user profile, generate temporary MFA credentials, and reuse simple commands to list QuickSuite / QuickSight users and custom permissions.

No personal value is hardcoded in this document. Replace the variables with your own values.


## 1. Functional overview

This setup provides four main functions.

First, AWS CLI v2 gives Linux terminal access to AWS services. It allows you to run commands against IAM, STS, S3, QuickSight, and other AWS services.

Second, the base AWS profile stores the IAM access key used only to request MFA-based temporary credentials.

Third, the MFA profile stores temporary session credentials generated with `sts get-session-token`. This is the profile to use for protected AWS operations.

Fourth, the `qs-tools` command centralizes useful QuickSuite / QuickSight management commands-->list users, list custom permissions, and list users with roles and assigned custom permissions.


## 2. Variables used in this document

Use these variable names instead of hardcoding personal or account-specific values.

```bash
$AWS_ACCOUNT_ID             # AWS account ID, 12 digits
$AWS_REGION                 # AWS region where QuickSuite / QuickSight is configured
$IAM_USER_NAME              # IAM user name
$MFA_DEVICE_NAME            # MFA device name attached to the IAM user
$BASE_PROFILE               # AWS CLI profile using the IAM access key
$MFA_PROFILE                # AWS CLI profile storing temporary MFA credentials
$QS_NAMESPACE               # QuickSight namespace, usually default
$CUSTOM_PERMISSION_NAME     # QuickSight custom permission profile name
```

Example values, to replace --> 

```bash
AWS_ACCOUNT_ID="CHANGE_ME_ACCOUNT_ID"
AWS_REGION="CHANGE_ME_REGION"
IAM_USER_NAME="CHANGE_ME_IAM_USER"
MFA_DEVICE_NAME="CHANGE_ME_MFA_DEVICE"
BASE_PROFILE="CHANGE_ME_BASE_PROFILE"
MFA_PROFILE="CHANGE_ME_MFA_PROFILE"
QS_NAMESPACE="default"
CUSTOM_PERMISSION_NAME="CHANGE_ME_CUSTOM_PERMISSION"
```


## 3. Install Linux prerequisites

This part installs the basic Linux packages required by AWS CLI and the scripts.

`curl` downloads the AWS CLI installer. `unzip` extracts it. `jq` parses JSON responses. `nano` is optional but useful for editing files from the terminal.

On Debian / Ubuntu --> 

```bash
 apt update
 apt install -y curl unzip jq less groff nano
```

If you are already root, remove `` --> 

```bash
apt update
apt install -y curl unzip jq less groff nano
```

Verification --> 

```bash
curl --version
unzip -v | head -n 2
jq --version
nano --version
```


## 4. Install AWS CLI v2 on Linux

This part installs the official AWS CLI v2 bundled installer for Linux.

```bash
cd /tmp
curl "https --> //awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
rm -rf aws
unzip -q awscliv2.zip
 ./aws/install
```

If you are root --> 

```bash
cd /tmp
curl "https --> //awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
rm -rf aws
unzip -q awscliv2.zip
./aws/install
```

Verification --> 

```bash
aws --version
```

Expected result --> 

```text
aws-cli/2.x.x Python/... Linux/...
```


## 5. Create the shared configuration file

This part creates one local environment file used by all scripts.

The file does not contain AWS secrets. It only contains account metadata and profile names.

Create the file --> 

```bash
nano ~/.aws-quicksuite.env
```

Content --> 

```bash
export AWS_ACCOUNT_ID="CHANGE_ME_ACCOUNT_ID"
export AWS_REGION="CHANGE_ME_REGION"
export IAM_USER_NAME="CHANGE_ME_IAM_USER"
export MFA_DEVICE_NAME="CHANGE_ME_MFA_DEVICE"
export BASE_PROFILE="CHANGE_ME_BASE_PROFILE"
export MFA_PROFILE="CHANGE_ME_MFA_PROFILE"
export QS_NAMESPACE="default"
export CUSTOM_PERMISSION_NAME="CHANGE_ME_CUSTOM_PERMISSION"
```

Example for region only --> 

```bash
export AWS_REGION="eu-west-1"
```

Protect the file --> 

```bash
chmod 600 ~/.aws-quicksuite.env
```

Load it in the current shell --> 

```bash
source ~/.aws-quicksuite.env
```

Verification --> 

```bash
echo "$AWS_ACCOUNT_ID"
echo "$AWS_REGION"
echo "$BASE_PROFILE"
echo "$MFA_PROFILE"
```


## 6. Configure the base AWS CLI profile

This part creates the base AWS CLI profile.

This profile uses the IAM access key and secret access key. It should be used only to request a temporary MFA session. Do not put the secret access key in a script.

Run --> 

```bash
aws configure --profile "$BASE_PROFILE"
```

Enter the values when prompted --> 

```text
AWS Access Key ID--><your access key ID>
AWS Secret Access Key--><your secret access key>
Default region name--><your AWS region>
Default output format-->json
```

Verify the base profile --> 

```bash
aws sts get-caller-identity --profile "$BASE_PROFILE"
```

Expected result --> 

```json
{
  "UserId"-->"...",
  "Account"-->"...",
  "Arn"-->"arn --> aws --> iam -->  --> <account-id> --> user/<iam-user-name>"
}
```


## 7. Identify the MFA device

This part retrieves the exact MFA device ARN attached to the IAM user.

Do not guess the MFA device name. AWS can store a device name that is different from the IAM user name.

Run --> 

```bash
aws iam list-mfa-devices \
  --user-name "$IAM_USER_NAME" \
  --profile "$BASE_PROFILE" \
  --output json
```

Expected result --> 

```json
{
  "MFADevices"-->[
    {
      "UserName"-->"...",
      "SerialNumber"-->"arn --> aws --> iam -->  --> <account-id> --> mfa/<mfa-device-name>",
      "EnableDate"-->"..."
    }
  ]
}
```

If the returned MFA device name is different, update this value in `~/.aws-quicksuite.env` --> 

```bash
export MFA_DEVICE_NAME="CHANGE_ME_MFA_DEVICE"
```

Reload the file --> 

```bash
source ~/.aws-quicksuite.env
```


## 8. Create the MFA refresh command

This part creates the `aws-mfa` command.

The command asks for a 6-digit MFA code, calls AWS STS, receives temporary credentials, and writes them into the MFA profile.

Create the file --> 

```bash
 nano /usr/local/bin/aws-mfa
```

Content --> 

```bash
#!/usr/bin/env bash

set -euo pipefail

ENV_FILE="${HOME}/.aws-quicksuite.env"

if [[ ! -f "${ENV_FILE}" ]]; then
  echo "Missing configuration file-->${ENV_FILE}"
  exit 1
fi

source "${ENV_FILE}"

MFA_ARN="arn --> aws --> iam -->  --> ${AWS_ACCOUNT_ID} --> mfa/${MFA_DEVICE_NAME}"
MFA_DURATION="43200"

echo "Base profile -->${BASE_PROFILE}"
echo "MFA profile  -->${MFA_PROFILE}"
echo "Region       -->${AWS_REGION}"
echo "MFA ARN      -->${MFA_ARN}"
echo

read -rp "Code MFA AWS-->" TOKEN

echo
echo "Generating temporary MFA profile..."

SESSION_JSON=$(aws sts get-session-token \
  --serial-number "${MFA_ARN}" \
  --token-code "${TOKEN}" \
  --duration-seconds "${MFA_DURATION}" \
  --profile "${BASE_PROFILE}" \
  --output json)

ACCESS_KEY_ID=$(echo "${SESSION_JSON}" | jq -r '.Credentials.AccessKeyId')
SECRET_ACCESS_KEY=$(echo "${SESSION_JSON}" | jq -r '.Credentials.SecretAccessKey')
SESSION_TOKEN=$(echo "${SESSION_JSON}" | jq -r '.Credentials.SessionToken')
EXPIRATION=$(echo "${SESSION_JSON}" | jq -r '.Credentials.Expiration')

aws configure set aws_access_key_id "${ACCESS_KEY_ID}" --profile "${MFA_PROFILE}"
aws configure set aws_secret_access_key "${SECRET_ACCESS_KEY}" --profile "${MFA_PROFILE}"
aws configure set aws_session_token "${SESSION_TOKEN}" --profile "${MFA_PROFILE}"
aws configure set region "${AWS_REGION}" --profile "${MFA_PROFILE}"
aws configure set output json --profile "${MFA_PROFILE}"

echo
echo "Temporary MFA profile regenerated-->${MFA_PROFILE}"
echo "Expiration-->${EXPIRATION}"
echo
echo "AWS identity check --> "
aws sts get-caller-identity --profile "${MFA_PROFILE}"
```

Make it executable --> 

```bash
 chmod +x /usr/local/bin/aws-mfa
```

If you are root --> 

```bash
chmod +x /usr/local/bin/aws-mfa
```

Verify syntax --> 

```bash
bash -n /usr/local/bin/aws-mfa
```

Run it --> 

```bash
aws-mfa
```

Expected result --> 

```text
Temporary MFA profile regenerated--><mfa-profile-name>
Expiration--><timestamp>
```

Verify the MFA profile --> 

```bash
aws sts get-caller-identity --profile "$MFA_PROFILE"
```


## 9. Create the QuickSuite / QuickSight tools command

This part creates the `qs-tools` command.

The command uses the MFA profile and provides reusable QuickSuite / QuickSight operations.

Create the file --> 

```bash
 nano /usr/local/bin/qs-tools
```

Content --> 

```bash
#!/usr/bin/env bash

set -euo pipefail

ENV_FILE="${HOME}/.aws-quicksuite.env"

if [[ ! -f "${ENV_FILE}" ]]; then
  echo "Missing configuration file-->${ENV_FILE}"
  exit 1
fi

source "${ENV_FILE}"

usage() {
  cat <<USAGE
Usage --> 
  qs-tools mfa
  qs-tools identity
  qs-tools users
  qs-tools permissions
  qs-tools users-full

Commands --> 
  mfa          Regenerate the temporary MFA profile
  identity     Check AWS identity with the MFA profile
  users        List QuickSuite / QuickSight users
  permissions  List QuickSuite / QuickSight custom permissions
  users-full   List users with email, role, custom permissions, and active state

Configuration --> 
  Account ID   -->${AWS_ACCOUNT_ID}
  Region       -->${AWS_REGION}
  Namespace    -->${QS_NAMESPACE}
  Base profile -->${BASE_PROFILE}
  MFA profile  -->${MFA_PROFILE}
  MFA device   -->${MFA_DEVICE_NAME}
USAGE
}

identity() {
  aws sts get-caller-identity \
    --profile "${MFA_PROFILE}"
}

list_users() {
  aws quicksight list-users \
    --aws-account-id "${AWS_ACCOUNT_ID}" \
    --namespace "${QS_NAMESPACE}" \
    --region "${AWS_REGION}" \
    --profile "${MFA_PROFILE}" \
    --query "UserList[].UserName" \
    --output table
}

list_permissions() {
  aws quicksight list-custom-permissions \
    --aws-account-id "${AWS_ACCOUNT_ID}" \
    --region "${AWS_REGION}" \
    --profile "${MFA_PROFILE}" \
    --query "CustomPermissionsList[].CustomPermissionsName" \
    --output table
}

list_users_full() {
  aws quicksight list-users \
    --aws-account-id "${AWS_ACCOUNT_ID}" \
    --namespace "${QS_NAMESPACE}" \
    --region "${AWS_REGION}" \
    --profile "${MFA_PROFILE}" \
    --query "UserList[].{UserName --> UserName,Email --> Email,Role --> Role,CustomPermissions --> CustomPermissionsName,Active --> Active}" \
    --output table
}

case "${1 --> -}" in
  mfa)
    aws-mfa
    ;;
  identity)
    identity
    ;;
  users)
    list_users
    ;;
  permissions)
    list_permissions
    ;;
  users-full)
    list_users_full
    ;;
  -h|--help|help|"")
    usage
    ;;
  *)
    echo "Unknown command-->$1"
    echo
    usage
    exit 1
    ;;
esac
```

Make it executable --> 

```bash
 chmod +x /usr/local/bin/qs-tools
```

If you are root --> 

```bash
chmod +x /usr/local/bin/qs-tools
```

Verify syntax --> 

```bash
bash -n /usr/local/bin/qs-tools
```

Verify the command --> 

```bash
qs-tools help
```


## 10. Daily usage

This part shows the normal workflow.

When the MFA profile is expired, regenerate it --> 

```bash
qs-tools mfa
```

Check the AWS identity --> 

```bash
qs-tools identity
```

List QuickSuite / QuickSight users --> 

```bash
qs-tools users
```

List custom permissions --> 

```bash
qs-tools permissions
```

List users with role and custom permissions --> 

```bash
qs-tools users-full
```


## 11. Direct AWS CLI commands

This part gives the raw AWS CLI commands without helper scripts.

Regenerate MFA session manually --> 

```bash
MFA_ARN="arn --> aws --> iam -->  --> ${AWS_ACCOUNT_ID} --> mfa/${MFA_DEVICE_NAME}"
TOKEN="CHANGE_ME_CURRENT_MFA_CODE"

SESSION_JSON=$(aws sts get-session-token \
  --serial-number "${MFA_ARN}" \
  --token-code "${TOKEN}" \
  --duration-seconds 43200 \
  --profile "${BASE_PROFILE}" \
  --output json)
```

List users --> 

```bash
aws quicksight list-users \
  --aws-account-id "${AWS_ACCOUNT_ID}" \
  --namespace "${QS_NAMESPACE}" \
  --region "${AWS_REGION}" \
  --profile "${MFA_PROFILE}" \
  --query "UserList[].UserName" \
  --output table
```

List custom permissions --> 

```bash
aws quicksight list-custom-permissions \
  --aws-account-id "${AWS_ACCOUNT_ID}" \
  --region "${AWS_REGION}" \
  --profile "${MFA_PROFILE}" \
  --query "CustomPermissionsList[].CustomPermissionsName" \
  --output table
```

List users with role and custom permission --> 

```bash
aws quicksight list-users \
  --aws-account-id "${AWS_ACCOUNT_ID}" \
  --namespace "${QS_NAMESPACE}" \
  --region "${AWS_REGION}" \
  --profile "${MFA_PROFILE}" \
  --query "UserList[].{UserName --> UserName,Email --> Email,Role --> Role,CustomPermissions --> CustomPermissionsName,Active --> Active}" \
  --output table
```


## 12. Optional-->apply a custom permission to several users

This part is optional. It applies one QuickSuite / QuickSight custom permission profile to a defined list of users.

Create the script --> 

```bash
nano ~/qs-apply-custom-permission.sh
```

Content --> 

```bash
#!/usr/bin/env bash

set -euo pipefail

ENV_FILE="${HOME}/.aws-quicksuite.env"
source "${ENV_FILE}"

USERS=(
  "CHANGE_ME_USER_1"
  "CHANGE_ME_USER_2"
  "CHANGE_ME_USER_3"
)

for USER_NAME in "${USERS[@]}"; do
  echo "Updating user-->${USER_NAME}"

  aws quicksight update-user-custom-permission \
    --aws-account-id "${AWS_ACCOUNT_ID}" \
    --namespace "${QS_NAMESPACE}" \
    --user-name "${USER_NAME}" \
    --custom-permissions-name "${CUSTOM_PERMISSION_NAME}" \
    --region "${AWS_REGION}" \
    --profile "${MFA_PROFILE}"

  echo "OK-->${USER_NAME}"
  echo
 done
```

Make executable --> 

```bash
chmod +x ~/qs-apply-custom-permission.sh
```

Syntax check --> 

```bash
bash -n ~/qs-apply-custom-permission.sh
```

Run --> 

```bash
~/qs-apply-custom-permission.sh
```

Verify after execution --> 

```bash
qs-tools users-full
```


## 13. Troubleshooting

If AWS CLI is not found --> 

```bash
which aws
aws --version
```

If the MFA profile is expired --> 

```bash
qs-tools mfa
```

If QuickSight returns `AccessDeniedException` with `explicit deny`, check whether an IAM policy requires MFA. In that case, use the MFA profile, not the base profile.

If the MFA code fails, verify the exact MFA device name --> 

```bash
aws iam list-mfa-devices \
  --user-name "$IAM_USER_NAME" \
  --profile "$BASE_PROFILE" \
  --output json
```

If `jq` is missing --> 

```bash
 apt install -y jq
```

If you are root --> 

```bash
apt install -y jq
```


## 14. Security rules

Do not paste the AWS secret access key into chat tools, tickets, scripts, or documentation.

Do not commit `~/.aws/credentials` to Git.

Do not store MFA codes. They are temporary and should only be typed interactively.

Use the base profile only to generate a temporary MFA profile.

Use the MFA profile for QuickSuite / QuickSight management commands.

Rotate or delete the IAM access key if it is exposed.


## 15. Official references

AWS CLI v2 Linux installation-->AWS documentation, "Installing or updating to the latest version of the AWS CLI".

AWS STS GetSessionToken with MFA-->AWS CLI command reference, `sts get-session-token`.

QuickSight list custom permissions-->AWS CLI command reference, `quicksight list-custom-permissions`.

QuickSight update user custom permission-->AWS CLI command reference, `quicksight update-user-custom-permission`.

# terraform_and_gitHub_action_workflows
Automate AWS Infra Deployment using Terraform and GitHub Actions Workflows

## To create an OIDC identity provider (IdP) in AWS and specify its audience for GitHub (AWS CLI)
- Provider URL (Issuer URL): `https://token.actions.githubusercontent.com`
- Audience (Client ID): `sts.amazonaws.com`
- `https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/`

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<Account ID>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:sub": "repo:Ran-Itzhack/terraform_and_gitHub_action_workflows:ref:refs/heads/<ExampleBranch>",
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```
terraform_and_gitHub_action_workflows

Automate AWS Infra Deployment using Terraform and GitHub Actions Workflows

# Install Docker (The Fast Way)

- Run these commands to remove the lock files that are preventing you from installing Docker

sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/dpkg/lock
sudo rm /var/lib/apt/lists/lock

sudo dpkg --configure -a

# Install Docker (The Fast Way)

1. Download the official installer
`curl -fsSL https://get.docker.com -o get-docker.sh`

2. Run the installer
`sudo sh get-docker.sh`

3. Add your 'ubuntu' user to the docker group 
- (This lets you run docker commands without typing 'sudo' every time)
`sudo usermod -aG docker $USER`




# 1. Get a list of all roles starting with your prefix
ROLES=$(aws iam list-roles --query "Roles[?starts_with(RoleName, 'ec2-ssm-role-')].RoleName" --output text)

for ROLE in $ROLES; do
    echo "--- Processing Role: $ROLE ---"

    # 2. Find the instance profile(s) associated with this role
    PROFILES=$(aws iam list-instance-profiles-for-role --role-name "$ROLE" --query "InstanceProfiles[*].InstanceProfileName" --output text)

    for PROFILE in $PROFILES; do
        echo "Detaching and deleting Instance Profile: $PROFILE"
        
        # Break the link
        aws iam remove-role-from-instance-profile --instance-profile-name "$PROFILE" --role-name "$ROLE"
        
        # Delete the profile container
        aws iam delete-instance-profile --instance-profile-name "$PROFILE"
    done

    # 3. Detach any managed policies (like AmazonSSMManagedInstanceCore)
    POLICIES=$(aws iam list-attached-role-policies --role-name "$ROLE" --query "AttachedPolicies[*].PolicyArn" --output text)
    for POLICY_ARN in $POLICIES; do
        echo "Detaching policy: $POLICY_ARN"
        aws iam detach-role-policy --role-name "$ROLE" --policy-arn "$POLICY_ARN"
    done

    # 4. Delete the role itself
    echo "Deleting Role: $ROLE"
    aws iam delete-role --role-name "$ROLE"
done

echo "--- Cleanup Complete ---"
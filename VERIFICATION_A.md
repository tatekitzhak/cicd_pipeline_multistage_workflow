# CD Pipeline Verification Checklist

Use this checklist to ensure the pipeline is secure and the EC2 instance is ready for Docker deploy.

---

## 1. IAM: GitHub OIDC (OpenID Connect) vs long‑lived keys

- **Prefer OIDC**: The Terraform code creates an **IAM Identity Provider** for GitHub and an **IAM Role** with a trust policy that allows only GitHub Actions to assume it (no long‑lived access keys).
- **What Terraform creates**:
  - `aws_iam_openid_connect_provider.github` – GitHub’s OIDC provider in your account.
  - `aws_iam_role.github_actions` – Role with a trust policy that:
    - Allows `sts:AssumeRoleWithWebIdentity`.
    - Restricts to `token.actions.githubusercontent.com` and subject `repo:<org/repo>:*`.
- **What you must do**:
  - Set **Repository variable** (or Terraform variable) `github_repository` to your repo (e.g. `myorg/ci_cd_pipeline`).
  - After first apply, set GitHub **secret** `AWS_ROLE` to the role ARN (Terraform output `github_actions_role_arn`).
  - The workflow uses `aws-actions/configure-aws-credentials@v4` with `role-to-assume: ${{ secrets.AWS_ROLE }}` and `id-token: write` (no `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`).

---

## 2. EC2 Security Group: Inbound for the Docker app

- The Security Group allows **inbound** traffic so users can reach the container:
  - **Port 80** – e.g. when the container is run with `-p 80:3000`.
  - **Port 8080** – if your app is published on 8080.
  - Ports 22 (SSH) and 443 (HTTPS) are also allowed.
- **Check**: In AWS Console → EC2 → Security Groups, confirm the group used by the instance has inbound rules for **80** and **8080** (and 22/443 if needed).

---

## 3. Docker group (no sudo required for Docker)

- By default, Docker requires `sudo`. SSM **RunShellScript** runs as **root**, so `docker` (without sudo) works.
- The **user_data** script `install_docker_on_ubuntu_aws_ec2.sh` also adds users `ubuntu` and `ec2-user` to the **docker** group so that non‑root users can run `docker` without sudo if needed.
- **Check on the instance** (e.g. via SSM or SSH):
  - `groups ubuntu` → should include `docker`.
  - Running `docker ps` as `ubuntu` (no sudo) should work.

---

## 4. Docker daemon: Start on boot

- The install script runs:
  - `systemctl enable docker.service`
  - `systemctl start docker.service`
- **Check on the instance**:
  - `sudo systemctl is-enabled docker` → should print **enabled**.
  - `sudo systemctl status docker` → should show **active (running)**.

The CD workflow has a step **“Verify Docker enabled on boot (SSM)”** that runs `systemctl is-enabled docker` on the instance via SSM.

---

## 5. EC2 ready for CD (Docker installed and running)

- **user_data** runs at first boot; Docker install can take a few minutes.
- The CD job does **not** run deploy immediately; it:
  1. **Waits for Docker**: Sends an SSM command that loops until `docker info` succeeds (with a timeout).
  2. **Verifies** `systemctl is-enabled docker`.
  3. Then runs **docker login**, **docker pull**, and **docker run** via SSM.
- So the pipeline ensures the instance is **ready for a CD pull** before running deploy.

---

## 6. GitHub secrets and variables

| Name | Where | Purpose |
|------|--------|--------|
| `AWS_ROLE` | Repo secret | ARN of the GitHub OIDC IAM role (from Terraform output `github_actions_role_arn`) |
| `AWS_REGION` | Repo secret | AWS region (e.g. `us-east-2`) |
| `DOCKERHUB_USERNAME` | Repo secret | Docker Hub username for `docker login` |
| `DOCKERHUB_TOKEN` | Repo secret | Docker Hub token (or password) for `docker login` |
| `DOCKER_IMAGE` | Repo secret (optional) | Override image (e.g. `myuser/cddemo:latest`). If unset, uses `$DOCKERHUB_USERNAME/cddemo:latest` |
| `github_repository` | Terraform variable / tfvars | GitHub repo for OIDC trust, e.g. `myorg/ci_cd_pipeline` |

---

## 7. Terraform: First run and role permissions

- The **first** Terraform apply (that creates the OIDC provider and role) must be run with credentials that can create IAM resources (e.g. IAM user with admin, or another role). After that, you can switch the workflow to use the OIDC role.
- The **GitHub Actions IAM role** needs:
  - **SSM + EC2 describe**: Included in Terraform (`iam_github_oidc.tf`).
  - **Terraform resources**: Attach an additional policy (e.g. **AdministratorAccess** or a custom policy for VPC, EC2, IAM, etc.) to `aws_iam_role.github_actions` so the Terraform and deploy jobs can create/manage resources and run SSM.

---

## Quick verification commands (on the EC2 instance)

```bash
# Docker enabled on boot
sudo systemctl is-enabled docker

# Docker running
sudo systemctl status docker

# User in docker group (optional)
groups ubuntu

# Docker works without sudo (as ubuntu)
docker info
```

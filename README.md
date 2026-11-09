# Terraform AWS DevOps Assignment

This repository implements all three requested deployment patterns:
1. Single EC2: Flask on TCP 5000 and Express on TCP 3000.
2. Separate EC2: Flask and Express on distinct instances.
3. Docker + ECR + ECS Fargate + VPC + Application Load Balancer.

**Region default:** `ap-south-1` (Mumbai). Change it in the relevant `variables.tf` or tfvars file if needed.

## Prerequisites
- AWS account with billing enabled and permissions to create EC2, VPC, IAM, S3, ECR, ECS, ELB and CloudWatch resources.
- AWS CLI v2 configured: `aws configure`; verify with `aws sts get-caller-identity`.
- Terraform >= 1.5, Docker Desktop/Engine, Git, and (for SSH) an SSH client.
- Set a cost budget/alarm in AWS Billing. Fargate, ALB, NAT gateways and public IPv4 addresses can incur charges. This project avoids NAT gateways, but ALB/ECS/EC2/ECR/S3 still cost money.

## Repository layout
- `apps/flask`, `apps/express`: app source and Dockerfiles.
- `part1-single-ec2`: one EC2 deployment.
- `part2-two-ec2`: two EC2 deployment.
- `part3-ecs-ecr-alb`: ECR, VPC, ECS Fargate and ALB.
- `state-bootstrap`: creates an encrypted/versioned private S3 state bucket.
- `scripts`: build/push and verification helper scripts.
- `Documentation.docx`: submission report template and evidence checklist.

## Important security notes
This is a learning/demo deployment. Part 1/2 expose demo app ports publicly as requested. SSH is restricted to your own public IP using `ssh_cidr`. Replace `YOUR_PUBLIC_IP/32` before apply. For production, do not expose Flask directly; put it behind the ALB, use HTTPS, private subnets/NAT or VPC endpoints, secrets management, least-privilege IAM, and tighter ingress. Never commit AWS credentials, state files, or real `.tfvars`.

## One-time AWS setup
1. Install/configure AWS CLI and Terraform. Run `aws sts get-caller-identity`.
2. Create the state bucket with a globally unique name. From `state-bootstrap`:
   - `cp terraform.tfvars.example terraform.tfvars`
   - Edit `state_bucket` to a globally unique lowercase name.
   - `terraform init`
   - `terraform fmt -recursive`
   - `terraform validate`
   - `terraform plan`
   - `terraform apply`
3. Keep this S3 bucket. It has versioning, encryption and public access blocks enabled. S3 native lockfile is enabled for Part 3 backend.

## Part 1 — single EC2
```bash
cd part1-single-ec2
cp terraform.tfvars.example terraform.tfvars
# Edit ssh_cidr to YOUR_PUBLIC_IP/32
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
terraform output
```
Wait 3–5 minutes for cloud-init. Open the `frontend_url` output in a browser. Test Flask with `curl http://PUBLIC_IP:5000/api/hello`. Test Express with `curl http://PUBLIC_IP:3000/`. On the instance, inspect `sudo cloud-init status --wait`, `sudo systemctl status flask express`, and `sudo journalctl -u flask -u express --no-pager`.

Destroy when evidence is captured: `terraform destroy`.

## Part 2 — separate EC2 instances
```bash
cd part2-two-ec2
cp terraform.tfvars.example terraform.tfvars
# Edit ssh_cidr
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
terraform output
```
Wait for user-data completion. Test the frontend URL (Express) and backend URL (Flask). Frontend user-data receives the backend's private IP; the backend SG allows TCP 5000 from the frontend SG and public demo testing. Inspect both instances' cloud-init/systemd logs if needed. Destroy when finished: `terraform destroy`.

## Part 3 — Docker, ECR, ECS Fargate and ALB

### Build and push images (two-phase deployment)
ECR repositories and the ALB/VPC/ECS definitions are Terraform-managed. The first Terraform apply creates ECR and the infrastructure, but ECS tasks cannot start until images have been pushed. To avoid that bootstrap dependency, the task definitions use tag `v1`; perform the steps below:
1. Configure the S3 backend before init (see next section).
2. `cd part3-ecs-ecr-alb`, copy `terraform.tfvars.example` to `terraform.tfvars`, edit if needed.
3. Copy `backend.hcl.example` to `backend.hcl`, replace the bucket with the state bucket you created. Then run:
```bash
terraform init -backend-config=backend.hcl
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```
4. From the repository root, log in/build/push:
```bash
chmod +x scripts/part3-build-push.sh
AWS_REGION=ap-south-1 ./scripts/part3-build-push.sh
```
Or manually:
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
docker build -t FLASK_ECR_URI:v1 apps/flask
docker push FLASK_ECR_URI:v1
docker build -t EXPRESS_ECR_URI:v1 apps/express
docker push EXPRESS_ECR_URI:v1
```
5. Force ECS services to deploy the pushed images:
```bash
aws ecs update-service --cluster terraform-demo-p3-cluster --service terraform-demo-p3-flask --force-new-deployment --region ap-south-1
aws ecs update-service --cluster terraform-demo-p3-cluster --service terraform-demo-p3-express --force-new-deployment --region ap-south-1
```
If you changed `project`, use the actual names from Terraform outputs/AWS console. Wait until both ECS services show ` desiredCount=1 runningCount=1` and both target groups show healthy targets.
6. Open `alb_url` output. `/` is Express; `/api/hello` is Flask. Check ECS task logs in CloudWatch under `/ecs/terraform-demo-p3/flask` and `/ecs/terraform-demo-p3/express`.

### State backend details
Terraform backend blocks cannot interpolate variables. The `backend.hcl` file provides the bucket, key and region at init time. Keep backend config local and do not commit sensitive values. The backend bucket must exist before `terraform init`. If you change backend settings, run `terraform init -reconfigure`.

### Teardown
Destroy Part 3 first (from `part3-ecs-ecr-alb`: `terraform destroy`). ECR repositories are configured with `force_delete = true` so Terraform can remove demo images. Then optionally destroy the state bucket last from `state-bootstrap` (`terraform destroy`) only after all remote states are no longer needed. Never delete a state bucket while it contains state you still need.

## Troubleshooting
- EC2 user-data: `sudo cloud-init status --long`; `sudo tail -n 200 /var/log/cloud-init-output.log`; `sudo systemctl status flask express`.
- ECS tasks stopped: inspect ECS Service > Events, stopped task reason, and CloudWatch logs. Confirm images exist in ECR and task execution role is attached.
- ALB targets unhealthy: verify target health paths `/health` (Flask) and `/healthz` (Express), container ports, task SG ingress from ALB SG, and service subnets.
- Cannot connect: check instance/ALB public DNS, route table default route to IGW, security group rules, and that deployment completed.
- Terraform errors: `terraform fmt -recursive`, `terraform validate`, check AWS region/permissions/quota, and inspect exact error before retrying.

## Mandatory submission evidence
Capture genuine screenshots from your AWS account/terminal; do not use sample or fabricated screenshots. Include:
1. `terraform version`, `aws sts get-caller-identity` (hide account ID if desired).
2. Each part's `terraform init`, `terraform validate`, `terraform plan` and successful `terraform apply`.
3. Part 1 EC2 instance details + browser/curl proof for ports 3000 and 5000.
4. Part 2 both EC2 instances + browser/curl proof for frontend and backend.
5. Part 3 ECR repositories/images, ECS cluster/services with 1/1 tasks, healthy ALB target groups, and browser proof for `/` and `/api/hello`.
6. `terraform output` for each part and final project folder/ZIP.
7. GitHub repository page showing your pushed project.

Add screenshots to `evidence/` and update the report with captions and short explanations. Redact credentials, access keys, account-sensitive data and public IPs if required by your course.

## GitHub and ZIP submission
Create a GitHub repository, then from the project root:
```bash
git init
git add .
git commit -m "Complete Terraform AWS deployment assignment"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git
git push -u origin main
```
Replace the URL with your actual repository. Put that real URL in `GITHUB_REPOSITORY.txt` and the report. Keep `.tfstate`, `.terraform`, credentials and real `.tfvars` out of Git. Finally zip the entire `Terraform_YourName` folder and upload that one ZIP to the portal.

## Useful verification commands
```bash
terraform fmt -check -recursive
terraform validate
aws ec2 describe-instances --region ap-south-1
aws ecs list-services --cluster terraform-demo-p3-cluster --region ap-south-1
aws ecr describe-repositories --region ap-south-1
```

**Cost reminder:** run `terraform destroy` after capturing evidence. Check AWS Billing/Cost Explorer for any remaining resources (especially load balancers, EC2 instances, public IPv4, ECR images and S3 state bucket).

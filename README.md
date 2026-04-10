High-level Plan for DevOps Process
Client-CRC
BU-Wellington

 	DevOps CI/CD Design for Amazon Connect Using GitHub Actions and Terraform:
INDEX:
1.	Environment Strategy and Account Isolation.
2.	CI/CD Tooling Overview.
3.	GitHub to AWS Authentication Using OIDC.
3.1 OIDC Authentication Flow.
4.	AWS IAM Design for Cross-Account role.
 4.1 AWS Account Structure.
 4.2 IAM Roles Required for CI/CD.
5.	Terraform Management Strategy.
6.	GitHub Repository Strategy (Mono Repo and Multi Repo).
7.	Pipeline Model and Deployment Strategy.
8.	Environment Promotion Strategy (Branch Based Deployment Model).
9.	CI/CD Workflow – Design and Planning Overview.
9.1 CI Workflow (Pull Request Validation).
9.2 CD Workflow (Environment Deployment).
10.	Overall End to End Delivery Flow.
11.	Runner Strategy – GitHub Actions Execution Model
12.	Destroy Workflow Policy.





1.	Environment Isolation Strategy (Best Practice):
AWS Organization:
•	CRC-Dev-Account.
•	CRC -UAT-Account.
•	CRC -Prod-Account.
Benefits:
•	Security isolation.
•	Billing separation.
•	Access control.
•	Reduced blast radius.

2.	CI/CD Tooling Overview:
GitHub Enterprise:
 	GitHub is used as the central source code repository for all Terraform code related to Amazon Connect infrastructure and configuration. It also acts as the CI/CD orchestration platform through GitHub Actions workflows.
GitHub Actions:
 	GitHub Actions is used to automate CI and CD workflows, including Terraform validation, planning, and deployment across Dev, UAT, and Production environments.
Terraform (Infrastructure as Code):
 	Terraform is used to provision and manage AWS infrastructure and Amazon Connect resources in a consistent and repeatable manner. All infrastructure changes are applied through Terraform code executed via CI/CD pipelines.
Authentication Mechanism:
 	CI/CD workflows authenticate to AWS using OpenID Connect (OIDC), eliminating the need for long lived AWS access keys in GitHub. Temporary credentials are obtained at runtime.
Remote State Management:
 	Terraform state is stored remotely in Amazon S3 with state locking enabled to prevent concurrent updates and ensure deployment consistency.
Environment Aware Deployments:
 	CI/CD workflows are environment aware and map Git branches to corresponding AWS environments (Dev, UAT, Prod) during deployment.
3	GitHub Authenticates to AWS (OIDC): 
GitHub Authentication to AWS using OpenID Connect (OIDC) is a secure method that allows GitHub Actions workflows to access AWS resources using temporary, short-lived credentials, eliminating the need for storing long-term AWS access keys as GitHub secrets. The process involves configuring a trust relationship between AWS IAM and GitHub's OIDC provider.

 

3.1 How the OIDC Flow Works:
•	Trust Establishment: 
o	An AWS IAM OIDC Identity Provider is created, which trusts the GitHub OIDC provider (https://token.actions.githubusercontent.com) to issue tokens. IAM role trust policies will validate (audience = sts.amazonaws.com) and repository/branch claims for least-privilege access.
•	Role Creation: 
o	An IAM role with a restrictive trust policy is created in AWS. This policy defines which specific GitHub repositories, branches, or environments are allowed to assume the role.
•	Token Request: 
During a workflow run, a GitHub Action step requests a unique, short-lived JSON Web Token (JWT) from GitHub's OIDC provider.
•	Token Validation & Credential Issue: 
The workflow presents the JWT to the AWS Security Token Service (STS). AWS validates the token's signature and claims against the pre-configured trust policy. If the conditions are met, AWS STS issues temporary access credentials (access key ID, secret access key, and session token).
•	Access Resources: 
The workflow uses these temporary credentials to interact with AWS services for the duration of the job.
Low-Level Steps for OIDC Authentication:
Ensure the following accounts are already provisioned via AWS Control Tower:
•	Dev Account
•	UAT Account
•	Prod Account
Login to each account individually before performing the below steps.
Create OIDC Identity Provider (Repeat in each account).
Step-by-Step:
•	Login to AWS Console
•	Navigate to IAM → Identity Providers
•	Click “Add Provider”
•	Select Provider Type: OpenID Connect
•	Enter Provider URL: https://token.actions.githubusercontent.com
•	Click “Get thumbprint” (auto fetch)
•	Audience: sts.amazonaws.com
•	Click “Add Provider”
Verify provider ARN is created:
Format: arn:aws:iam:::oidc-provider/token.actions.githubusercontent.com
•	Create IAM Role (Repeat per environment)
Dev Role
•	Go to IAM → Roles → Create Role.
•	Select Trusted Entity Type: Web Identity.
•	Choose Provider: token.actions.githubusercontent.com.
•	Audience: sts.amazonaws.com.
•	Click Next.

•	Attach Permissions
Attach policies based on environment:
o	Dev: PowerUserAccess (or scoped custom policy).
o	UAT: Limited deployment permissions.
o	Prod: Strict least privilege policy.
•	Configure Trust Policy (Critical Step)
o	Capture Role ARN (Important)
o	After role creation, copy ARN:
o	arn:aws:iam:::role/CRC-GH-Dev-Role
o	Repeat for UAT and Prod roles
•	GitHub Repository Setup
Enable OIDC in GitHub Actions
•	Go to Settings → Actions → General
•	Ensure Actions are enabled
•	Under Workflow permissions:
•	Select “Read repository contents”
•	Enable “Allow GitHub Actions to create and use OIDC tokens”
Add Role ARN in GitHub (Best Practice)
Option 1: Using Repository Secrets
•	 Go to Settings → Secrets and Variables → Actions
•	 Click “New Repository Secret”
Add secrets:
• Name: AWS_DEV_ROLE_ARN Value: arn:aws:iam:::role/CRC-GH-Dev-Role
• Name: AWS_UAT_ROLE_ARN Value: arn:aws:iam:::role/CRC-GH-UAT-Role
• Name: AWS_PROD_ROLE_ARN Value: arn:aws:iam:::role/CRC-GH-Prod-Role

Option 2: Using Environment Secrets
•	Go to Settings → Environments
•	Create environments:
o	Dev
o	 Uat
o	 prod
Inside each environment, add secret:
(Dev Environment): - 
Name: AWS_ROLE_ARN - Value: arn:aws:iam:::role/CRC-GH-Dev-Role
Repeat for UAT and Prod.
Configure Environment Protection
• Dev: No restriction
• UAT: Add required reviewers
• Prod: Add required reviewers + optional wait timer
4.	AWS IAM Design for Cross-Account role:
GitHub Actions should obtain short lived AWS credentials via OIDC, then deploy Amazon Connect configuration via Terraform into DEV/UAT/PROD accounts, without any long lived AWS keys.
•	 
•	AWS SIDE — COMPLETE SETUP:
Prerequisites:
1.	You must have permission to create 
o IAM Identity Provider (OIDC) 
o IAM Roles and Policies.
2.	Confirm your GitHub repository will run workflows with OIDC (No stored AWS keys). GitHub describes OIDC as the method to avoid long lived credentials.
Create GitHub OIDC Identity Provider (in Shared Services Account)
AWS Console steps:
  	 1.Go to AWS IAM Console ➜ Identity providers ➜ Add provider. 
 2. Provider type: OpenID Connect. 
 3. Provider URL: https://token.actions.githubusercontent.com 
 4. Audience (Client ID): sts.amazonaws.com 
 5. Save.
Create Role 1 — CRC-GitHub-OIDC-Role (Shared Services Account)
Create the role (Console)
1.IAM ➜ Roles ➜ Create role 
2. Trusted entity type: Web identity 
 3. Identity provider: select the GitHub provider you created. 
4. Audience: sts.amazonaws.com 
5. Create role with name: CRC-GitHub-OIDC-Role
Update the Trust Policy
restrict by GitHub Environment
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<SHARED_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:environment:DEV"
        }
      }
    }
  ]
}


Attach permissions to CRC-GitHub-OIDC-Role (ONLY assume workload roles)  
          This role must only be able to assume your DEV/UAT/PROD deployment roles
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AssumeWorkloadDeployRoles",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::<DEV_ACCOUNT_ID>:role/CRC-AmazonConnectDeploymentRole-DEV",
        "arn:aws:iam::<UAT_ACCOUNT_ID>:role/CRC-AmazonConnectDeploymentRole-UAT",
        "arn:aws:iam::<PROD_ACCOUNT_ID>:role/CRC-AmazonConnectDeploymentRole-PROD"
      ]
    }
  ]
}

Create Roles - Deployment roles in each workload account
You will do the same steps in DEV, UAT, PROD accounts.
Role names
•	DEV: CRC-AmazonConnectDeploymentRole-DEV 
•	 UAT: CRC-AmazonConnectDeploymentRole-UAT 
•	 PROD: CRC-AmazonConnectDeploymentRole-PROD
Trust policy (trust central OIDC role)
In each workload account role, set trust:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TrustCentralOidcRole",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<SHARED_ACCOUNT_ID>:role/CRC-GitHub-OIDC-Role"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

In PROD you can also enforce approvals on GitHub side (recommended) rather than complicated IAM conditions on the workload role. GitHub explicitly recommends environment protections when using environments.

Permissions for Deployment Roles (Amazon Connect + dependencies)
 Best practice (least privilege)
AWS provides the authoritative list of:
•	Amazon Connect actions/resources/condition keys for building least privilege policies. 
AWS Verification
Check 1: OIDC provider exists in Shared account
IAM ➜ Identity providers ➜ confirm:
•	token.actions.githubusercontent.com
•	audience sts.amazonaws.com.
Check 2: Trust policy includes sub
GitHub explicitly recommends using sub to restrict assumptions.
Check 3: Cross account trust corrects.
Workload roles must trust the shared role ARN correctly.
•	GITHUB SIDE — COMPLETE SETUP
Repository prerequisites
1.	Ensure workflows can request OIDC:
o	In workflow YAML, you must set permissions: id-token: write
GitHub states this is required to request JWT. 
2.	Use GitHub Environments for DEV/UAT/PROD (recommended for enterprise) GitHub explains environment based sub and recommends adding protection rules for environments. 



Configure GitHub Environments (recommended)
In GitHub repo:
1.	Settings ➜ Environments ➜ create:
o	DEV
o	UAT
o	PROD
2.	Add protection rules:
o	PROD: required reviewers / approvals
GitHub Actions Workflow YAML
This template assumes:
•	First assume shared role (CRC-GitHub-OIDC-Role)
•	Then assume environment role (CRC-AmazonConnectDeploymentRole-DEV/UAT/PROD)
•	Then run Terraform
plan.yml (PR / branch plan)
name: terraform-plan

on:
  pull_request:
    branches: [ develop, release, main ]

permissions:
  id-token: write
  contents: read

jobs:
  plan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC -> Shared Role)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<SHARED_ACCOUNT_ID>:role/CRC-GitHub-OIDC-Role
          aws-region: <REGION>

      - name: Assume workload role (example: DEV)
        run: |
          aws sts assume-role \
            --role-arn arn:aws:iam::<DEV_ACCOUNT_ID>:role/CRC-AmazonConnectDeploymentRole-DEV \
            --role-session-name crc-plan-${{ github.run_id }} > /tmp/creds.json

          export AWS_ACCESS_KEY_ID=$(jq -r .Credentials.AccessKeyId /tmp/creds.json)
          export AWS_SECRET_ACCESS_KEY=$(jq -r .Credentials.SecretAccessKey /tmp/creds.json)
          export AWS_SESSION_TOKEN=$(jq -r .Credentials.SessionToken /tmp/creds.json)

          terraform -version
          terraform init
          terraform plan



Apply.yml (environment deployment with approvals)
name: terraform-apply
on:
  workflow_dispatch:
    inputs:
      target_env:
        description: "DEV | UAT | PROD"
        required: true
        type: choice
        options: [DEV, UAT, PROD]
permissions:
  id-token: write
  contents: read
jobs:
  apply:
    runs-on: ubuntu-latest
    environment: ${{ inputs.target_env }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC -> Shared Role)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<SHARED_ACCOUNT_ID>:role/CRC-GitHub-OIDC-Role
          aws-region: <REGION>
      - name: Assume workload role based on env
        shell: bash
        run: |
          if [ "${{ inputs.target_env }}" = "DEV" ]; then
            ROLE_ARN="arn:aws:iam::<DEV_ACCOUNT_ID>:role/CRC-AmazonConnectDeploymentRole-DEV"
          elif [ "${{ inputs.target_env }}" = "UAT" ]; then
            ROLE_ARN="arn:aws:iam::<UAT_ACCOUNT_ID>:role/CRC-AmazonConnectDeploymentRole-UAT"
          else
            ROLE_ARN="arn:aws:iam::<PROD_ACCOUNT_ID>:role/CRC-AmazonConnectDeploymentRole-PROD"
          fi

          aws sts assume-role --role-arn "$ROLE_ARN" \
            --role-session-name crc-apply-${{ inputs.target_env }}-${{ github.run_id }} > /tmp/creds.json

          export AWS_ACCESS_KEY_ID=$(jq -r .Credentials.AccessKeyId /tmp/creds.json)
          export AWS_SECRET_ACCESS_KEY=$(jq -r .Credentials.SecretAccessKey /tmp/creds.json)
          export AWS_SESSION_TOKEN=$(jq -r .Credentials.SessionToken /tmp/creds.json)

      - name: Terraform Apply
        run: |
          terraform init
          terraform apply -auto-approve




5.	Terraform Repository Directory Structure: 
 
Terraform Backend State Management
•	Each environment (Dev, UAT, Prod) uses its own S3 bucket to store Terraform state.
•	State files are isolated at the AWS account or environment level.
•	Strong isolation between environments.


State Locking Strategy
•	S3 Lock File (Recommended)
•	Terraform uses an S3 lock file mechanism to prevent concurrent state changes. 
•	No additional AWS services required.


6.	GitHub Repository Strategy – Mono Repository:
 	the repository design approach for managing Landing Zone, Shared Services, Workloads, and Integration components under the DevOps CI/CD model.
Mono Repository Approach:
In a mono repository model, all infrastructure and configuration code is maintained within a single Git repository.
•	Centralized storage of all code under one repository.
•	Single CI/CD pipeline framework.
•	Common tooling, scanners, and policies applied uniformly.

7.	Pipeline Model and Deployment Strategy:
The pipeline architecture is designed according to the following principles:
1.	Separation of duties by layer
o	Core infrastructure and LOB application infrastructure are deployed using distinct pipelines and distinct Terraform states.
2.	Environment promotion
o	Changes are promoted in order: Dev → UAT → Prod, without skipping environments.
3.	Plan-first governance
o	Any change must produce a Terraform plan that is reviewed before apply is permitted.
4.	Least-privilege access
o	GitHub Actions access AWS via OIDC with environment-scoped IAM roles and minimal permissions.
5.	Auditable approvals
o	UAT and Prod deployments are protected through environment-based approvals and reviewer gates.
6.	Deterministic execution
o	Pipelines use explicit working directories and explicit tfvars selection to avoid ambiguous deployments.
8.	Environment Promotion Strategy (Branch Based Deployment Model):
The CI/CD design follows a branch driven environment promotion model, where each Git branch is explicitly mapped to a single AWS environment. This approach ensures controlled promotion of changes from lower to higher environments while maintaining clear separation between development, testing, and production workloads.

Branch to Environment Mapping:

Git Branch	Target Environment	Use
Develop	Dev	Continuous development and frequent iterations
Release	UAT	Stabilization, functional testing, and validation
Main	Prod	Approved and production ready changes only

Promotion Flow:
•	Changes are first introduced and validated in the Dev environment through the Develop branch.
•	Stabilized changes are promoted to UAT via the Release branch for controlled testing and business validation.
•	Only fully validated and approved changes are merged into the Main branch and deployed to Production.

Promotion Path:
Develop → Dev
Release → UAT
Main → Prod





9.	CI/CD Workflow – Design and Planning Overview:
The workflow is divided into two clear stages:
•	Validation stage (CI)
•	Deployment stage (CD)

CI Workflow – Change Validation:
The CI workflow exists to validate changes early, before they are allowed to reach any environment. This stage ensures that incorrect or incomplete changes are identified during review, not during deployment.
No infrastructure is modified in this stage.
When CI Runs:
•	CI runs automatically whenever a Pull Request is raised or updated.
•	It applies equally to: 
•	Development changes.
•	UAT preparation changes.
•	Production ready changes.
•	Only validated Pull Requests can be merged
•	Invalid changes are stopped early
•	This reduces delays, rollbacks, and deployment risk later in the process

CD Workflow – Environment Deployment:
The CD workflow is responsible for executing approved changes in the correct environment.
 	This stage applies changes only after they have passed validation and review.
When CD Runs:
•	CD runs only after a change is merged.
•	The target environment is determined automatically based on the branch
•	This ensures that deployments are intentional and traceable.








10.	Overall End to End Delivery Flow:

 



11.	Runner Strategy – GitHub Actions Execution Model:
The CI/CD pipelines require runners to execute validation and deployment workflows. Based on security, network, and access requirements, the following runner model is adopted.
Runner Options: 
o	GitHub Hosted Runner.  
o	Self Hosted Runner.  
GitHub Hosted Runners (Default Choice)  
Use when:  
o	AWS APIs, terraform backend, and services are reachable via public endpoints.
o	No access needed to private VPC / internal systems.
Key points:  
o	Ephemeral VM per job.  
o	No infrastructure or OS management.
o	Secure baseline managed by GitHub.  
o	Fast setup, minimal ops.
 Self Hosted Runners (Use Only When Needed)  
Use when:  
o	Pipeline needs access to private VPC resources  
o	Access required to internal DBs, APIs, or tools  
o	Custom tools, compliance, or data locality required  
 Key points:  
o	Runs inside your infra (EC2 / VM / container / on prem)  
o	Full control over network and tools  
o	Requires OS patching, security hardening, monitoring  
o	Higher operational and security responsibility  
 Recommendation (Enterprise)  
o	Prefer GitHub hosted runners by default  
o	Use self hosted runners only if private access is mandatory  
o	Networking Considerations (Self Hosted Only)  
o	Runner should not be publicly reachable  
 

Inbound:  
o	SSH only from bastion / corporate network (if needed)  
 Outbound:  
o	HTTPS (443) to GitHub  
o	AWS APIs  
o	Required internal services only  
o	Prefer Security Group to Security Group rules (no wide CIDRs)  
Workflow Usage  
GitHub hosted:  
o	runs-on: ubuntu-latest  
Self hosted (with labels):  
o	runs-on: [self-hosted, Linux, terraform] 

12.	Destroy Workflow Policy (Controlled and Restricted Operations):
•	Destroy actions are not part of standard CI/CD pipelines and are fully separated from PR and merge workflows. 
•	Destroy operations are manually triggered only, ensuring explicit intent before execution. 
•	Access is restricted to platform or administrator roles to prevent accidental usage. 
•	Explicit approvals are required, especially for UAT and Production environments. 
•	Production resources are treated as high risk, with additional safeguards in place. 
•	Manual execution ensures full auditability and traceability.



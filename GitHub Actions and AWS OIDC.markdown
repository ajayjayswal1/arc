# Centralized CI/CD Platform with GitHub Actions and AWS OIDC

This repository provides a framework for a unified, scalable CI/CD platform leveraging GitHub Enterprise Cloud, Kubernetes (EKS), and AWS OIDC. It aims to replace traditional centralized Jenkins models with a more modern, secure, and efficient approach.

## Key Benefits

- **Security**: Eliminates the need for long-lived AWS credentials by using AWS OIDC for secure, keyless authentication.
- **Centralization**: Centralizes build and deployment logic in versioned workflows stored in a central "ci-templates" repository.
- **Scalability**: Dynamically scales runners in Kubernetes to handle varying workloads.
- **Flexibility**: Provides direct, per-account IAM roles via OIDC for fine-grained access control.

## Architecture Overview

The platform is built around the following components:

- **GitHub Enterprise Cloud**: Hosts all repositories and Actions workflows. It provides an OIDC endpoint for secure authentication.
- **Central/Service AWS Account**: Contains the ECR registry for storing container images, an EKS cluster running self-hosted runners via Actions Runner Controller (ARC), and an IAM OIDC provider that trusts GitHub’s OIDC tokens.
- **Downstream AWS Accounts**: Each account has its own EKS cluster and IAM OIDC provider. These accounts define specific deploy roles that GitHub Actions can assume for deployment tasks.
- **Reusable Workflows Repository (ci-templates)**: Contains standardized workflows for building and deploying applications, which are invoked by individual application repositories.

```
+--------------------------------------------------------------+
|                      GitHub Enterprise                       |
|  • Central “ci-templates” repo with reusable workflows       |
|  • Hundreds of App repos invoking templates via workflow_call|
+---------------------------+----------------------------------+
                            |
                            | GitHub Actions job dispatch
                            v
+---------------------------+----------------------------------+
| Central/Service AWS Account                                |
|  • ECR registry                                              |
|  • EKS cluster (ARC runners)                                 |
|  • IAM OIDC provider                                         |
+---------------------------+----------------------------------+
                            |
                            | OCI build → ECR push (AssumeRoleWithWebIdentity)
                            v
                       ECR Registry
                            |
                            | OIDC assume-role
                            v
+---------------+   +---------------+   +---------------+
| AWS Account A |   | AWS Account B | … | AWS Account N |
| • EKS Cluster |   | • EKS Cluster |   | • EKS Cluster |
| • IAM OIDC    |   | • IAM OIDC    |   | • IAM OIDC    |
+---------------+   +---------------+   +---------------+
       |                  |                    |
       | GitHub Actions   | GitHub Actions     |
       | (OIDC→EKS Deploy) | (OIDC→EKS Deploy)  |
       +------------------+--------------------+
```

## Core Components

- **GitHub Enterprise Cloud**: The central hub for all source code and CI/CD workflows. It provides the OIDC endpoint `token.actions.githubusercontent.com` for secure authentication with AWS.
- **Central/Service AWS Account (123456789012)**:
  - **ECR**: A private registry to store all container images built by the CI pipelines.
  - **EKS (ARC)**: Hosts self-hosted GitHub Actions runners managed by the Actions Runner Controller (ARC).
  - **IAM OIDC Provider**: Configured to trust GitHub’s OIDC tokens, enabling secure role assumption.
- **Downstream AWS Accounts (20+)**: Each account contains an EKS cluster for application deployments and an IAM OIDC provider. These accounts have specific IAM roles that GitHub Actions can assume for deployment tasks.
- **Reusable Workflows Repository (ci-templates)**: Contains two primary workflows:
  - `ci-build-push.yml`: For building and pushing container images to ECR.
  - `cd-deploy.yml`: For deploying applications to EKS clusters using Helm.

## Implementation Steps

1. **Enable OIDC in AWS Accounts**:
   - Run the AWS CLI command to create an OIDC provider in each AWS account:
     ```bash
     aws iam create-open-id-connect-provider \
       --url https://token.actions.githubusercontent.com \
       --client-id-list sts.amazonaws.com \
       --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
     ```

2. **Create IAM Roles**:
   - **CI Role (Central Account)**: Allows pushing images to ECR. Example ARN: `arn:aws:iam::123456789012:role/GitHubActionsCIToECRRole`.
   - **CD Roles (Downstream Accounts)**: Allows deploying to EKS clusters. Example ARN: `arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsEKSDeployRole`.

3. **Map IAM Roles to EKS RBAC**:
   - Update the `aws-auth` ConfigMap in each EKS cluster to map the deploy roles to appropriate Kubernetes groups.

4. **Deploy ARC & Runners in Central EKS**:
   - Install the Actions Runner Controller using Helm.
   - Create a service account with permissions to push to ECR.
   - Define a RunnerDeployment to manage self-hosted runners.

5. **Create Reusable Workflows**:
   - Set up the "ci-templates" repository with the `ci-build-push.yml` and `cd-deploy.yml` workflows.

6. **Invoke Workflows in Application Repos**:
   - In each application repository, create a workflow that calls the reusable workflows with appropriate parameters.

## Security & Governance

- **Zero Static Keys**: All AWS access is managed via OIDC tokens and `AssumeRoleWithWebIdentity`, eliminating the need for long-lived credentials.
- **Least Privilege**: Separate roles for CI and CD tasks with minimal permissions.
- **RBAC**: IAM roles are mapped to Kubernetes RBAC for fine-grained access control in EKS.
- **Secrets Management**: Sensitive information is stored securely in GitHub Secrets or Environments.

## Operational Considerations

- **Runner Autoscaling**: Utilize ARC’s RunnerAutoscaler and EKS NodeGroup autoscaling to dynamically adjust to workload demands.
- **Workflow Updates**: Version reusable workflows and use tools like Dependabot to manage updates.
- **Monitoring & Logging**: Centralize logs from runners and Actions for auditing and compliance.

## Next Steps

1. **POC Deployment**: Set up ARC in the central EKS cluster and test with one downstream account.
2. **Create ci-templates Repo**: Develop and test the reusable workflows.
3. **Pilot App Repos**: Convert a few sample repositories to use the new CI/CD setup.

## Reference
- Reusing workflows (GitHub Docs) Explains how to define and call reusable workflows across repos or orgs.  
  https://docs.github.com/en/actions/sharing-automations/reusing-workflows
- How to start using reusable workflows with GitHub Actions (GitHub Blog) A practical introduction with real-world examples and limitations.      
- https://github.blog/developer-skills/github/using-reusable-workflows-github-actions/
- Create reusable workflows in GitHub Actions (GitHub Resources) Step-by-step guide on inputs, secrets, and organization-wide sharing.      
- https://resources.github.com/learn/pathways/automation/intermediate/create-reusable-workflows-in-github-actions/
- Configuring OpenID Connect in Amazon Web Services (GitHub Docs) Official walkthrough for setting up the GitHub OIDC provider and trust policies in AWS.     
- https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- GitHub Actions OIDC AWS Integration: A Step-by-Step Guide (DevOpsCube) Hands-on tutorial covering OIDC setup, role policies, and deploying to EKS.    
- https://devopscube.com/github-actions-oidc-aws/
- Aws-actions/configure-aws-credentials (GitHub) The official Action for assuming roles via OIDC and configuring AWS creds in your workflow.  
- https://github.com/aws-actions/configure-aws-credentials

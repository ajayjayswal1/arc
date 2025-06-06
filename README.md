# Centralized GitHub Runners Documentation Proposal

A centralized solution for running GitHub Actions in a GitHub Enterprise environment provides a unified, secure, and scalable way to execute CI/CD workflows across multiple organizations and repositories. This document outlines two approaches to deploy self-hosted GitHub runners:

1. **Terraform AWS GitHub Runner** – Uses Terraform to provision EC2 instances managed via AWS Auto Scaling Groups.
2. **Actions Runner Controller (ARC)** – A Kubernetes-native solution that deploys ephemeral runner pods in an EKS cluster using CRDs.

---

## Table of Contents

- [Overview](#overview)
- [Approach 1: Terraform AWS GitHub Runner](#approach-1-terraform-aws-github-runner)
  - [Description](#description)
  - [Advantages](#advantages)
  - [Disadvantages](#disadvantages)
  - [Security Considerations](#security-considerations)
  - [Implementation Flow Diagram](#implementation-flow-diagram)
  - [Multi-EKS & Connectivity](#multi-eks--connectivity)
- [Approach 2: Actions Runner Controller (ARC)](#approach-2-actions-runner-controller-arc)
  - [Description](#description-1)
  - [Advantages](#advantages-1)
  - [Disadvantages](#disadvantages-1)
  - [Security Considerations](#security-considerations-1)
  - [Implementation Flow Diagram](#implementation-flow-diagram-1)
  - [Multi-EKS & Connectivity](#multi-eks--connectivity-1)
- [Comparison Summary](#comparison-summary)
- [Recommendations & Next Steps](#recommendations--next-steps)
- [Conclusion](#conclusion)
- [Architecture Diagrams](#architecture-diagrams)

---

## Overview

For GitHub Enterprise environments with multiple organizations and repositories, centralized CI/CD runners simplify operations, enhance security, and ensure consistent process adherence. The two primary approaches discussed here are:

1. **Terraform AWS GitHub Runner**  
   Uses Terraform along with AWS EC2 and Auto Scaling Groups to provision runners.

2. **Actions Runner Controller (ARC)**  
   Leverages Kubernetes-native management to deploy ephemeral runners as pods in an EKS cluster, managed via CRDs.

> **Note:** Place your architecture diagrams in the "Architecture Diagrams" section below.

---

## Approach 1: Terraform AWS GitHub Runner

### Description

This method uses the [terraform-aws-github-runner](https://github.com/github-aws-runners/terraform-aws-github-runner) module to provision and manage EC2 instances that run self-hosted GitHub runners. AWS Auto Scaling Groups (ASGs) automatically scale the number of runners based on workflow demand.

### Advantages

- **Simplicity & Maturity:**  
  - Straightforward configuration for teams familiar with Terraform and AWS.
  - Well-documented and mature Terraform module.
  
- **Broad Compatibility:**  
  - Registered as enterprise-level runners to serve multiple organizations and repositories.
  
- **Automated Scaling:**  
  - Uses AWS autoscaling mechanics to provision just the right number of runners.

### Disadvantages

- **Resource Overhead:**  
  - EC2-based runners can have longer startup times.
  - Higher cost if scaling is not properly optimized.
  
- **Cross-Account/Connectivity Complexity:**  
  - Requires additional configuration (VPC peering, Transit Gateway, or VPNs) when connecting to target resources such as private EKS clusters.

### Security Considerations

- **Authentication:**  
  - Runners are registered using Personal Access Tokens (PATs) with minimal necessary scopes.
  - Secure storage using AWS Secrets Manager or encrypted Terraform state.
  
- **Network Hardening:**  
  - Deploy runners in private subnets.
  - Use security groups and IAM policies to enforce least-privilege access.

### Implementation Flow Diagram

> **Insert Diagram 1 here**  
> Example Diagram:
> 
> ```
>              +-----------------------+
>              |  GitHub Enterprise    |
>              |  (Multiple Orgs/Repos)|
>              +----------+------------+
>                         |
>                         v
>           +-----------------------------+
>           |  GitHub Event (PR/Commit)   |
>           +------------+----------------+
>                        |
>                        v
>          +-------------------------------+
>          |  AWS API (Terraform/AWS ASG)  |
>          |   Provisions EC2 Runners      |
>          +------------+------------------+
>                        |
>                        v
>          +-------------------------------+
>          |  Runner Registers with GitHub |
>          |   via PAT & Starts CI/CD Job  |
>          +-------------------------------+
> ```

### Multi-EKS & Connectivity

- **Connectivity:**  
  - Configure EC2 runners to securely access target resources (e.g., multiple EKS clusters) via mechanisms like VPC peering.
- **Sharing Across Organizations:**  
  - By registering runners at an enterprise level and using appropriate labels, they can service multiple GitHub orgs and repositories.

---

## Approach 2: Actions Runner Controller (ARC)

### Description

ARC is a Kubernetes controller that deploys self-hosted runners as ephemeral pods in an EKS cluster. Runners are managed via Kubernetes Custom Resource Definitions (CRDs) and can autoscale rapidly based on demand.

### Advantages

- **Kubernetes-Native:**  
  - Seamlessly integrates with existing Kubernetes (EKS) environments.
  - Declarative management aligns with GitOps practices.
  
- **Dynamic & Efficient:**  
  - Faster pod startup times and efficient resource utilization.
  - Pod-level isolation improves security.
  
- **Declarative Management:**  
  - RunnerDeployments and configurations can be version-controlled as code.

### Disadvantages

- **Authentication Limitations:**  
  - ARC requires a PAT for GitHub Enterprise because GitHub App integration isn’t supported.
  
- **Complexity:**  
  - Initial setup and management of CRDs, secrets, and network policies demand Kubernetes expertise.
  - Troubleshooting connectivity or registration issues can be more involved.

### Security Considerations

- **PAT Handling:**  
  - PAT is stored as a Kubernetes secret and must be managed/rotated securely.
  - Apply strict RBAC policies within the cluster.
  
- **Pod Isolation:**  
  - Use Kubernetes network policies and namespace segmentation to secure runner pods.
  
- **Network Security:**  
  - Outbound HTTPS connections (port 443) are used; no inbound ports need to be opened.
  - Private clusters use NAT gateways or VPC endpoints, allowing secure access to GitHub endpoints.

### Implementation Flow Diagram

> **Insert Diagram 2 here**  
> Example Diagram:
> 
> ```
>             +-----------------------+
>             |  GitHub Enterprise    |
>             |  (Multiple Orgs/Repos)|
>             +----------+------------+
>                        |
>                        v
>           +-----------------------------+
>           | GitHub Event Triggers ARC   |
>           |  (via Repo's Workflow YAML) |
>           +-------------+---------------+
>                         |
>                         v
>          +---------------------------------+
>          |   ARC in EKS (Dev Environment)  |
>          |   Deploys Runner Pods via CRDs  |
>          +-------------+-------------------+
>                        |
>                        v
>          +---------------------------------+
>          |  Runner Pod Registers with GitHub|
>          |   via PAT & Executes CI/CD Job  |
>          +---------------------------------+
> ```

### Multi-EKS & Connectivity

- **Private Subnets:**  
  - ARC can run fully in private EKS clusters provided outbound connectivity via NAT gateways or VPC endpoints.
- **Cross-Cluster Use:**  
  - ARC deployments can be extended to multiple EKS clusters if desired.
- **Sharing Across Organizations:**  
  - Centralized ARC deployment can serve multiple GitHub organizations using appropriate labels and runner group configurations.

---

## Comparison Summary

| **Criteria**                           | **Terraform AWS GitHub Runner**                                                  | **Actions Runner Controller (ARC)**                                         |
| -------------------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **Sharing Across Orgs**                | Enterprise-level runner registration; supports multiple repos/orgs.             | Centralized ARC deployment with runner labels & runner groups.              |
| **Deployment Environment**             | EC2-based instances provisioned via Terraform & ASGs.                           | Kubernetes pods in EKS; ephemeral and dynamic with CRD management.          |
| **Security & Compliance**              | PAT-based registration, AWS IAM/network hardening, manageable with familiar AWS tooling. | PAT-based registration stored as K8s secrets; benefits from pod isolation & advanced network policies. |
| **Scalability/Autoscaling**            | Uses AWS ASGs; relatively slower instance spin-up times.                        | Leverages Kubernetes autoscaling (HPA) for rapid, efficient scaling.        |
| **Networking & Connectivity**          | Configure using VPC peering, NAT, or VPN for cross-account/EKS connectivity.      | Outbound-only HTTPS connectivity without the need for inbound port exposure.|
| **Cost & Complexity**                  | Easier for AWS-centric operations; cost may be higher if not optimized.           | Lower operational overhead if Kubernetes-centric; higher initial setup complexity. |

**Industry Trends:**  
- **Terraform AWS Runner:** Popular among teams with an AWS-centric view and familiarity with Terraform.
- **ARC Approach:** Increasingly adopted by organizations embracing Kubernetes and GitOps practices due to its dynamic scaling and integrated container orchestration.

---

## Recommendations & Next Steps

1. **Pilot Deployment:**  
   - Choose a subset of repositories to test either approach.
   - Validate runner registration, job execution, and secure connectivity, especially from private subnets.

2. **Security & Compliance:**  
   - Secure and rotate PATs using AWS Secrets Manager or Kubernetes secrets.
   - Implement robust monitoring of outbound connectivity and network activity.

3. **Documentation & Rollout Plan:**  
   - Finalize runbooks outlining deployment, scaling, troubleshooting, and credential management.
   - Populate the architecture diagrams section with detailed visuals.

---

## Conclusion

Both the **Terraform AWS GitHub Runner** and **Actions Runner Controller (ARC)** approaches provide robust means to centralize CI/CD runners in a GitHub Enterprise environment. The choice depends on your organization's existing infrastructure, expertise, and strategic goals:

- **Terraform AWS GitHub Runner** is ideal for AWS-centric teams leveraging mature infrastructure-as-code practices.
- **ARC** is preferable for organizations standardized on Kubernetes, offering dynamic scalability and enhanced integration with GitOps methodologies.

This document serves as a comprehensive framework for evaluating and implementing a centralized GitHub runner solution.

---

## Architecture Diagrams

- **Diagram 1:** [Insert Terraform AWS GitHub Runner architecture diagram-https://github-aws-runners.github.io/terraform-aws-github-runner/assets/aws-architecture.light.png#only-light]
- **Diagram 2:** [ARC-based runner architecture diagram-https://media.githubusercontent.com/media/actions/actions-runner-controller/master/docs/gha-runner-scale-set-controller/arc-diagram-light.png]

---

*Prepared for internal review and iterative development. Contributions and suggestions for improvements are welcome.*


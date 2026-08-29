# Purpose: Migrate an existing Jenkins-based CI/CD pipeline to GitHub Actions, for a Node.js application deployed to an EC2 instance.

# Architecture  
    GitHub push/PR → GitHub Actions → AWS (via OIDC) → ECR → EC2 (deployed via SSM)
    
# Key Design Choices (vs. Jenkins)  
No static AWS credentials — authentication to AWS uses OIDC (OpenID Connect) federation, so GitHub Actions gets short-lived, auto-expiring credentials instead of long-lived IAM access keys stored as secrets.
No SSH to EC2 — deployment uses AWS Systems Manager (SSM) send-command to remotely execute Docker commands on the EC2 instance, removing the need for open SSH ports or SSH key management (which Jenkins typically relied on).

# Workflow: CI-CD (.github/workflows/ci-cd.yml)

**Triggers:** push or pull request to main

# Job 1 — build-test-push

- Checkout repo  
- Set up Node.js 22  
- Install dependencies (npm install in app/)  
- Run tests (npm test)  
- Authenticate to AWS via OIDC — assumes GitHubActionsMigrationRole  
- Verify identity (aws sts get-caller-identity)  
- Log in to Amazon ECR  
- Build Docker image, tagged with the Git commit SHA  
- Push image to ECR (jenkins-migration-demo repo)  

# Job 2 — deploy (runs after Job 1 succeeds)

- Authenticate to AWS via OIDC (same role)  
- Deploy via SSM send-command — remotely runs shell commands on the EC2 instance to:  
- Log Docker into ECR  
- Pull the new image  
- Stop/remove the old container  
- Run the new container (port 8082:8080)  
- Wait for the SSM command to complete  
- Run a smoke test — curl the /health endpoint on the EC2 public IP

#  Create the OIDC Identity Provider

In AWS Console: IAM → Identity providers → Add provider

Field	Value  
- Provider type	**OpenID Connect**    
- Provider URL	**https://token.actions.githubusercontent.com**  
- Audience	**sts.amazonaws.com**  

This creates:

     arn:aws:iam::289835835122:oidc-provider/token.actions.githubusercontent.com

<img width="789" height="483" alt="image" src="https://github.com/user-attachments/assets/904395ca-62f9-4681-8757-66d49e889cb5" />

# GitHub Actions OIDC Role — Setup Steps

This is the role your GitHub Actions workflow assumes (GitHubActionsMigrationRole) to get temporary AWS credentials. It needs both ECR permissions (build-test-push job) and SSM permissions (deploy job), since you're attaching everything to one role.

<img width="735" height="439" alt="image" src="https://github.com/user-attachments/assets/5117510e-8fee-4652-b695-4ddb087ba280" />

<img width="1781" height="776" alt="image" src="https://github.com/user-attachments/assets/2efb3387-7d25-4f7d-b3cd-e7fd08a9d3ad" />

<img width="920" height="327" alt="image" src="https://github.com/user-attachments/assets/e638caea-c862-4c67-bdc8-61ae5603a93c" />

# Trust Policy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::289835835122:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:vamshireddy903@204807918/cicd-migration@1346798147:ref:refs/heads/main"
                }
            }
        }
    ]
}
```
<img width="695" height="419" alt="image" src="https://github.com/user-attachments/assets/0fb74e1d-eb7d-4f3d-b45e-815be767fe0d" />

Also add AmazonEC2ContainerRegistryPowerUser permission and cerate OIDC role

# Create EC2 role

<img width="808" height="648" alt="image" src="https://github.com/user-attachments/assets/6575a23e-dfd3-42c7-bf9d-5c17a4082bda" />

# Attach the Role to the EC2 Instance

Since IAM roles attach to EC2 via an instance profile:

Console: EC2 → Instances → select instance → Actions → Security → Modify IAM role → choose EC2-CICD-InstanceRole → Update IAM role 

# Add Repository Variables in GitHub  

GitHub repo → Settings → Secrets and variables → Actions → Variables tab → New repository variable

<img width="717" height="176" alt="image" src="https://github.com/user-attachments/assets/6d5c6bfc-ca68-4a96-8002-1a298639cd55" />


<img width="846" height="648" alt="image" src="https://github.com/user-attachments/assets/095a944c-6dab-4fa0-8c5e-82529597e8bc" />


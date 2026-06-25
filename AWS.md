# Appendix AWS setup checklist (step by step)

The following checklist is a step-by-step guide to configure AWS for the OPA Workloads GitHub Actions workflow, in order to enable the optional AWS Security Group auto-update feature. It assumes you have an AWS account and the necessary permissions to create IAM roles, policies, and security groups.

1. Create or identify the target Security Group
   - Go to `EC2 > Network & Security > Security Groups`.
   - Create or select the Security Group that must be updated temporarily during the workflow run.
   - Record:
     - Security Group ID (placeholder example: `sg-0a1bc23d456e7890f`)
     - AWS region (example: `us-east-1`)

2. Attach the Security Group to the target EC2 instance
   - Go to `EC2 > Instances`, select the target VM, then open `Security > Security groups > Edit`.
   - Add the Security Group from step 1 to the instance (or replace the current one, depending on your design).
   - Save and verify that the instance (or its ENI) is using that Security Group.
   - Ensure the workflow port is aligned with your service exposure:
     - default is `7234`
     - or set `AWS_SECURITY_GROUP_PORT` in GitHub Actions variables

3. Create (or verify) the GitHub OIDC identity provider in IAM
   - Go to `IAM > Identity providers > Add provider`.
   - Provider type: `OpenID Connect`
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
   - If it already exists, reuse it.

4. Create a minimal IAM policy for only that Security Group
   - Go to `IAM > Policies > Create policy > JSON`.
   - Use a least-privilege policy like this:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "ec2:AuthorizeSecurityGroupIngress",
            "ec2:RevokeSecurityGroupIngress",
            "ec2:CreateTags"
          ],
          "Resource": [
            "arn:aws:ec2:<region>:<account-id>:security-group/<security-group-id>",
            "arn:aws:ec2:<region>:<account-id>:security-group-rule/*"
          ]
        }
      ]
    }
    ```

   - Concrete placeholder example:

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "ec2:AuthorizeSecurityGroupIngress",
            "ec2:RevokeSecurityGroupIngress",
            "ec2:CreateTags"
          ],
          "Resource": [
            "arn:aws:ec2:us-east-1:123456789012:security-group/sg-0a1bc23d456e7890f",
            "arn:aws:ec2:us-east-1:123456789012:security-group-rule/*"
          ]
        }
      ]
    }
    ```

5. Create an IAM role for GitHub Actions (not EC2)
   - Go to `IAM > Roles > Create role`.
   - Trusted entity type: `Web identity`
   - Identity provider: `token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
  - Attach the policy from step 4.
   - Example role ARN format:
     - `arn:aws:iam::123456789012:role/GitHubActionsRoleForOPAGateway`

6. Restrict the role trust policy to your repository
   - In the role, open `Trust relationships > Edit trust policy`.
   - Restrict by `aud` and `sub` (repo and optionally branch/environment).
   - Example:

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
          },
          "Action": "sts:AssumeRoleWithWebIdentity",
          "Condition": {
            "StringEquals": {
              "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
              "token.actions.githubusercontent.com:sub": "repo:xyz/okta-lab-workloads:ref:refs/heads/main"
            }
          }
        }
      ]
    }
    ```

7. Configure GitHub Actions variables in the repository
   - Go to `Settings > Secrets and variables > Actions > Variables`.
   - Required for AWS mode:
     - `AWS_GROUP_ID=sg-0a1bc23d456e7890f`
     - `AWS_ROLE_TO_ASSUME=arn:aws:iam::123456789012:role/GitHubActionsRoleForOPAGateway`
   - Optional:
     - `AWS_REGION=us-east-1`
     - `AWS_SECURITY_GROUP_PORT=7234`

8. Verify workflow permissions and run
   - Ensure the workflow keeps:
     - `permissions.id-token: write`
     - `permissions.contents: read`
   - Run `OPA Workloads - SSH Linux` and confirm these steps appear:
     - `Configure AWS credentials for temporary Security Group access`
     - `Allow current runner IP in AWS Security Group`
     - `Run SSH test`
     - `Remove current runner IP from AWS Security Group`

If your AWS policy validates request tags or rule tags, the workflow tags newly created Security Group rules with:

- `CreatedBy=GitHubActions`
- `Repository=<owner>/<repo>`
- `GitHubRunId=<run-id>`
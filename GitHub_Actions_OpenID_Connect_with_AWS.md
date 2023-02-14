					#===============================================#
					# How to setup OpenID Connect in Github and AWS #
					#===============================================#

Maintainer: Lior Dux.
Date: 14-02-2023.
Last Update: 14-02-2023.

## Step 1: Add GitHub as an Identity Provider:
----------------------------------------------
- Select OpenID Connect
- For the provider URL: Use https://token.actions.githubusercontent.com
- Click on 'Get thumbprint'
- For the "Audience": Use sts.amazonaws.com if you are using the official action
- Tag as you wish

## Step 2: Create an Role with the relevent access required for you liking.
---------------------------------------------------------------------------
```GitHub-Actions-OIDC-Policy
{    //# Example of a Role 'my-github-actions-role-test' with attached policy (with access to ECR and ECS):
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:CompleteLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:InitiateLayerUpload",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage"
            ],
            "Resource": "arn:aws:ecr:eu-west-1:111111111111:repository/test"
        },
        {
            "Effect": "Allow",
            "Action": "ecr:GetAuthorizationToken",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ecs:RegisterTaskDefinition",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ecs:UpdateService",
                "ecs:DescribeServices"
            ],
            "Resource": "arn:aws:ecs:eu-west-1:111111111111:service/*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::111111111111:role/ecsTaskExecutionRole"
        },
        {
            "Effect": "Allow",
            "Action": "sts:TagSession",
            "Resource": [
                "arn:aws:iam::111111111111:role/ecsTaskExecutionRole"
            ]
        }
    ]
}
```

## Step 3: Edit the Trust Policy of your Role for a better match with JWT:
--------------------------------------------------------------------------
```
{   //# Example (all branches in my repo):
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:sub": "repo:<GithubOrgAccount>/<MyRepo>:*"
                }
            }
        }
    ]
}
```

## Step 4: Edit your workflow:
------------------------------

#### Add the plugin to your workflow:
----------------------------------
```permissions.yml
# permission can be added at job level or workflow level    
permissions:
  id-token: write # This is required for requesting the JWT (Json Web Token)
  contents: read  # This is required for actions/checkout
```

#### Add the plugin to your workflow:
----------------------------------
```plugin.yml
    - name: Configure AWS credentials from Test account
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::111111111111:role/my-github-actions-role-test
        aws-region: us-east-1
```

#### Example of workflow integration:
----------------------------------
```oidc-workflow.yml
deploy:
    name: Upload to Amazon S3
    runs-on: ubuntu-latest
    
    # These permissions are needed to interact with GitHub's OIDC Token endpoint.
    permissions:
      id-token: write
      contents: read
    
    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Configure AWS credentials from Test account
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::111111111111:role/my-github-actions-role-test
        role-session-name: GitHub-Actions-OIDC
        aws-region: eu-west-1
```
## Step 5: All Done!
--------------------
:party:

## Info, links, etc
-------------------
(GitHub OpenID Connect Docs)[https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services]
(AWS Configure Plugin Github Repo)[https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions]
(Youtube Tutorial)[https://www.youtube.com/watch?v=k2Tv-EJl7V4]

## Maintainer notes:
--------------------
1. Using roles is best practice. 
2. Decoupling rules privilages is even better practice.
3. Can have multiple calls to aws configure plugin for different roles with different abilities.
4. Token expiry can be set using 'role-duration-seconds: 3600'.
5. According to plugin documentation, default session duration is 1h, and could be extended to 6h using a access-key combo.
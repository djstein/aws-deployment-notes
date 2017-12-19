# AWS Deployment Notes

This document is currently a set of notes to be rearranged into steps for deploying a modern AWS website.

The goal is to have a standard for:
- creating Docker images which will be used for local development
- building Docker images that will be stored on ECS/EKS for testing, distrubution and production
- deploying and serving a static frontend website, bundles, and static files from S3
- routing at a DNS level using Route 53 to ALB or S3 respectively
- usage of Docker Compose along with Docker images stored on ECS/EKS to create a Fargate system
- balancing the Fargate system with ALB
- hosting a test, replication, and production database on RDS
- storing backups of S3 and RDS files to Glacier
- using CloudWatch and ALB to monitor the health of instances, databases, etc
- storing backend, frontend, and deployment code on private GitHub repositories
- use of LetsEncrypt to create SSL/TLS certificates and use with Route 53
- host a Docker based Jenkins system to test changes to repositories (also routed with Route 53)
- automatically rebuild and deploy system based on frontend, backend, or image changes
- have entirely seperate / replicated system as a staging environment

Hosting a static website on S3 with Route 53 routing:
https://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html

Routing ALB with Route 53
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html

Creating an ALB instance
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancer-getting-started.html

Using ECS Compose for Docker Compose Files
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html

## Development Process
- git clone backend
- git clone frontend
- git clone deployment
- docker-compose up --build deployment/local/
- edit code
- push to github

## CLI Process Frontend:
- git push frontend
- jenkins webhook
- build with new hash and run jest tests
- push bundle to S3 test bucket
- build backend (test variant) and run integration tests against bundle
- if 100% test pass, place dist folder in production S3
- bust index.html hash, it contains new hashed bundle url

## CLI Process Backend:
- git push backend
- jenkins webhook
- build new docker images for backend processes
- run unit tests against new docker images
- run integration tests with latest bundle against new docker images
- if 100% test pass, push new images to ECS/EKS
- ECS/EKS will auto deploy changes

## URL Structure
There will be one domain name with four subdomains. Requests to these URLs will be handled with Route 53
- example.com -> loads static files from S3 bucket. www.example.com is redirected to example.com
- api.example.com -> used for backend services on ALB
- staging.example.com -> loads static files from S3 bucket for staging environment
- stagingapi.example.com -> used for the staging version of backend services on ALB
- jenkins.example.com -> loads Jenkins page which is handled by ALB


## Backend Requirements
- Python 3
- use Django 2 + DRF
- register user via API
- send registration email to user
- login user only with email
- use JWT for verification of user
- logout via API if JWT present
- allow for password changes via API
- allow for user model updates
- allow for email preference updates
- allow for attached user settings
- save and update Stripe token per user
- send emails
- send tasks to workers
- upload photos / files to S3
- connect to test DB
- connect to prod / staging DB
- when local and debug, serve static files from Django
- when test and debug, serve static files from S3
- when prod, serve static files from S3
- environment variables stored on .env for local
- environment variables stored on AWS for test, staging, prod
- unit Tests for all views, models, serializers
- integration Tests for workflows with frontend

## Frontend Requirements
- use React 16 + ES6+
- use React Router, Redux, Redux Router
- use Babel Transpile
- build with new assets hash
- no css, all styles as styled components
- static files are placed on S3 with build dist folder (not saved in bundle)
- handle storing JWT token for authentication of requests (bypass CSRF Token)
- provide login, logut, registration flows
- provide forms for user info, settings, email preferences, stripe additions as settings
- provide theme through styled components
- tests for al components with Jest + Enzyme

# Setup VPC
- setup VPC (http://docs.aws.amazon.com/AmazonVPC/latest/GettingStartedGuide/getting-started-ipv4.html)

# Setup AWS CLI
(http://docs.aws.amazon.com/cli/latest/userguide/installing.html) (http://docs.aws.amazon.com/cli/latest/userguide/cli-environment.html)
- `pip install awscli`
- create access key: https://console.aws.amazon.com/iam/home?region=us-east-1#/users/djstein?section=security_credentials
- put access key into config
```
$ aws configure
AWS Access Key ID [None]: AWS_ACCESS_KEY_ID
AWS Secret Access Key [None]: AWS_SECRET_ACCESS_KEY
Default region name [None]: AWS_DEFAULT_REGION
Default output format [None]: json
```
- check user: `aws iam get-user`

# MFA
https://aws.amazon.com/premiumsupport/knowledge-center/authenticate-mfa-cli/
https://github.com/aws/amazon-ecs-cli/issues/284
arn-of-the-mfa-device: https://console.aws.amazon.com/iam/home?region=<region>#/users/<username>?section=security_credentials
code-from-token: from device
```aws sts get-session-token --serial-number <arn-of-the-mfa-device> --token-code <code-from-token>```
replace values in .aws/credentials with values from output:
```
[profile-name]
output = json
region = us-east-1
aws_access_key_id = <Access-key-as-in-returned-output>
aws_secret_access_key = <Secret-access-key-as-in-returned-output>
aws_session_token = <Session-Token-as-in-returned-output>
```
Either set the values in above .aws/creds or as environment variables

# Begin ECS CLI setup
```aws iam --region us-east-2 create-role --role-name ecsExecutionRole --assume-role-policy-document file://execution-assume-role.json
aws iam --region us-east-2 attach-role-policy --role-name ecsExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```
- install ecs cli:
```
sudo curl -o /usr/local/bin/ecs-cli https://s3.amazonaws.com/amazon-ecs-cli/ecs-cli-darwin-amd64-latest
sudo chmod +x /usr/local/bin/ecs-cli
ecs-cli --version
```
at .aws create file execution-assume-role.json
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
run
```
aws iam --region us-east-1 create-role --role-name ecsExecutionRole --assume-role-policy-document file://execution-assume-role.json
aws iam --region us-east-1 attach-role-policy --role-name ecsExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```
create cluster
```
ecs-cli configure --cluster tutorial --region us-east-1 --default-launch-type FARGATE --config-name tutorial
```
create ecs profile
```
ecs-cli configure profile --access-key $AWS_ACCESS_KEY_ID --secret-key $AWS_SECRET_ACCESS_KEY --profile-name  tutorial
```
run up command. if MFA used, ensure account details are in environemt variables or tell ecs-cli to use profile
```
ecs-cli up   - or -  ecs-cli up --aws-profile <name>
```
Save the subnets returned for  ecs-params.yml

Using the AWS CLI create a security group using the VPC ID from the previous output: 
```
aws ec2 create-security-group --group-name "my-sg" --description "My security group" --vpc-id "VPC_ID"
```
Save this for the ecs-params.yml

Using AWS CLI, add a security group rule to allow inbound access on port 80:
```
aws ec2 authorize-security-group-ingress --group-id "security_group_id" --protocol tcp --port 80 --cidr 0.0.0.0/0
```

Create a version 2 docker-compose.yml file and populate it

Create a ecs-params.yml file and populate it with:
```
version: 1
task_definition:
  task_execution_role: ecsExecutionRole
  ecs_network_mode: awsvpc
  task_size:
    mem_limit: 0.5GB
    cpu_limit: 256
run_params:
  network_configuration:
    awsvpc_configuration:
      subnets:
        - "subnet ID 1"
        - "subnet ID 2"
      security_groups:
        - "security group ID"
      assign_public_ip: ENABLED
```

once configured both files run:
```ecs-cli compose --project-name tutorial service up```

check on the containers status with:
```ecs-cli compose --project-name tutorial service ps```

get the ip address from ps and view

clean up with
```
ecs-cli compose --project-name tutorial service down
ecs-cli down --force
```

# Pushing Docker Images to AWS ECS
start by getting the login command and token
```aws ecr get-login --no-include-email --region us-east-1 ```
copy and paste the result to login
move to the directory of docker build the image you wish to push
```docker build -t <org>/<image> .```
tag your image
```docker tag <org>/<image>:latest <vals>.<location>.amazonaws.com/<org>/<image>:latest```
after authenticating push the image
```docker push <vals>.<location>.amazonaws.com/<org>/<image>:latest```

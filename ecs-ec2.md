# Notes for deploying ECS on EC2 (instead of Fargate)

Cluster Config
```
ecs-cli configure --cluster <project_name> --region us-east-1 --default-launch-type EC2 --config-name <project_name>
```

Add Credentials
```
ecs-cli configure profile --access-key ${AWS_ACCESS_KEY_ID} --secret-key ${AWS_SECRET_ACCESS_KEY} --profile-name  <project_name>
```

Set up id_rsa https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair
after creating the keys be sure to do the following:
```
chmod 400 <name>.pem
ssh-add <name>.pen
```

Create instances
```
ecs-cli up --keypair id_rsa --capability-iam --size 2 --instance-type t2.medium

# if using a non default config and mfa
ecs-cli up --keypair id_rsa --capability-iam --size 2 --instance-type t2.medium --cluster <project> --aws-profile mfa

# if with vpc, subnets, and force to shut down exisiting cluster
ecs-cli up --keypair ~/.ssh/id_rsa --capability-iam --size 2 --instance-type t2.medium --vpc vpc-<val> --subnets subnet-<val>,subnet-<val> --aws-profile mfa --force
```

Compose Up
```
ecs-cli compose up --create-log-groups --cluster <project> --aws-profile mfa
```

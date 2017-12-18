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

Hosting a static website on S3 with Route 53 routing:
https://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html

Routing ALB with Route 53
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html

Creating an ALB instance
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancer-getting-started.html

Using ECS Compose for Docker Compose Files
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html

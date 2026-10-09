# aws-cloudformation-templates

Infrastructure as Code with AWS CloudFormation: modular templates for networking, compute, databases, CI/CD pipelines, CDN and observability.

The templates are designed for API/web workloads. A main stack creates the network, load balancer, Auto Scaling Group, database and deployment pipelines. Smaller companion stacks add caching, alarms, CDN, email logging and video processing, and read the main stack's outputs through `Fn::ImportValue`.

## Templates

| Template | Purpose | Depends on main stack |
|---|---|---|
| `templates/appstack.yml` | Main stack: VPC, subnets, ALB, EC2 Auto Scaling Group, RDS MySQL (KMS-encrypted), EC2 Image Builder AMI pipeline, CodeBuild/CodeDeploy/CodePipeline for API, CMS and web, VPC flow logs, CloudWatch agent config | n/a |
| `templates/redis.yml` | ElastiCache Redis replication group with encryption at rest and in transit | Yes |
| `templates/cloudwatch-alarms.yml` | SNS topic with email/HTTPS subscribers and CPU, disk and memory alarms for the ASG | Yes |
| `templates/cloudfront.yml` | S3 asset bucket with Origin Access Control, plus CloudFront distributions for API, CMS and web in front of the ALB | Needs ALB values |
| `templates/ses-logging.yml` | SES identities, SNS topic and a Lambda function that writes SES events to CloudWatch Logs | Optional |
| `templates/mediaconvert-vod.yml` | Video-on-demand foundation: S3 buckets, MediaConvert job template, CloudFront delivery, EventBridge status webhook (based on the AWS "Video on Demand on AWS Foundation" solution) | No |
| `templates/landing-page-pipeline.yml` | Standalone build and deploy pipeline for a static landing page | No |

## Architecture overview

```
Internet
   │
CloudFront ─────────► ALB (public subnets, 2 AZs)
                         │
                  EC2 Auto Scaling Group  ◄── CodePipeline → CodeBuild → CodeDeploy
                         │
        ┌────────────────┴───────────────┐
   RDS MySQL (private)           ElastiCache Redis (private)

CloudWatch agent → logs + metrics → alarms → SNS (email / HTTPS)
VPC flow logs → CloudWatch Logs
```

## Prerequisites

- An AWS account and the AWS CLI v2 configured (`aws configure sso` or an IAM profile)
- Permissions to create IAM roles, VPC, EC2, RDS, S3, CodePipeline and CloudFront resources
- `cfn-lint` (optional, recommended): `pip install cfn-lint`

## Deployment order

1. **Main stack first.** Its outputs (VPC ID, subnets, Auto Scaling Group) are imported by the other stacks.
2. Deploy the companion stacks, passing the main stack's name as `ParentStack`.

### 1. Main stack

```bash
aws cloudformation deploy \
  --template-file templates/appstack.yml \
  --stack-name myproject-dev \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      environment=dev \
      ProjectName=myproject \
      Region=ap-southeast-2
```

Auto Scaling `MinSize`, `MaxSize` and `DesiredCapacity` default to `0`, so no EC2 instances launch until you raise them. This keeps a first deployment cheap.

### 2. Redis

```bash
aws cloudformation deploy \
  --template-file templates/redis.yml \
  --stack-name myproject-dev-redis \
  --parameter-overrides ParentStack=myproject-dev ProjectName=myproject environment=dev
```

### 3. CloudWatch alarms

```bash
aws cloudformation deploy \
  --template-file templates/cloudwatch-alarms.yml \
  --stack-name myproject-dev-alarms \
  --parameter-overrides \
      ParentStack=myproject-dev ProjectName=myproject environment=dev \
      EmailAdd1=you@example.com EmailAdd2=team@example.com \
      HttpsEndpnt=https://example.com/alerts
```

### 4. CloudFront

Needs the ALB DNS name and ID:

```bash
aws elbv2 describe-load-balancers --names myproject-dev-alb \
  --query 'LoadBalancers[0].[DNSName,LoadBalancerArn]' --output text
```

```bash
aws cloudformation deploy \
  --template-file templates/cloudfront.yml \
  --stack-name myproject-dev-cdn \
  --parameter-overrides \
      ParentStack=myproject-dev ProjectName=myproject environment=dev \
      ALBDomain=<alb-dns-name> ALBId=<alb-arn>
```

### 5. Optional stacks

```bash
# SES logging
aws cloudformation deploy --template-file templates/ses-logging.yml \
  --stack-name myproject-dev-ses --capabilities CAPABILITY_IAM \
  --parameter-overrides ParentStack=myproject-dev ProjectName=myproject environment=dev \
      ExternalEmailIdentity=you@example.com ExternalDomainIdentity=example.com

# MediaConvert VOD
aws cloudformation deploy --template-file templates/mediaconvert-vod.yml \
  --stack-name myproject-dev-vod --capabilities CAPABILITY_IAM \
  --parameter-overrides WebhookAuthorizationUrl=https://example.com/hook \
      WebhookAuthorizationKey=<secret>
```

## Parameters you will change most

| Parameter | Description | Default |
|---|---|---|
| `environment` | `dev`, `staging` or `prod` | `dev` |
| `ProjectName` | Lowercase, unique, no special characters, 32 chars max. Used in resource names | `projectName` |
| `Region` | Target region (used for subnet AZ placement) | `ap-southeast-2` |
| `VPCCidr` and subnet CIDRs | Network layout (1 VPC, 2 public, 2 private subnets) | `10.0.0.0/16` and /24s |
| `InstanceType`, `ImageId` | Web server size and base AMI | `t4g.small`, Ubuntu arm64 |
| `RDSInstance` | Database instance class | `db.t3.micro` |
| `*RepositoryUrl` | Source repositories for the API, CMS and web builds | placeholder |

## Security notes

- **Never commit secrets.** Do not put access tokens in repository URLs or passwords in parameter defaults. Use Secrets Manager (`{{resolve:secretsmanager:...}}`), `NoEcho` parameters, a CodeStar connection for source access, and `ManageMasterUserPassword` for RDS.
- Replace placeholder values (`example.com`, email addresses, repository URLs) before deploying.
- Restrict SSH ingress to your own IP range, or use SSM Session Manager and remove the SSH rule.
- Review IAM roles and scope them down to least privilege before using any stack outside a sandbox.

## Cost and cleanup

Several resources are billed while they exist: NAT-less VPC resources are free, but the ALB, RDS, ElastiCache, CloudFront, Image Builder runs and EC2 instances are not. Set an AWS Budget alert first and delete stacks when you are done, in reverse order of creation:

```bash
aws cloudformation delete-stack --stack-name myproject-dev-cdn
aws cloudformation delete-stack --stack-name myproject-dev-alarms
aws cloudformation delete-stack --stack-name myproject-dev-redis
aws cloudformation delete-stack --stack-name myproject-dev
```

S3 buckets must be emptied before their stack can be deleted.

## Validate templates

```bash
cfn-lint templates/*.yml
```

## Roadmap

- [ ] GitHub Actions workflow running `cfn-lint` and Checkov on every push
- [ ] Replace broad IAM managed policies with least-privilege policies
- [ ] Use AWS-managed secrets for the database and Redis auth token
- [ ] Export the ALB DNS name and ARN from the main stack so the CloudFront stack can import them
- [ ] Architecture diagram

## License

MIT

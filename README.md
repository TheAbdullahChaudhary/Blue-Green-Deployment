# Implementing Blue-Green Deployments for Dockerized Flask Applications on AWS

## Introduction

In today's fast-paced digital landscape, ensuring zero-downtime deployments is crucial for maintaining user satisfaction and business continuity. After extensively working with AWS infrastructure for containerized applications, I'm excited to share my implementation of a blue-green deployment strategy for a dockerized Flask application using AWS services.

## The Challenge

Traditional deployment methods often involve some degree of downtime during updates, which can impact user experience and potentially lead to lost revenue. For our Flask application, we needed a robust solution that would:

1. Allow updates without service interruption
2. Provide quick rollback capability if issues arose
3. Enable proper testing of new versions in a production-like environment
4. Maintain high availability and scalability

## The Solution: Blue-Green Deployment on AWS

Our architecture leverages several AWS services to create a seamless blue-green deployment pipeline:

- **Virtual Private Cloud (VPC)**: Provides network isolation and security
  - Multiple Availability Zones for high availability
  - Public subnets for ALB, private subnets for EC2 instances
  - CIDR block: 10.0.0.0/16 with /24 subnet divisions
  - Custom route tables for secure traffic flow

- **Application Load Balancer (ALB)**: Routes traffic to the appropriate environment
  - Internet-facing configuration
  - Cross-zone load balancing enabled
  - Idle timeout set to 60 seconds
  - HTTP to HTTPS redirection rules

- **Auto Scaling Groups**: Manages EC2 instance fleets for both blue and green environments
  - Desired capacity: 2 (minimum: 2, maximum: 6)
  - Scale-out policy: CPU utilization > 70%
  - Scale-in policy: CPU utilization < 30%
  - Cooldown period: 300 seconds
  - Termination policy: Default (oldest instance first)

- **Target Groups**: Registers and monitors instance health for each environment
  - Protocol: HTTP
  - Port: 80
  - Health check path: /health
  - Health check interval: 30 seconds
  - Health check timeout: 5 seconds
  - Healthy threshold: 2 consecutive checks
  - Unhealthy threshold: 3 consecutive checks
  - Deregistration delay: 60 seconds

- **Launch Templates**: Defines EC2 configuration and bootstrap script
  - AMI: Ubuntu Server 22.04 LTS
  - Instance type: t3.small
  - EBS volumes: 8 GB gp3 (3000 IOPS)
  - Detailed monitoring enabled
  - IMDSv2 required

- **Amazon ECR**: Stores our Docker images securely
  - Image scanning on push
  - Tag immutability enabled
  - Lifecycle policy: Keep last 5 images

- **IAM Roles**: Grants EC2 instances permission to pull from ECR
  - AmazonEC2ContainerRegistryReadOnly managed policy
  - CloudWatch agent policy for metrics
  - Principle of least privilege applied

### Deployment Architecture Overview

In our blue-green deployment setup:

1. We maintain two separate environments (blue and green)
2. Only one environment serves production traffic at any given time
3. Updates are deployed to the inactive environment
4. After testing confirms the new environment is stable, we switch traffic by updating the ALB's target group

This approach allows us to validate new versions in a production-like environment before exposing them to users, with the ability to instantly revert to the previous stable version if needed.

## Launch Template Configuration

The heart of our deployment strategy lies in properly configured Launch Templates. Here's the bootstrap script we use to initialize EC2 instances in our environment:

```bash
#!/bin/bash
sudo su
apt update -y
apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update -y
apt install -y docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker

# AWS ECR Login
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 421983921734.dkr.ecr.us-west-2.amazonaws.com

# Pull and run the image
docker pull 421983921734.dkr.ecr.us-west-2.amazonaws.com/flask-app:latest
docker run -d -p 80:80 421983921734.dkr.ecr.us-west-2.amazonaws.com/flask-app:latest
```

This script:
1. Updates system packages
2. Installs Docker and its dependencies
3. Configures Docker to start automatically
4. Authenticates with Amazon ECR using the instance's IAM role
5. Pulls our Flask application Docker image
6. Runs the container, mapping port 80

## The Deployment Process

Our deployment workflow consists of these steps:

### 1. Preparation Phase
- Push the new Docker image to ECR with appropriate tags
  - `docker build -t flask-app:${VERSION} .`
  - `docker tag flask-app:${VERSION} 421983921734.dkr.ecr.us-west-2.amazonaws.com/flask-app:latest`
  - `docker tag flask-app:${VERSION} 421983921734.dkr.ecr.us-west-2.amazonaws.com/flask-app:${VERSION}`
  - `docker push 421983921734.dkr.ecr.us-west-2.amazonaws.com/flask-app:latest`
  - `docker push 421983921734.dkr.ecr.us-west-2.amazonaws.com/flask-app:${VERSION}`
- Verify the Auto Scaling Group for the green environment is properly configured
  - Update instance refresh settings: Minimum healthy percentage: 90%
  - Check instance protection settings: Protect from scale-in disabled
  - Verify launch template version is correctly specified
- Create or update the Launch Template with the bootstrap script above
  - Set user data encoding to base64
  - Enable detailed CloudWatch monitoring
  - Set metadata response hop limit to 2
  - Configure instance metadata tags: enabled

### 2. Deployment Phase
- Initiate the green environment deployment
  - Create new Target Group for green environment:
    ```
    aws elbv2 create-target-group \
      --name flask-app-green \
      --protocol HTTP \
      --port 80 \
      --vpc-id vpc-12345678 \
      --health-check-path /health \
      --health-check-interval-seconds 30 \
      --healthy-threshold-count 2 \
      --unhealthy-threshold-count 3 \
      --target-type instance
    ```
  - Update Auto Scaling Group to use new Target Group:
    ```
    aws autoscaling update-auto-scaling-group \
      --auto-scaling-group-name flask-app-green-asg \
      --target-group-arns arn:aws:elasticloadbalancing:us-west-2:421983921734:targetgroup/flask-app-green/abcdef123456
    ```
- Auto Scaling Group launches instances using the Launch Template
  - Instance warm-up period: 180 seconds
  - Health check grace period: 300 seconds
  - Termination policies: [Default, OldestLaunchTemplate]
- Each instance bootstraps with Docker and pulls the new application version
  - Bootstrap logs available in /var/log/cloud-init-output.log
  - Docker container logs available with `docker logs [container_id]`
- A new Target Group is created to monitor the health of green instances
  - Load balancer algorithm: Round robin
  - Stickiness: Disabled
  - Slow start duration: 0 seconds

### 3. Validation Phase
- Verify all instances in the green environment pass health checks
  - Check target group health status using AWS console or CLI:
    ```
    aws elbv2 describe-target-health \
      --target-group-arn arn:aws:elasticloadbalancing:us-west-2:421983921734:targetgroup/flask-app-green/abcdef123456
    ```
  - Expected state: "healthy" for all instances
  - Verify registration delay has elapsed (typically 60-120 seconds)
- Conduct smoke tests to ensure the application functions correctly
  - Test directly against individual EC2 instances using their public IPs
  - Test via test endpoint on ALB (using path-based routing rules)
  - Run integration test suite against new environment:
    ```
    pytest --environment=green -v tests/integration/
    ```
- Monitor application metrics for any anomalies
  - CloudWatch dashboard metrics:
    - Request latency < 200ms (p95)
    - Error rate < 0.1%
    - CPU utilization < 60%
    - Memory utilization < 70%
  - Check ELB 5xx errors: Should be zero

### 4. Traffic Switch
- Update the ALB listener rule to route traffic to the green environment's Target Group
  ```
  aws elbv2 modify-listener \
    --listener-arn arn:aws:elasticloadbalancing:us-west-2:421983921734:listener/app/flask-app-alb/1234567890abcdef/fedcba0987654321 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-west-2:421983921734:targetgroup/flask-app-green/abcdef123456
  ```
- Monitor the application closely after the switch
  - Set up CloudWatch alarm for elevated error rates
  - Watch for significant changes in latency metrics
  - Monitor instance health in the target group
  - Check application logs for unexpected errors:
    ```
    aws logs filter-log-events \
      --log-group-name /aws/ec2/flask-app \
      --filter-pattern "ERROR"
    ```
- Keep the blue environment running as a quick rollback option
  - Do not terminate blue instances for at least 1 hour
  - Configure rollback script for one-command reversion:
    ```
    aws elbv2 modify-listener \
      --listener-arn arn:aws:elasticloadbalancing:us-west-2:421983921734:listener/app/flask-app-alb/1234567890abcdef/fedcba0987654321 \
      --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-west-2:421983921734:targetgroup/flask-app-blue/123456abcdef
    ```

### 5. Cleanup Phase
- Once the green environment is stable, the blue environment can be decommissioned or kept as standby
  - Wait period: At least 24 hours after successful cutover
  - Terminate blue instances using Auto Scaling Group update:
    ```
    aws autoscaling update-auto-scaling-group \
      --auto-scaling-group-name flask-app-blue-asg \
      --min-size 0 \
      --max-size 0 \
      --desired-capacity 0
    ```
  - Option: Keep minimum capacity of 1 for immediate rollback capability
  - Archive blue environment's CloudWatch logs
  - Create AMI backup of blue environment instance (optional):
    ```
    aws ec2 create-image \
      --instance-id i-0abc123def456789 \
      --name "flask-app-blue-backup-$(date +%Y%m%d)" \
      --description "Backup AMI for blue environment before decommission"
    ```
- Update documentation to reflect the current active environment
  - Update deployment runbook with current active color
  - Log deployment details in change management system
  - Update monitoring dashboards to focus on active environment
  - Update emergency contact information if ownership changes
  - Update DNS CNAME records if applicable

## Benefits We've Experienced

Since implementing this blue-green deployment strategy:

1. **Zero Downtime**: Our users experience no interruption during updates
2. **Increased Confidence**: We thoroughly test new versions before exposing them to users
3. **Quick Recovery**: If issues arise, we can revert to the previous version in seconds by switching the ALB target group
4. **Better Resource Utilization**: Auto Scaling Groups manage capacity efficiently based on demand
5. **Enhanced Security**: Regular updates with minimal risk keep our application secure

## Best Practices and Lessons Learned

Through our implementation, we've identified several best practices:

- **Automate Everything**: Script all deployment steps to reduce human error
  - Use AWS CLI or CloudFormation for infrastructure management
  - Create Jenkins/GitHub Actions pipelines for deployment automation
  - Implement validation checks with automatic rollback capability
  - Example rollback trigger: `if [[ $(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health) != "200" ]]; then ./rollback.sh; fi`

- **Standardize Configurations**: Use consistent naming conventions and configurations
  - Naming convention: `{application}-{environment}-{resource_type}`
  - Tag all resources with Environment, Owner, Cost-Center, and ManagedBy tags
  - Use Launch Template versions instead of AMI updates
  - Security groups should follow principle of least privilege

- **Implement Comprehensive Monitoring**: Set up detailed CloudWatch metrics and alarms
  - Custom metrics for application-specific concerns
    ```
    aws cloudwatch put-metric-data \
      --namespace "FlaskApp" \
      --metric-name "DatabaseConnections" \
      --value 42 \
      --dimensions Environment=Production
    ```
  - Set up composite alarms for complex conditions
  - Create dashboards for each environment with consistent metrics
  - Monitor network traffic between components

- **Test Rollbacks Regularly**: Practice reverting to previous versions to ensure the process works
  - Schedule monthly rollback drills
  - Measure time-to-rollback as a KPI
  - Test rollbacks during non-peak hours
  - Ensure database compatibility across versions

- **Document Everything**: Maintain detailed documentation of the current state and processes
  - Create architecture diagrams in draw.io or Lucidchart
  - Record all configuration parameters in a central wiki
  - Document expected metrics and baseline performance
  - Create troubleshooting runbooks for common issues

- **Use Parameter Store**: Store sensitive information and configuration in AWS Parameter Store rather than hardcoding values
  - Example parameter structure:
    ```
    /flask-app/production/database/connection_string
    /flask-app/production/api/key
    /flask-app/production/features/new_ui_enabled
    ```
  - Use encryption for sensitive parameters
  - Implement least-privilege access to parameters
  - Reference parameters in User Data scripts:
    ```bash
    DB_CONNECTION=$(aws ssm get-parameter --name "/flask-app/production/database/connection_string" --with-decryption --query "Parameter.Value" --output text)
    ```

- **Container Resource Limits**: Set appropriate CPU and memory limits for Docker containers
  ```bash
  docker run -d -p 80:80 --memory="512m" --cpus="1.0" 421983921734.dkr.ecr.us-west-2.amazonaws.com/flask-app:latest
  ```

- **Network Security**: Implement VPC Flow Logs and review regularly
  ```
  aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids vpc-12345678 \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-destination arn:aws:logs:us-west-2:421983921734:log-group:/vpc/flowlogs
  ```

## Security Considerations

During our implementation, we prioritized security at every level:

- **Network Security**:
  - VPC Flow Logs enabled for all traffic monitoring
  - Security Groups restricted to minimum required ports
  - Network ACLs as an additional security layer
  - All internal traffic routed through Transit Gateway for inspection

- **Container Security**:
  - ECR image scanning enabled with action on critical vulnerabilities
  - Non-root user in Docker containers: 
    ```Dockerfile
    RUN groupadd -r appuser && useradd -r -g appuser appuser
    USER appuser
    ```
  - Read-only file system where possible:
    ```bash
    docker run --read-only [...]
    ```
  - Security options for container hardening:
    ```bash
    docker run --security-opt=no-new-privileges [...]
    ```

- **IAM Security**:
  - Instance roles follow principle of least privilege
  - Permission boundaries applied to all production roles
  - AWS Organizations SCPs to prevent privilege escalation
  - Regular IAM credential rotation and access key audits

- **Data Security**:
  - All EBS volumes encrypted using KMS keys
  - S3 buckets with default encryption enabled
  - TLS 1.2+ for all ALB listeners
  - Secrets rotation using AWS Secrets Manager

## Cost Optimization Techniques

Our implementation includes several cost optimization strategies:

- **Instance Right-sizing**: Using t3.small instances based on CPU/memory profiling
- **Spot Instances**: For non-critical environments, saving up to 70%
- **Auto Scaling**: Scale in during low traffic periods (especially nights/weekends)
- **Reserved Instances**: For baseline capacity, saving approximately 40%
- **Cost Allocation Tags**: Tracking resources by project, department, and environment
- **S3 Lifecycle Policies**: For log archival to S3 Glacier after 90 days
- **CloudWatch Log Retention**: Set to 14 days for most logs to reduce storage costs

## Conclusion

Implementing blue-green deployments for our containerized Flask application on AWS has significantly improved our deployment process, reducing risk and eliminating downtime. The combination of Docker containerization with AWS's robust infrastructure services provides a flexible, scalable, and reliable platform for modern web applications.

By carefully orchestrating Launch Templates, Auto Scaling Groups, Target Groups, and ALB configuration, we've created a deployment pipeline that supports continuous delivery while maintaining high availability and performance.

The detailed configuration options and automation scripts shared in this article should provide you with a solid foundation for implementing your own blue-green deployment strategy. Remember that while the initial setup requires careful planning and configuration, the long-term benefits of improved stability, security, and operational efficiency make this approach well worth the investment.

I hope this detailed walkthrough helps others looking to implement similar deployment strategies. Feel free to connect if you have questions about this implementation or want to discuss AWS deployment strategies further.

#AWS #DevOps #Docker #FlaskApp #BlueGreenDeployment #CloudComputing #Containerization #ZeroDowntimeDeployment #InfrastructureAsCode



Project URL: https://roadmap.sh/projects/blue-green-deployment

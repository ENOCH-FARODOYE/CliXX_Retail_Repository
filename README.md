# CliXX Retail Application

E-commerce retail application built with PHP and WordPress, deployed as a containerized application on Amazon ECS. Features automated code quality scanning, containerized deployment, and CI/CD pipeline integration.

## Overview

CliXX is a retail web application that demonstrates modern DevOps practices applied to a PHP/WordPress stack. While WordPress is often deployed manually, this project shows how to properly containerize, automate, and deploy WordPress applications using enterprise DevOps tools and practices.

The application includes:
- WordPress-based e-commerce functionality
- Dockerized deployment for consistency
- Automated code quality scanning with SonarQube
- CI/CD pipeline with Jenkins
- Deployment to Amazon ECS
- MySQL database with automated restore capability

## Application Architecture

### Technology Stack

**Application Layer:**
- PHP 7.4+ with required extensions
- WordPress core and e-commerce plugins
- Custom themes and configurations
- Dynamic wp-config.php (reads from environment variables)

**Database Layer:**
- MySQL 5.7 database
- Automated backup and restore scripts
- Database migrations tracked
- Connection pooling configured

**Infrastructure:**
- Docker containers for portability
- Amazon ECS for orchestration
- Amazon ECR for image storage
- Application Load Balancer for traffic distribution
- Multi-AZ deployment for high availability

### Docker Configuration

The application is containerized with:

**Dockerfile:**
- Based on official PHP-Apache image
- WordPress installed and configured
- PHP extensions for MySQL connectivity
- Apache configured for WordPress permalinks
- Custom entrypoint script for dynamic configuration

**entrypoint.sh:**
- Reads database credentials from environment
- Generates wp-config.php at runtime
- Waits for database availability before starting
- Applies database migrations if needed
- Starts Apache in foreground

**wp-config.php:**
- Database host, user, password from environment variables
- Redis/Memcached configuration for caching
- Security keys from AWS Systems Manager
- Environment-specific settings (dev/staging/prod)

## CI/CD Pipeline

### Jenkins Pipeline Stages

The automated deployment pipeline consists of these stages:

**1. Declarative: Checkout SCM**
- Pulls latest code from GitHub repository
- Ensures pipeline uses current application code

**2. SonarQube Scan**
- Static code analysis for PHP code
- Checks for code smells, bugs, vulnerabilities
- Analyzes code coverage and complexity
- Scans WordPress plugins for security issues
- Average time: 12-14 seconds

**3. Quality Gate**
- Evaluates SonarQube results against quality standards
- Fails build if critical issues found
- Checks metrics like:
  - Code coverage threshold
  - Duplicated code percentage
  - Critical/blocker bugs count
  - Security vulnerabilities
- Average time: 2-3 minutes

**4. Build Docker Image**
- Builds Docker image from Dockerfile
- Tags image with version number and commit hash
- Includes application code and dependencies
- Optimized for size (multi-stage build)
- Average time: 30-40 seconds

**5. Starting Docker Image**
- Starts container locally for testing
- Verifies container starts successfully
- Checks application responds to health checks
- Validates environment variable configuration
- Average time: 12-15 minutes

**6. Restore CLIXX Database**
- Connects to test database
- Restores database backup for testing
- Runs database migrations
- Verifies data integrity
- Average time: 7-8 minutes

**7. Log Into ECR and Push Docker Image**
- Authenticates with Amazon ECR
- Pushes Docker image to container registry
- Tags image as "latest" and with version number
- Makes image available for ECS deployment
- Average time: 15-20 seconds

**8. Tear Down CLIXX Docker Image and Database**
- Stops and removes test containers
- Cleans up test database
- Removes unused Docker volumes
- Frees up resources for next build
- Average time: 4 minutes

### Pipeline Execution

**Trigger methods:**
- Automatic: Push to Git repository triggers via webhook
- Manual: Click "Build Now" in Jenkins
- Scheduled: Nightly builds for testing

**Build frequency:**
- Typically 5-10 builds per day during active development
- Full pipeline run time: 44 minutes average
- Most time spent in database restore and container validation

## Code Quality Standards

### SonarQube Integration

The SonarQube scan checks for:

**Code Quality Issues:**
- Code smells and maintainability problems
- Duplicated code blocks
- Cyclomatic complexity
- Cognitive complexity
- Naming conventions

**Security Vulnerabilities:**
- SQL injection risks
- Cross-site scripting (XSS) vulnerabilities
- Authentication and authorization issues
- Sensitive data exposure
- WordPress-specific security patterns

**Bugs and Reliability:**
- Null pointer exceptions
- Resource leaks
- Logic errors
- Exception handling problems

**Code Coverage:**
- Unit test coverage percentage
- Uncovered lines of code
- Branch coverage metrics

### Quality Gate Criteria

Build fails if:
- Critical or blocker issues found
- Code coverage below 60%
- Duplicated code above 10%
- Security vulnerabilities rated high or critical
- Maintainability rating below B

These standards ensure only quality code reaches production.

## Deployment Process

### ECS Deployment

After the Docker image is pushed to ECR:

1. **ECS Task Definition Update:**
   - New task definition created with latest image
   - Environment variables configured
   - Resource limits set (CPU, memory)
   - Health check parameters defined

2. **Service Update:**
   - ECS service updated to use new task definition
   - Rolling update strategy (no downtime)
   - Old tasks drained of traffic
   - New tasks receive traffic after health checks pass

3. **Health Validation:**
   - Load balancer performs health checks
   - Application must respond with 200 OK
   - Unhealthy tasks are automatically replaced
   - Rollback if health checks fail repeatedly

### Database Management

Database changes are handled separately:

**Schema Migrations:**
- Tracked in version control
- Applied automatically during deployment
- Rollback scripts available for each migration
- Tested in staging before production

**Data Backups:**
- Automated daily backups to S3
- Point-in-time recovery enabled
- Cross-region backup replication
- Retention policy: 30 days

## Development Workflow

### Local Development

Developers run the application locally using Docker:

```bash
# Start local environment
docker-compose up

# Access application
http://localhost:8080

# View logs
docker-compose logs -f

# Stop environment
docker-compose down
```

The docker-compose.yml includes WordPress, MySQL, and Redis for complete local environment.

### Code Changes

Standard workflow:

1. Create feature branch from main
2. Make changes to PHP code or WordPress configuration
3. Test locally with Docker
4. Commit changes with descriptive message
5. Push to GitHub
6. Jenkins pipeline runs automatically
7. Review SonarQube findings
8. Fix any quality issues
9. Merge to main after approval
10. Automatic deployment to staging
11. Manual promotion to production

## Monitoring and Observability

### Application Metrics

CloudWatch tracks:
- Request count and latency
- Error rates (4xx, 5xx responses)
- Database connection pool usage
- PHP-FPM process metrics
- Memory and CPU utilization

### Logs

Centralized logging:
- Apache access logs
- PHP error logs
- WordPress debug logs
- All forwarded to CloudWatch Logs
- Retained for 30 days
- Searchable and filterable

### Alerts

Configured alerts for:
- High error rate (>5% 5xx errors)
- Slow response time (>2 seconds)
- Database connection failures
- Container restart loops
- Disk space issues

## Security Considerations

### Application Security

**Input Validation:**
- All user input sanitized
- WordPress security functions used
- Prepared statements for database queries
- CSRF tokens on forms

**Authentication:**
- Strong password requirements
- Session timeout after 15 minutes inactivity
- Failed login attempt tracking
- Two-factor authentication available

**Data Protection:**
- Sensitive data encrypted at rest
- HTTPS enforced (redirects from HTTP)
- Security headers configured (CSP, HSTS)
- Regular security plugin updates

### Infrastructure Security

**Container Security:**
- Non-root user in container
- Read-only file system where possible
- Minimal base image (official PHP image)
- Vulnerability scanning with Inspector

**Network Security:**
- Application in private subnets
- Load balancer in public subnets
- Database in isolated data subnet
- Security groups restrict traffic

**Secrets Management:**
- No credentials in code or environment files
- Database passwords in AWS Systems Manager
- API keys in Secrets Manager
- Automatic rotation enabled

## Performance Optimization

### Caching Strategy

**Object Caching:**
- Redis for WordPress object cache
- Transient API optimization
- Query result caching
- Reduces database load by 70%

**Page Caching:**
- Full page caching for anonymous users
- Cache invalidation on content updates
- CDN integration for static assets
- Browser caching headers configured

**Database Optimization:**
- Indexed frequently queried columns
- Query optimization for WooCommerce
- Connection pooling
- Read replicas for reporting queries

## Cost Optimization

**Current monthly costs:**
- ECS tasks (Fargate): $60-80
- RDS MySQL: $50-75
- Application Load Balancer: $20-25
- ECR storage: $5-10
- Data transfer: $10-15
- **Total: ~$145-205/month**

**Optimization strategies:**
- Right-sized ECS task resources
- Reserved Instance pricing for database
- S3 lifecycle policies for old images
- CloudWatch log retention limits

## Troubleshooting

### Common Issues

**Container won't start:**
- Check CloudWatch logs for PHP errors
- Verify database connectivity
- Validate environment variables
- Check wp-config.php generation

**Slow performance:**
- Review CloudWatch metrics for bottleneck
- Check database slow query log
- Verify Redis cache is working
- Review PHP-FPM configuration

**Failed deployments:**
- Check SonarQube quality gate results
- Review Jenkins console output
- Verify ECR image was pushed
- Check ECS task definition is valid

## Related Infrastructure

This application is deployed using:

- **CLIXX-ECS** - ECS cluster and service definitions
- **STACK_TERRAFORM** - VPC, networking, and supporting infrastructure
- **golden-ami-pipeline** - Base AMI for EC2 instances (if using EC2 launch type)

## Author

**Enoch Farodoye**

LinkedIn: [linkedin.com/in/enoch-farodoye](https://linkedin.com/in/enoch-farodoye)

Email: farodoyeenoch1@gmail.com

GitHub: [@ENOCH-FARODOYE](https://github.com/ENOCH-FARODOYE)

---

This project demonstrates how traditional PHP/WordPress applications can be modernized with containerization, automated testing, and cloud-native deployment practices while maintaining the simplicity and flexibility that makes WordPress popular.

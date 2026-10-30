# Blue-Green Deployment on AWS ECS

A containerized application deployment project demonstrating the
**blue-green deployment strategy** on Amazon Web Services (AWS). The
objective is to release a new application version with minimal downtime
while keeping the previous version available for a quick rollback.

## Project Overview

Blue-green deployment uses two separate application environments:

-   **Blue environment:** The currently active version receiving
    production traffic.
-   **Green environment:** The new version deployed and tested before
    receiving production traffic.

An **Application Load Balancer (ALB)** routes user requests to the
active environment. After the green environment passes health checks and
validation, traffic can be shifted from blue to green. If an issue
occurs, traffic can be routed back to the previous environment.

## Objectives

-   Containerize the application using Docker.
-   Store and manage container images in Amazon Elastic Container
    Registry (Amazon ECR).
-   Define application containers using Amazon ECS task definitions.
-   Run separate blue and green ECS services or task sets.
-   Configure an ALB with target groups and health checks.
-   Shift traffic to the new version only after validation.
-   Test rollback to the previous stable version.

## Architecture

``` text
                         Users
                           |
                           v
                Application Load Balancer
                           |
                 Listener / Traffic Routing
                      /          \
                     v            v
              Blue Target      Green Target
              Group            Group
                  |                |
                  v                v
             Blue ECS         Green ECS
              Service           Service
                  |                |
                  v                v
             Old Version      New Version
             Containers       Containers

       Docker Image ---> Amazon ECR ---> ECS Task Definition
```

The ALB listener forwards traffic to the target group associated with
the active environment. Both environments can remain available during
the release so that rollback is possible.

## AWS Services and Tools

  -----------------------------------------------------------------------
  Service / Tool                      Purpose
  ----------------------------------- -----------------------------------
  Docker                              Builds the application container
                                      image

  Amazon ECR                          Stores versioned Docker images

  Amazon ECS                          Runs and manages application
                                      containers

  ECS Task Definitions                Define the image, CPU, memory,
                                      ports, and container settings

  ECS Services                        Maintain the desired number of
                                      running tasks

  Application Load Balancer           Routes incoming application traffic

  ALB Target Groups                   Register and health-check blue and
                                      green targets

  AWS CodeDeploy (if configured)      Coordinates ECS blue-green
                                      deployments and traffic shifting

  CloudWatch (if configured)          Provides logs, metrics, and
                                      deployment monitoring
  -----------------------------------------------------------------------

## Deployment Workflow

1.  **Build the application image**
    -   Create a Docker image from the application source code.
    -   Tag the image with a version identifier.
2.  **Push the image to Amazon ECR**
    -   Authenticate Docker with the ECR registry.
    -   Push the tagged image to the ECR repository.
3.  **Create the ECS task definition**
    -   Configure the container image, CPU, memory, port mappings, and
        required environment settings.
    -   Register a new task-definition revision for the release.
4.  **Run the blue and green environments**
    -   Keep the current stable version in the blue environment.
    -   Deploy the new version to the green environment using the
        appropriate ECS service or task-set configuration.
5.  **Configure the Application Load Balancer**
    -   Attach the blue and green target groups to the ALB listener or
        deployment configuration.
    -   Configure the application health-check path, port, and success
        criteria.
6.  **Validate the green environment**
    -   Confirm that the new ECS tasks are running.
    -   Verify that the green target group reports healthy targets.
    -   Test the new application version before shifting production
        traffic.
7.  **Shift traffic**
    -   Move traffic from blue to green using the configured ALB
        listener or CodeDeploy traffic-shifting strategy.
    -   Monitor application health during and after the switch.
8.  **Rollback if required**
    -   If the new version fails validation or causes issues, route
        traffic back to the previous stable blue environment.
    -   Investigate the issue before attempting another release.

## Health Checks and Validation

Before directing production traffic to the new version, verify:

-   ECS tasks reach the `RUNNING` state.
-   The green target group reports healthy targets.
-   The configured health-check endpoint returns the expected success
    response.
-   The application is reachable through the ALB.
-   Application logs and metrics do not show critical errors.

Example AWS CLI checks (replace placeholders with your actual resource
names and region):

``` bash
aws ecs describe-services \
  --cluster <cluster-name> \
  --services <blue-service-name> <green-service-name> \
  --region <aws-region>

aws elbv2 describe-target-health \
  --target-group-arn <green-target-group-arn> \
  --region <aws-region>

aws ecs describe-task-definition \
  --task-definition <task-definition-family-or-revision> \
  --region <aws-region>
```

## Rollback Strategy

The previous stable environment is retained during the deployment. If
the green release is unhealthy:

1.  Stop or reverse the traffic shift.
2.  Route the ALB listener back to the blue target group, or use the
    rollback action supported by the configured deployment controller.
3.  Confirm that blue targets are healthy and the application is
    accessible.
4.  Review logs and deployment events to identify the cause.
5.  Correct the issue and deploy a new green revision when ready.

The exact rollback operation depends on whether the ECS service uses the
ECS rolling deployment controller or the CodeDeploy blue-green
deployment controller.

## Expected Benefits

-   Minimal application downtime during releases.
-   Safer production deployments through pre-release validation.
-   Faster recovery by retaining the previous stable version.
-   Clear separation between the current and new application versions.
-   Improved release confidence through health checks and controlled
    traffic shifting.

## Prerequisites

-   An AWS account with permissions to use ECR, ECS, Elastic Load
    Balancing, and (if used) CodeDeploy.
-   AWS CLI configured with the intended account and region.
-   Docker installed and running.
-   An application with a working Dockerfile.
-   An ECR repository, ECS cluster, task definition, services, ALB, and
    target groups configured for the chosen deployment approach.

## Important Notes

-   Replace all `<placeholder>` values in the commands with your actual
    AWS resource names and ARNs.
-   This README describes the project architecture and workflow. Only
    list AWS resources, CodeDeploy automation, monitoring, or test
    results as implemented if you actually created and verified them.
-   For automated ECS blue-green deployments, configure the ECS service
    with the CodeDeploy deployment controller and define the required
    CodeDeploy application, deployment group, listener, and target
    groups. A manual ALB traffic switch is a different implementation
    and should be described as such.

## Conclusion

This project demonstrates how blue-green deployment on Amazon ECS can
make application releases safer and more reliable. By deploying a new
version separately, validating its health, shifting traffic through an
ALB, and retaining the previous version for rollback, teams can reduce
deployment risk and minimize service disruption.

# Seclock – DevOps CI/CD and Monitoring Project

## Project Overview

Seclock is a web application deployed on Amazon EKS using a DevOps workflow that automates application testing, code quality analysis, Docker image building, and deployment.

This project demonstrates how to combine CI/CD, GitOps, containerization, Kubernetes, and monitoring to build a repeatable application deployment process.

The application source code is maintained in GitHub, Docker images are stored in Amazon ECR, Jenkins automates the CI/CD pipeline, and Argo CD synchronizes Kubernetes manifests from GitHub to the EKS cluster.

## Architecture

```text
Developer
   |
   v
GitHub Repository
   |
   v
Jenkins CI/CD Pipeline
   |
   +--> Install Dependencies
   |
   +--> Run Tests
   |
   +--> SonarQube Analysis
   |
   +--> Build Docker Image
   |
   +--> Push Image to Amazon ECR
   |
   +--> Update Kubernetes Manifest
                |
                v
          GitHub (GitOps)
                |
                v
             Argo CD
                |
                v
          Amazon EKS
                |
                v
         Seclock Application
                |
                v
          AWS LoadBalancer

Monitoring:
Prometheus --> Grafana
EC2 Cron Job --> HTTP Health-Check Log
```

## Technologies Used

| Technology | Purpose |
|---|---|
| Git and GitHub | Source code and version control |
| Jenkins | CI/CD pipeline automation |
| SonarQube | Static code quality analysis |
| Docker | Containerization |
| Amazon ECR | Container image registry |
| Amazon EKS | Managed Kubernetes deployment |
| Kubernetes | Application deployment and service management |
| Argo CD | GitOps-based deployment synchronization |
| Prometheus | Collection of Kubernetes and container metrics |
| Grafana | Monitoring dashboards and visualization |
| Helm | Installation of Prometheus and Grafana |
| Linux and Bash | Server administration and automation |
| AWS IAM | AWS access through an EC2 instance role |

## CI/CD Workflow

### 1. Source Code Management

The application source code and Kubernetes manifests are maintained in GitHub. Git is used to track changes and manage the project.

### 2. Jenkins Pipeline

Jenkins runs the pipeline stages:

- Checks out the source code.
- Installs Python dependencies.
- Executes automated tests.
- Performs static code analysis using SonarQube.
- Builds the Docker image.
- Authenticates with Amazon ECR.
- Pushes the image to the ECR repository.
- Updates the Kubernetes deployment manifest in GitHub with the new image tag.

### 3. Containerization and Amazon ECR

The application is packaged into a Docker image using a Dockerfile. Jenkins builds and pushes the image to Amazon Elastic Container Registry (ECR).

Using versioned image tags helps identify which application image is deployed.

### 4. Deployment on Amazon EKS

Amazon EKS runs the Kubernetes workloads. The application is deployed with two replicas for basic redundancy.

Kubernetes Deployment and Service manifests define the application replicas, container configuration, and external access through an AWS LoadBalancer.

### 5. GitOps with Argo CD

Argo CD monitors the GitHub repository containing the Kubernetes manifests.

When Jenkins updates the deployment manifest, Argo CD detects the change and synchronizes the desired configuration with the EKS cluster.

This separates the CI process, which builds and publishes the image, from the CD process, which applies the deployment configuration.

## Monitoring and Observability

### Prometheus

Prometheus is installed in the `monitoring` namespace using Helm. It collects Kubernetes and container metrics, including CPU and memory usage.

The Prometheus configuration uses a lightweight setup to reduce resource consumption in the cluster.

### Grafana

Grafana is connected to Prometheus as a data source. A custom dashboard named **Seclock Monitoring** visualizes:

- Seclock application CPU usage.
- Seclock application memory usage.
- Container CPU usage by node.
- Container memory usage by node.

The node-level panels display aggregated container metrics for each node; they should not be interpreted as complete host CPU or memory utilization.

### Automated HTTP Health Check

A Bash script runs on the EC2 management server to check the application's external HTTP endpoint.

The script records:
- Timestamp of each check.
- HTTP response status.
- Request response time.
- Healthy or unhealthy status.

A cron job executes the script every minute and appends the results to a log file.

This provides basic periodic availability checks without deploying an additional Kubernetes pod. These HTTP check results are logged locally and are not currently integrated into Prometheus or Grafana.

## Kubernetes Resources

The main resources used in the project include:

- **Namespace:** `seclock` for the application.
- **Deployment:** Runs two application replicas.
- **Service:** Exposes the application through an AWS LoadBalancer.
- **Monitoring namespace:** Runs Prometheus and Grafana.
- **Argo CD namespace:** Contains the GitOps controllers and application management components.

Metrics Server is also installed to provide Kubernetes resource metrics through commands such as `kubectl top nodes`.

## Challenges and Solutions

### 1. Kubernetes Pod and Resource Limits

**Challenge:** Running the application, Argo CD, Prometheus, Grafana, and supporting services within a small EKS worker-node configuration required careful resource management.

**Solution:** Used resource requests and limits for the application and monitoring workloads, disabled unnecessary Prometheus chart components, and checked pod distribution and node memory usage before considering additional monitoring services.

### 2. Monitoring Without Additional Pods

**Challenge:** Adding another monitoring exporter could increase pod and memory consumption.

**Solution:** Used an EC2-hosted Bash health-check script and cron to perform periodic HTTP checks without adding another Kubernetes pod.

### 3. GitOps Deployment Updates

**Challenge:** Building a new container image does not automatically update a Kubernetes deployment when the deployment manifest still references an older tag.

**Solution:** Added a Jenkins stage to update the image tag in the GitHub deployment manifest, allowing Argo CD to synchronize the new desired configuration.

### 4. Accessing Private Dashboards

**Challenge:** Prometheus, Grafana, and Argo CD dashboards were not exposed directly to the public internet.

**Solution:** Used Kubernetes port forwarding and SSH tunnels to access the dashboards locally without exposing their services through public LoadBalancers.

### 5. Verifying Application Availability

**Challenge:** A successful deployment does not, by itself, confirm that the application is responding correctly through its external endpoint.

**Solution:** Tested the LoadBalancer endpoint using `curl` and created a scheduled health-check script that records HTTP status and response time.

## Validation and Testing

The following checks were performed during the project:

- Automated application test completed successfully.
- SonarQube analysis was included in the Jenkins pipeline.
- Docker image was built and pushed to Amazon ECR.
- Application endpoint returned HTTP 200.
- Two application replicas were running in EKS.
- Both EKS worker nodes reported `Ready`.
- Argo CD application reported `Synced` and `Healthy`.
- Prometheus successfully collected container metrics from both nodes.
- Grafana successfully queried Prometheus and displayed the monitoring dashboards.
- Cron successfully executed the HTTP health-check script and recorded results.

## Key Learnings

- Building CI/CD pipelines with Jenkins.
- Integrating automated tests and SonarQube analysis.
- Building and publishing Docker images to Amazon ECR.
- Deploying applications on Amazon EKS.
- Implementing GitOps using Argo CD.
- Monitoring Kubernetes workloads using Prometheus and Grafana.
- Managing AWS access through IAM roles.
- Troubleshooting Kubernetes resource constraints.
- Automating periodic HTTP checks with Bash and cron.

## Future Improvements

- Integrate HTTP health and response-time metrics into Prometheus and Grafana.
- Configure alerting for application failures and resource thresholds.
- Add HTTPS with a managed TLS certificate and a suitable ingress or load-balancing configuration.
- Improve persistent storage and retention for monitoring data.
- Add resource and cost optimization for AWS infrastructure.
- Automate infrastructure provisioning with Terraform.

## Repository Structure

```text
seclock/
├── Dockerfile
├── Jenkinsfile
├── requirements.txt
├── main.py
├── test_e2e.py
├── sonar-project.properties
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── prometheus-values.yaml
```

The repository may contain additional application files, static assets, and local monitoring configuration files.

## Conclusion

This project provided practical experience in automating the build, test, image publishing, and deployment workflow for an application using AWS and open-source DevOps tools.

It also demonstrated GitOps-based deployment management and Kubernetes resource monitoring using Prometheus and Grafana, along with a lightweight HTTP health-check process.

The setup is a hands-on learning project and can be extended with alerting, more complete observability, infrastructure automation, and production-grade security.


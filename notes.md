# Kubernetes on AWS (EKS) 

Amazon EKS (Elastic Kubernetes Service) is AWS’s **managed Kubernetes** offering, integrating seamlessly with other AWS services like EC2, IAM, VPC, ECR, and Fargate.

***

## 1. Container Services on AWS

AWS provides multiple services for container orchestration and storage:

| Service | Purpose |
|---|---|
| **ECS** | AWS-specific container orchestration service. |
| **EKS** | Managed Kubernetes (AWS-hosted Kubernetes control plane). |
| **ECR** | Private Docker registry on AWS, integrates with ECS/EKS. |

### ECS (Elastic Container Service)
- Orchestration with lifecycle mgmt (start, re-schedule, LB).
- ECS cluster = control plane managing connected EC2 instances running containers.
- Requires you to provision & patch EC2 nodes (contain container runtime & ECS agent).
- Can use **AWS Fargate** for serverless container deployment (no EC2 to manage, billed per use).

### EKS (Elastic Kubernetes Service)
- AWS provisions & manages Kubernetes control plane (multi-AZ masters, etcd replication).
- Worker node options:
  - **Self-managed EC2 group**.
  - **Managed Node Group** (AWS manages OS, Kubelet, etc.).
  - **Fargate** (fully serverless worker nodes; one VM per Pod).
- EKS vs ECS:
  - ECS is AWS-only, simpler, free control plane.
  - Kubernetes (EKS) is portable, open-source, more complex.

***

## 2. Creating an EKS Cluster via AWS Console

**High-level steps:**
1. **IAM Role for EKS Cluster**
   - Role: `EKS - Cluster`
   - Policy: `AmazonEKSClusterPolicy`
2. **VPC for EKS Worker Nodes**
   - Use AWS CloudFormation VPC template with public+private subnets, SG configured for EKS ([template link](https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml)).
3. **Create EKS Cluster**
   - Name, K8s version, EKS IAM role, VPC & subnets, SGs, public+private API access.
4. **Connect to EKS with kubectl**
   ```sh
   aws eks update-kubeconfig --name eks-cluster-test
   kubectl cluster-info
   ```
5. **IAM Role for Node Group**
   - Service: EC2
   - Policies:
     - `AmazonEKSWorkerNodePolicy`
     - `AmazonEC2ContainerRegistryReadOnly`
     - `AmazonEKS_CNI_Policy`
6. **Add Node Group**
   - Instance type, disk size, scaling config, SSH key.

***

## 3. Cluster Autoscaling in EKS

**Purpose:** Adjust EC2 nodes in a Node Group based on workload.

Steps:
- Custom IAM Policy for auto-scaler → attach to Node Group Role.
- Deploy Cluster Autoscaler:
  ```sh
  kubectl apply -f cluster-autoscaler-autodiscover.yaml
  kubectl edit deployment cluster-autoscaler -n kube-system
  # Configure cluster name, options, image version match, safe-to-evict annotation.
  ```
- Adjust ASG min/max in EC2 dashboard for testing.
- Verify logs and node scaling.

**Load Test Example:** Deploy Nginx + LoadBalancer, increase replicas to trigger autoscaler.

***

## 4. Fargate Profile for EKS

- Fully-managed worker nodes (no EC2 management).
- Limitations:
  - No stateful apps.
  - No DaemonSets.
- Create IAM Role for `EKS - Fargate pod` with `AmazonEKSFargatePodExecutionRolePolicy`.
- Create Fargate Profile:
  - Namespace + label selector.
- Deploy workloads with matching namespace/labels → scheduled onto Fargate-managed infra.

***

## 5. eksctl CLI – Rapid EKS Creation

**Install (Mac example):**
```sh
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```
**Auth (shares AWS CLI creds):**
```sh
aws configure
```
**Create cluster:**
```sh
eksctl create cluster \
  --name demo-cluster \
  --version 1.28 \
  --region eu-central-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.micro \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3
```
- Automatically configures kubeconfig.

***

## 6. Deploying to EKS from Jenkins

**Jenkins container prerequisites:**
- Install `kubectl`.
- Install `aws-iam-authenticator` (needed for EKS auth).
- Prepare kubeconfig:
  - Generate with `aws eks update-kubeconfig`.
  - In Jenkins, replace `exec` section to use:
    ```yaml
    exec:
      command: aws-iam-authenticator
      args: ["token", "-i", "demo-cluster"]
    ```
- Copy kubeconfig into `/var/jenkins_home/.kube/config` and chown to `jenkins`.
- Add AWS credentials in Jenkins (Secret Text entries).

**Jenkinsfile deploy example:**
```groovy
stage('deploy') {
  environment {
    AWS_ACCESS_KEY_ID = credentials('jenkins-aws_access_key_id')
    AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
  }
  steps {
    sh 'kubectl create deployment nginx-deployment --image=nginx'
  }
}
```

***

## 7. Deploying to Linode LKE from Jenkins

Steps differ from EKS:
- Create LKE cluster & download kubeconfig.
- Upload kubeconfig to Jenkins as "Secret file".
- Install "Kubernetes CLI" plugin.
- Jenkinsfile deploy example:
```groovy
withKubeConfig([credentialsId: 'lke-credentials', serverUrl: '']) {
  sh 'kubectl create deployment nginx-deployment --image=nginx'
}
```

***

## 8. Jenkins Credentials Best Practices

- Always create **least-privilege** service accounts/users per environment (AWS, EC2, EKS, LKE).
- Store only these scoped credentials in Jenkins.

***

## 9. Complete CI/CD Pipeline – EKS + DockerHub

**K8s Manifests** (`kubernetes/deployment.yaml` & `service.yaml`):
- Use env vars (`$APP_NAME`, `$IMAGE_TAG`) and `envsubst` to inject values in Jenkins pipeline.
- Install `gettext-base` in Jenkins container for `envsubst`.

**Registry Secret in K8s**:
```sh
kubectl create secret docker-registry my-registry-key \
  --docker-server=docker.io \
  --docker-username=myuser \
  --docker-password=mypassword
```
**Jenkinsfile deploy snippet:**
```groovy
sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
```

***

## 10. CI/CD with EKS + ECR

- In ECR: Create private repo (per app).
- Jenkins: Add ECR creds (Username='AWS', Password from `aws ecr get-login-password`).
- In K8s: Secret for ECR registry with same creds.

**Jenkinsfile**:
```groovy
environment {
  DOCKER_REPO_HOST = '...amazonaws.com'
  DOCKER_REPO_URI  = "${DOCKER_REPO_HOST}/java-maven-app"
}
...
sh "docker build -t ${DOCKER_REPO_URI}:${IMAGE_TAG} ."
sh "echo $PASSWORD | docker login -u $USERNAME --password-stdin ${DOCKER_REPO_HOST}"
sh "docker push ${DOCKER_REPO_URI}:${IMAGE_TAG}"
```
- Update deployment manifest to use `${DOCKER_REPO_URI}:${IMAGE_TAG}`.
- Reference ECR secret in `imagePullSecrets`.

***

## 11. Cleanup EKS Resources

Before deleting cluster:
1. Delete Node Groups.
2. Delete Fargate Profiles.
3. Delete EKS Cluster.
4. Delete associated IAM Roles.
5. Optionally delete custom IAM policies.

***

### 🔹 Key Takeaways
- **EKS** control plane is managed by AWS, you choose between EC2 worker nodes, managed node groups, or Fargate.
- Use **IAM Roles** with correct policies for both cluster and worker nodes.
- Configure **autoscaling** using Cluster Autoscaler with ASG integration.
- **eksctl** is the most efficient CLI for EKS provisioning.
- For CI/CD deployments from Jenkins:
  - Ensure kubeconfig + AWS authenticator available in Jenkins.
  - Manage registry secrets for private image pulls.
  - Automate env value injection in K8s manifests.

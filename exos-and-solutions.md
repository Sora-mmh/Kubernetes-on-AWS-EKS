# Kubernetes on AWS — Exercises & Solutions

**Repository to use:** [GitLab – eks-exercises](https://gitlab.com/twn-devops-bootcamp/latest/11-eks/eks-exercises.git)

Your task in this module is to migrate your Kubernetes application from LKE/Minikube to AWS EKS, set up continuous deployment with Jenkins, and configure autoscaling to optimize costs.

---

## Exercise 1: Create EKS cluster

**Task:**
You decide to create an EKS cluster - AWS managed Kubernetes Service. To simplify the whole creation and configuration process, you use *eksctl*.
- With eksctl you create an EKS cluster with 3 Nodes and 1 Fargate profile

**Solution:**

First, install eksctl command line tool locally. See the installation guide [here](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html).

```sh
# create cluster with 3 EC2 instances and store access configuration to cluster in kubeconfig.my-cluster.yaml file 
eksctl create cluster --name=my-cluster --nodes=3 --kubeconfig=./kubeconfig.my-cluster.yaml

# create fargate profile in the cluster. It will apply for all K8s components in my-app namespace
eksctl create fargateprofile \
    --cluster my-cluster \
    --name my-fargate-profile \
    --namespace my-app

# point kubectl to your cluster - use absolute path to kubeconfigfile
export KUBECONFIG={absolute-path}/kubeconfig.my-cluster.yaml

# validate cluster is accessible and nodes and fargate profile created
kubectl get node
eksctl get fargateprofile --cluster my-cluster
```

---

## Exercise 2: Deploy Mysql and phpmyadmin

**Task:**
- You deploy mysql and phpmyadmin on the EC2 nodes using the same setup as before.

**Solution:**

**General notes:**
- All the k8s manifest files for the exercise are in "k8s-deployment" folder

```sh
# clone this repository locally
git clone https://gitlab.com/twn-devops-bootcamp/latest/11-eks/eks-exercises.git

# check out the solutions branch
git checkout feature/solutions

# change to k8s-deployment folder
cd k8s-deployment
```

**Deploy MySQL and phpMyAdmin:**

*Mysql Chart link: https://github.com/bitnami/charts/tree/master/bitnami/mysql*

```sh
# install Mysql chart 
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release bitnami/mysql -f mysql-chart-values-eks.yaml

# deploy phpmyadmin with its configuration for Mysql DB access
kubectl apply -f db-config.yaml
kubectl apply -f db-secret.yaml
kubectl apply -f phpmyadmin.yaml

# access phpmyadmin and login to mysql db
kubectl port-forward svc/phpmyadmin-service 8081:8081

# access in browser on
localhost:8081

# login with one of these 2 credentials
"my-user" : "my-pass"
"root" : "secret-root-pass"
```

---

## Exercise 3: Deploy your Java Application

**Task:**
- You deploy your Java application using Fargate with 3 replicas using the same setup as before.

**Solution:**

```sh
# Create namespace my-app to deploy our java application, because we are deploying java-app with fargate profile. 
# And fargate profile we create applies for my-app namespace. 
kubectl create namespace my-app

# We now have to create all configuration and secrets for our java app in the my-app namespace

# Create my-registry-key secret to pull image 
DOCKER_REGISTRY_SERVER=docker.io
DOCKER_USER=your dockerID, same as for `docker login`
DOCKER_EMAIL=your dockerhub email, same as for `docker login`
DOCKER_PASSWORD=your dockerhub pwd, same as for `docker login`

kubectl create secret -n my-app docker-registry my-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD \
--docker-email=$DOCKER_EMAIL

# Again from k8s-deployment folder, execute following commands. 
# By adding the my-app namespace, these components will be created with Fargate profile
kubectl apply -f db-secret.yaml -n my-app
kubectl apply -f db-config.yaml -n my-app
kubectl apply -f java-app.yaml -n my-app
```

---

## Exercise 4: Automate deployment

**Task:**
Now your application is running, and when you or others make changes to it, Jenkins pipeline builds the new image, but you have to manually deploy it into the cluster. From your experience you know how annoying that is for you and your team, so you want to automate deploying to the cluster as well.
- Setup automatic deployment to the cluster in the pipeline.

**Solution:**

**Current cluster setup:**
At this point, you already have an EKS cluster, where: 
- Mysql chart is deployed and phpmyadmin is running too
- my-app namespace was created
- db-config and db-secret were created in the my-app namespace for the java-app
- my-registry-key secret was created to fetch image from docker-hub
- your java app is also running 

**Steps to automate deployment for existing setup:**

```sh
# SSH into server where Jenkins container is running
ssh -i {private-key-path} {user}@{public-ip}

# Enter Jenkins container
sudo docker exec -it {jenkins-container-id} -u 0 bash

# Install aws-cli inside Jenkins container
# Link: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

# Install kubectl inside Jenkins container
# Link: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
chmod 644 /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubectl

# Install envsubst tool
# Link: https://command-not-found.com/envsubst
apt-get update
apt-get install -y gettext-base

# create 2 "secret-text" credentials for AWS access in Jenkins: 
- "jenkins_aws_access_key_id" for AWS_ACCESS_KEY_ID 
- "jenkins_aws_secret_access_key" for AWS_SECRET_ACCESS_KEY    

# Create 4 "secret-text" credentials for db-secret.yaml:
- id: "db_user", secret: "my-user"
- id: "db_pass", secret: "my-pass"
- id: "db_name", secret: "my-app-db"
- id: "db_root_pass", secret: "secret-root-pass"

# Set the correct values in Jenkins for following environment variables: 
- ECR_REPO_URL
- CLUSTER_REGION

# Create Jenkins pipeline using the Jenkinsfile in this branch, in the root folder
# Make sure the paths to the k8s manifest files in the "deploy" stage of the Jenkinsfile are all correct!!
```

---

## Exercise 5: Use ECR as Docker repository

**Task:**
So, the company wants to use ECR instead, again to have everything on 1 platform and also to let AWS manage the repository including storage, cleanups etc. Therefore you:
- Replace the docker repository in your pipeline with ECR

**Solution:**

```sh
# Create an ECR registry for your java-app image

# Locally, on your computer: Create a docker registry secret for ECR
DOCKER_REGISTRY_SERVER=your ECR registry server - "your-aws-id.dkr.ecr.your-ecr-region.amazonaws.com"
DOCKER_USER=your dockerID, same as for `docker login` - "AWS"
DOCKER_PASSWORD=your dockerhub pwd, same as for `docker login` - get using: "aws ecr get-login-password --region {ecr-region}"

kubectl create secret -n my-app docker-registry my-ecr-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD
```

*Note: The rest of the Jenkins setup from Exercise 4 applies here as well, with ECR configuration integrated into the pipeline.*

---

## Exercise 6: Configure Autoscaling

**Task:**
Now your application is running, whenever a change is made, it gets automatically deployed in the cluster etc. This is great, but you notice that most of the time the 3 nodes you have are underutilized, especially at the weekends, because your containers aren't using that many resources. However, your company is still paying full price for all of the servers.

You suggest to your manager that you will be able to save the company some infrastructure costs by configuring autoscaling. Your manager is happy with this suggestion and asks you to configure it. So you:
- Go ahead and configure autoscaling to scale down to a minimum of 1 node when servers are underutilized and maximum of 3 nodes when in full use.

**Solution:**

You learn how to scale the cluster up and down in the **_Kubernetes on AWS_** module, video **_3 - Configure Autoscaling in EKS cluster_**.

*Note: Detailed autoscaling configuration steps would involve setting up Cluster Autoscaler or using managed node groups with auto scaling groups configured with min=1 and max=3 nodes.*

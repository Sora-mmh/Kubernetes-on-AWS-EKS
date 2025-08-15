# Kubernetes-on-AWS-EKS

The listed project in this module cover:

### **1. Jenkins → EKS Deployment**
- Installed **kubectl** & **aws-iam-authenticator** inside the Jenkins container.  
- Built a **kubeconfig** file and stored it as **Jenkins credential**.  
- Created a **Jenkinsfile** that **applies Kubernetes manifests** directly to EKS.
- 
---

### **2. DockerHub & ECR CI/CD Pipelines**
| Registry | Extra Steps |
| --- | --- |
| **DockerHub** | ① `envsubst` to inject image tags<br>② K8s **imagePullSecret** for DockerHub |
| **AWS ECR** | ① Created ECR repo<br>② Jenkins **ECR credential**<br>③ Updated manifests to use ECR URL |

In both cases the Jenkins pipeline:
- Builds & pushes image  
- Updates manifest via `envsubst`  
- Runs `kubectl apply` on EKS  

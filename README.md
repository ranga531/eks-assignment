# EKS Clutser Provision

We have eks folder where we written all the terraform code to provision it in any region, to create eks cluster we need to run below terraform commands.
```
terraform init
```
### terraform init will install all the plugins for us and prepare our code to provision.

```
terraform plan
```
### terraform plan will execute our code as a dry run and will show us changes that are going to happen as Create, Update or Destroy.

```
terraform apply
```

### terraform apply will create all the resources that we written in terraform code, once this installation is done it will show us the cluster name as output

Once our cluster is ready then we create deployment and service as Load Balancer for our sample app. To create this service switch to sample-app folder and run ``` kubectl apply -f sample-app.yaml ```

this will create a nginx deployment with 1 pod and service which will be expose as LoadBalancer. Command to see LB URL ``` kubectl get svc ```, it will show output like below
![image](https://user-images.githubusercontent.com/24311808/145248631-fde9bf1e-b8f5-4da5-a2b4-ccf7ffbabbe3.png)

just copy the DNS NAME under external ip we will be able to see our Nginx server webpage.

![image](https://user-images.githubusercontent.com/24311808/145249054-4c6c46e6-efdb-41b0-ad34-6c41e444306e.png)

# App Mesh

To install app mesh we need to run below commands:

## App Mesh we will install via helm so install helm before go ahead.

### Add EKS Helm Repo 
```
helm repo add eks https://aws.github.io/eks-charts
```
## Install App Mesh Controller
### Create the namespace
``` kubectl create ns appmesh-system ```

### Install the App Mesh CRDs
``` kubectl apply -k "github.com/aws/eks-charts/stable/appmesh-controller//crds?ref=master" ```

### Create your OIDC identity provider for the cluster
```
eksctl utils associate-iam-oidc-provider \
  --cluster eksworkshop-eksctl \
  --approve
```
### Download the IAM policy for AWS App Mesh Kubernetes Controller
``` curl -o controller-iam-policy.json https://raw.githubusercontent.com/aws/aws-app-mesh-controller-for-k8s/master/config/iam/controller-iam-policy.json ```

### Create an IAM policy called AWSAppMeshK8sControllerIAMPolicy
```
aws iam create-policy \
    --policy-name AWSAppMeshK8sControllerIAMPolicy \
    --policy-document file://controller-iam-policy.json
```
### Create an IAM role for the appmesh-controller service account
```
eksctl create iamserviceaccount --cluster eksworkshop-eksctl \
    --namespace appmesh-system \
    --name appmesh-controller \
    --attach-policy-arn arn:aws:iam::369003951333:policy/AWSAppMeshK8sControllerIAMPolicy  \
    --override-existing-serviceaccounts \
    --approve
```
### Install App Mesh Controller into the appmesh-system namespace
```
helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=us-east-1 \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller \
    --set tracing.enabled=true \
    --set tracing.provider=x-ray
```
### validate if everything is installed, if it shows all the pods and deployment for appmesh controller then it is installed properly
```
kubectl get crds | grep appmesh
```
### We need add lables to our namespace  so automatic sidecar injection for pods can run.
#### we have a file mesh.yaml in mesh folder just apply it with kubectl

once sidecar injection is enabled we need to simply restart our existing container, envoy proxy as sidecar will start to run. then we need to create vitual node , virtual service and virtual router for our nginx apps to support canary releases. TO create canary deployment switch to **canary apps** and run ``` kubectl apply -f . ``` , if it is completed sucessfully then we will be able to see 2 versions for our nginx apps and with Virtual Gateway LB will automatcally forward the traffic according the threshold we have defined in **virtual router** manifest.






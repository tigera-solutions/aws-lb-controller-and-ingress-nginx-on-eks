# aws-lb-controller-and-ingress-nginx-on-eks

Example configuration for aws-load-balancer-controller and ingress-nginx controller on EKS with Calico CNI

## create EKS cluster with Calico CNI

>This example uses Cloud Shell to simplify environment configuration.

### provision EKS cluster

- login to AWS Console and open Cloud Shell.

- create EKS cluster using `eksctl`

>download [eksctl CLI](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) and [kubectl CLI](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

```bash
# set vars
export CLUSTER_NAME=eks-ingress-example
export AWS_REGION=us-west-2
export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))
EKS_VERSION="1.29"

# create EKS manifest file
cat > configs/tigera-workshop.yaml << EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: "${CLUSTER_NAME}"
  region: "${AWS_REGION}"
  version: "${EKS_VERSION}"

iam:
  withOIDC: true

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

# create cluster without default AWS VPC CNI plugin
addonsConfig:
  disableDefaultAddons: true
addons:
  - name: kube-proxy
  - name: coredns
  - name: aws-ebs-csi-driver

# enable all of the control plane logs:
cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
EOF

# create EKS cluster from the manifest file
eksctl create cluster -f configs/tigera-workshop.yaml
```

### install Calico

Follow Calico Enterprise or Calico Cloud official docs to install Calico on EKS cluster.

### create EKS nodegroup

```bash
# create node groups manifest
cat > configs/tigera-workshop-nodegroups.yaml << EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: "${CLUSTER_NAME}"
  region: "${AWS_REGION}"
  version: "${EKS_VERSION}"

# addonsConfig:
#   disableDefaultAddons: true

managedNodeGroups:
- name: "nix-t3-large"
  desiredCapacity: 3
  # choose proper size for worker node instance as the node size detemines the number of pods that a node can run
  # it's limited by a max number of interfeces and private IPs per interface
  # t3.large has max 3 interfaces and allows up to 12 IPs per interface, therefore can run up to 36 pods per node
  # see: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI
  instanceType: "t3.large"
  #ssh:
    # set KEYPAIR_NAME var and uncomment lines below to allow SSH access to the nodes using existing EC2 key pair
    #publicKeyName: ${KEYPAIR_NAME}
    #allow: true
EOF

# create EKS nodegroup
eksctl create nodegroup -f configs/tigera-workshop-nodegroups.yaml 
```

Once nodegroup is created, check Calico installation status

```bash
kubectl get tigerastatus
```

## configure `aws-load-balancer-controller`

References:

- review [example configuration](https://aws.amazon.com/blogs/containers/exposing-kubernetes-applications-part-3-nginx-ingress-controller/) of AWS LB controller with ingress-nginx controller.
- review [ALB ingress configuration](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html) article.

### configure IAM OIDC identity provider

```bash
# set vars
CLUSTER_NAME=eks-ingress-example
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
##############################
# create IAM Role using eksctl
##############################
## you ONLY need to create an IAM Role for the AWS Load Balancer Controller one per AWS account. Check if AmazonEKSLoadBalancerControllerRole exists in the IAM Console

# check if AmazonEKSLoadBalancerControllerRole role exists
aws iam get-role --role-name AmazonEKSLoadBalancerControllerRole

############################
# if the role doesn't exists
############################
## download IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
## create IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

## If you view the policy in the AWS Management Console, the console shows warnings for the ELB service, but not for the ELB v2 service. This happens because some of the actions in the policy exist for ELB v2, but not for ELB. You can ignore the warnings for ELB.

## create IAM role
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### install `aws-load-balancer-controller`

>this step uses [Helm](https://helm.sh/docs/intro/install/) to install the `aws-load-balancer-controller`.

```bash
# add the eks-charts Helm chart repository
helm repo add eks https://aws.github.io/eks-charts
# update your local repo to make sure that you have the most recent charts
helm repo update eks

# adjust vpcId value in the helm-values.yaml
VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text)
sed -i "s/vpcId:.*$/vpcId: ${VPC_ID}/g" configs/aws-load-balancer-controller/helm-values.yaml

# install the aws lb controller chart
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller -f configs/aws-load-balancer-controller/helm-values.yaml

# verify that the controller is installed
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## configure `ingress-nginx` controller

>this step uses [Helm](https://helm.sh/docs/intro/install/) to install the `aws-load-balancer-controller`.

### install `ingress-nginx` helm chart

```bash
# add ingress-nginx Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# install ingress-nginx
helm install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx -n ingress-nginx --create-namespace -f configs/ingress-nginx/helm-values.yaml

# verify ingress-nginx deployment
kubectl -n ingress-nginx get all

# when using ingress-nginx controller in host network mode with its service as NodePort,
# for testing open the NodePort in the AWS SecurityGroup for your VPC

NGINX_SVC_NP=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -ojsonpath='{.spec.ports[?(@.name=="http")].nodePort}')

# get SG
SG_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=*$CLUSTER_NAME*" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[*].NetworkInterfaces[0].Groups[0].GroupId' --output text)

# open nginx service NodePort in SG
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port $NGINX_SVC_NP --cidr 0.0.0.0/0
```

### deploy demo app

>when using `ingress-nginx` in host networked mode that requires its service to be NodePort, you should use a NodePort service for the application and reference it in the Ingress resource.

```bash
# deploy app
kubectl apply -f configs/demo/demo.app.deploy.yaml
kubectl apply -f configs/demo/demo.svc.np.yaml
kubectl apply -f configs/demo/demo.svc.yaml

# configure Ingress resource for the app fronted by ALB
kubectl apply -f configs/ingress-nginx/demo.ingress.alb.yaml

# you can use dig tool to find out IPs behind the LB created when you deploy the Ingress resource
# test access via ingress
LB_IP=<EC2_pub_IP_or_LB_IP>
curl http://www.demo.io --resolve www.demo.io:80:$LB_IP

# you should see this response if it works correctly
## <html><body><h1>It works!</h1></body></html>
```

## clean up

```bash
# delete ingress resource and demo app
kubectl delete -f configs/ingress-nginx/demo.ingress.alb.yaml
kubectl delete -f configs/demo/demo.svc.np.yaml
kubectl delete -f configs/demo/demo.svc.yaml
kubectl delete -f configs/demo/demo.app.deploy.yaml

# uninstall ingress-nginx helm chart
helm uninstall ingress-nginx -n ingress-nginx

# uninstall aws-load-balancer-controller helm chart
helm uninstall -n kube-system aws-load-balancer-controller

# destroy EKS cluster
eksctl delete cluster -f configs/tigera-workshop.yaml
```

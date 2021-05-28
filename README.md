### Set wordpress and mysql on AWS EKS
### CI (Continuous Integration) & CD (Continuous Delivery & Continuous Deployment) with Git, Dockerhub, ArgoCD

---

# Setting Cluster by Clusterconfig yaml file

>setup Directory

## myeks.yaml


```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: myeks
  region: ap-northeast-2
  version: "1.18"

# AZ
availabilityZones: 
  - ap-northeast-2a
  - ap-northeast-2b
  - ap-northeast-2c
  - ap-northeast-2d

# IAM OIDC & Service Account
iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      wellKnownPolicies:
        awsLoadBalancerController: true
    - metadata:
        name: ebs-csi-controller-sa
        namespace: kube-system
      wellKnownPolicies:
        ebsCSIController: true
    - metadata:
        name: cluster-autoscaler
        namespace: kube-system
      wellKnownPolicies:
        autoScaler: true
    
# Managed Node Groups
managedNodeGroups:
  # On-Demand Instance
  - name: myng-1
    instanceType: t3.medium
    minSize: 2
    desiredCapacity: 3
    maxSize: 4
    availabilityZones:
      - ap-northeast-2a
      - ap-northeast-2b
      - ap-northeast-2c
      - ap-northeast-2d
    ssh:
      allow: true
      publicKeyPath: ~/.ssh/id_rsa.pub
    iam:
      withAddonPolicies:
        autoScaler: true
        albIngress: true
        cloudWatch: true
        ebs: true

# Fargate Profiles
fargateProfiles:
  - name: fg-1
    selectors:
    - namespace: dev
      labels:
        env: fargate

# CloudWatch Logging
cloudWatch:
  clusterLogging:
    enableTypes: 
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler
```
---
> ## create cluster    
```  
$ eksctl create -f myeks.yaml  
```   
---  
# Add-ons

### aws-load-balancer-contoller

> https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

```sh
# apply target group binding custom resource
$ kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

# add eks-charts repository
$ helm repo add eks https://aws.github.io/eks-charts

# update local repo
$ helm repo update

# install aws load balancer contoller
# why image tag v2.1.3 = latest version has bug, not working
$ helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=spcluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.tag=v2.1.3 \
  -n kube-system

# verify
$ kubectl get deployment -n kube-system aws-load-balancer-controller
```

> **Could not find webhook error while using load balancer**

```sh
# Since we don't use webhooks, we can solve it by deleting webhook configuration.
$ kubectl delete validatingwebhookconfiguration aws-load-balancer-webhook
```

## EBS CSI Driver

> https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html

```sh
# add repo
$ helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

# repo update
$ helm repo update

# install aws ebs csi driver
$ helm upgrade -install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true \
  --set serviceAccount.controller.create=false \
  --set serviceAccount.controller.name=ebs-csi-controller-sa
```

## Metrics Server

> https://docs.amazonaws.cn/en_us/eks/latest/userguide/metrics-server.html

```sh
# Deploy the metrics server
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# verify
$ kubectl get deployment metrics-server -n kube-system
```

## Cluster Autoscaler

> https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html

```sh
# get cluster-autoscaler-autodiscover.yaml file
$ curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

```
> ### **edit** cluster-autoscaler-autodiscover.yaml 

```
command:
- ./cluster-autoscaler
- --v=4
- --stderrthreshold=info
- --cloud-provider=aws
- --skip-nodes-with-local-storage=false
- --expander=least-waste
- --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/**<YOUR CLUSTER NAME>**


```
# deploy
$ kubectl create -f cluster-autoscaler-autodiscover.yaml

# verify
$ kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler

## CloudWatch Container Insights

> https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-quickstart.html

```sh
# set variables
$ ClusterName=spcluster
$ RegionName=ap-northeast-2
$ FluentBitHttpPort='2020'
$ FluentBitReadFromHead='Off'
$ [[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
$ [[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'

# deploy
$ curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f - 

# verify
$ kubectl get po -n amazon-cloudwatch
```
# Create an Amazon EKS Cluster

## Create the cluster using the [ekctl](https://eksctl.io/) CLI

### Step 0: Prerequisites

1. Install and configure AWS CLI 

2. Install jq for your operating system
(https://stedolan.github.io/jq/download/)



### Step 1: Download the ekctl CLI, Create an EKS Cluster, Install Kubernetes Dashboard

``` 
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp


sudo mv -v /tmp/eksctl /usr/local/bin

```

Alternatively, macOS users can use Homebrew:

```
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```
and Windows users can use chocolatey:

```
chocolatey install eksctl
```

Confirm the ekctl command works by executing the following command

```
eksctl version
```

Create an EKS cluster with six nodes using the following ekctl command

```
eksctl create cluster --name=kf-sm-workshop --nodes=6 --managed --alb-ingress-access --region=us-west-2
```

Confirm you can see the cluster nodes. You should see six nodes

```
kubectl get nodes

```
Install Kubernetes Dashboard

The kubernetes dashboard is not deployed by default. Use the following command to deploy the dashboard

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

```
Since this is deployed to our private EKS cluster, we need to access it via a proxy. Kube-proxy is available to proxy our requests to the dashboard service

```
kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &

```

This will start the proxy, listen on port 8080, listen on all interfaces, and will disable the filtering of non-localhost requests.

This command will continue to run in the background of the current terminal’s session.

You can access the dashboard using the URL

```
http://localhost:8080/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

```

For the token, use the output from the following command 

```
aws eks get-token --cluster-name kf-sm-workshop | jq -r '.status.token'

```

Copy the output of this command and then click the radio button next to Token then in the text field below paste the output from the last command and click sign in.




  




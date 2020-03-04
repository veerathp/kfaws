## Install Kubeflow on Amazon EKS


### Download the kfctl v1.0 release:

```
curl --silent --location "https://github.com/kubeflow/kfctl/releases/download/v1.0/kfctl_v1.0-0-g94c35cf_darwin.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/kfctl /usr/local/bin
```


Create environment variables to make the deployment process easier:

```
export PATH=$PATH:/Users/veerathp/dev/kubeflow/kfctl
```

Use the following kfctl configuration file for the standard AWS setup:

```
export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v1.0-branch/kfdef/kfctl_aws.v1.0.0.yaml"
  
```

Set an environment variable for your AWS cluster name, and set the name of the Kubeflow deployment to the same as the cluster name

```
export AWS_CLUSTER_NAME=kf-sm-workshop
export KF_NAME=${AWS_CLUSTER_NAME}
```

Set the path to the base directory where you want to store one or more Kubeflow deployments 
Then set the Kubeflow application directory for this deployment:

```
export BASE_DIR=/Users/<usr>/dev/kubeflow
export KF_DIR=${BASE_DIR}/${KF_NAME}
```

### Setup your Kubeflow configuration

Download your configuration files, so that you can customize the configuration before deploying Kubeflow:

```
mkdir -p ${KF_DIR}
cd ${KF_DIR}
wget -O kfctl_aws.yaml $CONFIG_URI
export CONFIG_FILE=${KF_DIR}/kfctl_aws.yaml
```

### Setup your Kubeflow configuration

Replace the AWS cluster name in your ${CONFIG_FILE} file, by changing the value kubeflow-aws to ${AWS_CLUSTER_NAME} in multiple locations in the file. For example, use this sed command:

```
sed -i'.bak' -e 's/kubeflow-aws/'"$AWS_CLUSTER_NAME"'/' ${CONFIG_FILE}
```
Retrieve the AWS Region and IAM role name for your worker nodes. To get the IAM role name for your Amazon EKS worker node, run the following command:

```
aws iam list-roles \
    | jq -r ".Roles[] \
    | select(.RoleName \
    | startswith(\"eksctl-$AWS_CLUSTER_NAME\") and contains(\"NodeInstanceRole\")) \
    .RoleName"
```

Note: The above command assumes that you used eksctl to create your cluster. If you use other provisioning tools to create your worker node groups, find the role that is associated with your worker nodes in the Amazon EC2 console.

Change cluster region and worker role names in your ${CONFIG_FILE} file:

Note: If you have multiple node groups, you will see corresponding number of node group roles. In that case, please provide the role names as an array.

### Deploy Kubeflow

Run the following commands to initialize the Kubeflow cluster:

```
cd ${KF_DIR}
kfctl apply -V -f ${CONFIG_FILE}
```

Important!!! By default, these scripts create an AWS Application Load Balancer for Kubeflow that is open to public. This is good for development testing and for short term use, but we do not recommend that you use this configuration for production workloads.

To secure your installation, Follow the instructions to add authentication and authorization.

Wait for all the resources to become ready in the kubeflow namespace. Run the command below to check the status

```
kubectl get pods -n kubeflow
```
### Kubeflow Dashboard

Get Kubeflow service endpoint:

```
kubectl get ingress -n istio-system -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'

```
First time when you login, Click on Start Setup and then specify a namespace (eg. kf-sm-workshop)

Click finish to view the dashboard

### Jupyter notebook on Kubeflow

In the Kubeflow dashboard, click on Create a new Notebook server:

In the quick shortcuts, click on the "Create a Notebook server" link an select the namespace created above, provide the required details and click launch.

Once the notebook server is created, Click on CONNECT. This opens the Jupyter notebook interface in a new browser tab.

### Setup AWS credentials in EKS cluster

These credentials are stored in EKS cluster as Kubernetes secrets.

Create an IAM user ‘kf-s3user’, attach S3 access policy and retrieve temporary credentials


```
aws iam create-user --user-name kf-s3user
aws iam attach-user-policy --user-name kf-s3user --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam create-access-key --user-name s3user > /tmp/create_output.json
```

Next, save the new user’s credentials into  environment variables:

```
export AWS_ACCESS_KEY_ID_VALUE=$(jq -j .AccessKey.AccessKeyId /tmp/create_output.json | base64)
export AWS_SECRET_ACCESS_KEY_VALUE=$(jq -j .AccessKey.SecretAccessKey /tmp/create_output.json | base64)
```

Create the kubernetes secret:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: aws-secret
type: Opaque
data:
  AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID_VALUE
  AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY_VALUE
EOF

```

Create an S3 bucket to store training data:

```
export S3_BUCKET=pv-kf-workshop-data
export AWS_REGION=us-west-2
aws s3 mb s3://$S3_BUCKET --region $AWS_REGION
```

Run training using pod

```
kubectl create -f mnist-training.yaml
```







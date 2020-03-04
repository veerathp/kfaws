## Amazon SageMaker Operators for Kubernetes

Amazon SageMaker Operators for Kubernetes make it easier for developers and data scientists using Kubernetes to train, tune, and deploy machine learning models in Amazon SageMaker. You can install these SageMaker Operators on your Kubernetes cluster in Amazon Elastic Kubernetes Service (EKS) to create SageMaker jobs natively using the Kubernetes API and command-line Kubernetes tools such as ‘kubectl’. 

### Operator Deployment

Create an OpenID Connect Provider for Your Cluster

```
# Set the Region and cluster
export CLUSTER_NAME="<your cluster name>"
export AWS_REGION="<your region>"

```
Use the following command to associate the OIDC provider with your cluster.

```
eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} \
    --region ${AWS_REGION} --approve
```

Get the OIDC ID
To set up the ServiceAccount, first obtain the OpenID Connect issuer URL using the following command:

```
aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} \
    --query cluster.identity.oidc.issuer --output text
```

The command will return a URL like the following:

```
https://oidc.eks.${AWS_REGION}.amazonaws.com/id/D48675832CA65BD10A532F597OIDCID
```
In this URL, the value D48675832CA65BD10A532F597OIDCID is the OIDC ID. The OIDC ID for your cluster will be different. You need this OIDC ID value to create a role.

Create an IAM Role

Create a file named trust.json and insert the following trust relationship code block into it. Be sure to replace all OIDC ID, AWS account number, and EKS Cluster region placeholders with values corresponding to your cluster.


```

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS account number>:oidc-provider/oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>:aud": "sts.amazonaws.com",
          "oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>:sub": "system:serviceaccount:sagemaker-k8s-operator-system:sagemaker-k8s-operator-default"
        }
      }
    }
  ]
}

```
Run the following command to create a role with the trust relationship defined in trust.json. This role enables the Amazon EKS cluster to get and refresh credentials from IAM.

```
aws iam create-role --role-name <role name> --assume-role-policy-document file://trust.json --output=text
```

Take note of ROLE ARN, you pass this value to your operator.

Attach the AmazonSageMakerFullAccess Policy to the Role

To give the role access to Amazon SageMaker, attach the AmazonSageMakerFullAccess policy. If you want to limit permissions to the operator, you can create your own custom policy and attach it.

To attach AmazonSageMakerFullAccess, run the following command:

```
aws iam attach-role-policy --role-name <role name>  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess

```

The Kubernetes ServiceAccount sagemaker-k8s-operator-default should have AmazonSageMakerFullAccess permissions. Confirm this when you install the operator.


#### Deploy the Operator Using YAML

Download the installer script using the following command:

```
wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/release/rolebased/installer.yaml

```

Edit the installer.yaml file to replace eks.amazonaws.com/role-arn. Replace the ARN here with the Amazon Resource Name (ARN) for the OIDC-based role you’ve created.

Use the following command to deploy the cluster:

```
kubectl apply -f installer.yaml

```

Verify the operator deployment

```
kubectl get crd | grep sagemaker

```
Ensure that the operator pod is running successfully. Use the following command to list all pods:

```
kubectl -n sagemaker-k8s-operator-system get pods

```




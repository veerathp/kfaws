### Setup AWS credentials in Kubeflow Namespace

Create an IAM user ‘kf-smuser', attach S3 access policy and retrieve temporary credentials


```
aws iam create-user --user-name kf-smuser
aws iam attach-user-policy --user-name kf-smuser --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam create-access-key --user-name kf-smuser > /tmp/create_output.json
```

Next, save the new user’s credentials into  environment variables:

```
export AWS_ACCESS_KEY_ID_VALUE=$(jq -j .AccessKey.AccessKeyId /tmp/create_output.json | base64)
export AWS_SECRET_ACCESS_KEY_VALUE=$(jq -j .AccessKey.SecretAccessKey /tmp/create_output.json | base64)
```

Create the kubernetes secret in the kubeflow namespace:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: aws-secret
  namespace: kubeflow
type: Opaque
data:
  AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID_VALUE
  AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY_VALUE
EOF

```

Create SageMaker execution Role:

```
TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"Service\": \"sagemaker.amazonaws.com\" }, \"Action\": \"sts:AssumeRole\" } ] }"
aws iam create-role --role-name kfworkshop-sagemaker-kfp-role --assume-role-policy-document "$TRUST"
aws iam attach-role-policy --role-name kfworkshop-sagemaker-kfp-role --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam attach-role-policy --role-name kfworkshop-sagemaker-kfp-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam get-role --role-name kfworkshop-sagemaker-kfp-role --output text --query 'Role.Arn'

```
Assign sagemaker:InvokeEndpoint permission to the Worker node IAM role

```
cat <<EoF > sagemaker-invoke.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sagemaker:InvokeEndpoint"
            ],
            "Resource": "*"
        }
    ]
}
EoF
```

```
aws iam put-role-policy --role-name $ROLE_NAME --policy-name sagemaker-invoke-for-worker --policy-document file://sagemaker-invoke.json

```







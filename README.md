# Velero
This document describes how Velero works with Hyperstore
### Prerequisites
1.Access to a Kubernetes cluster, version 1.7 or later. Note: restic support requires Kubernetes version 1.10 or later, or an earlier version with the mount propagation feature enabled.<br>
For example,install Minikube
```
 $ brew install minikube
 $ minikube start --kubernetes-version v1.17.0
```
2. A DNS server on the cluster <br>
Modify the config-map of coreDNS
```
$ kubectl edit cm coredns -n kube-system 
```
Add the following code after the block of kubernetes cluster.local in-addr.arpa ip6.arpa
```
hosts {
           [Hyperstore ip] [Hyperstore s3Url]

           fallthrough
        }
```
For example:
```
hosts {
           10.10.3.203 s3-region1.cloudian.com

           fallthrough
        }
 ```
3. Modify host 


### Installing
1. Create a Velero-specific credentials file (credentials-velero) in your local directory:
```
 [default]
aws_access_key_id = [hyperstore aws_access_key_id]
aws_secret_access_key = [hyperstore ws_secret_access_key]
```

2. Run
```
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.0.0 \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=region1,s3ForcePathStyle="true",s3Url=http://s3-region1.cloudian.com:80
```

3. Deploy the example nginx application:
``` 
 kubectl apply -f examples/nginx-app/base.yaml
```
4. Check to see that both the Velero and nginx deployments are successfully created:
``` 
 kubectl get deployments -l component=velero --namespace=velero
 kubectl get deployments --namespace=nginx-example
```
### Administrator Creates Secret
This secret will contain the elevated/admin privileges needed by the provisioner to properly access and create S3 Buckets and IAM users and policies. The AWS Access ID and AWS Secret Key will be needed for this.<br>
1. Create the Kubernetes Secret for the Provisioner's Owner Access.
``` 
apiVersion: v1
kind: Secret
metadata:
  name: s3-bucket-owner [1]
  namespace: s3-provisioner [2]
type: Opaque
data:
  AWS_ACCESS_KEY_ID: *base64 encoded value* [3]
  AWS_SECRET_ACCESS_KEY: *base64 encoded value* [4]
``` 
[1] Name of the secret, this will be referenced in StorageClass.<br>
[2] Namespace where the Secret will exist.<br>
[3] Your AWS_ACCESS_KEY_ID base64 encoded. Encode your AWS_ACCESS_KEY_ID of your AWS account from https://www.base64encode.org/ <br>
[4] Your AWS_SECRET_ACCESS_KEY base64 encoded. Encode your AWS_SECRET_ACCESS_KEY of your AWS account from https://www.base64encode.org/ <br>
Then create secret/s3-bucket-owner
``` 
# kubectl create -f creds.yaml
secret/s3-bucket-owner created
``` 
2. Administrator Creates StorageClass
The StorageClass defines the name of the provisioner and holds other properties that are needed to provision a new bucket, including the Owner Secret and Namespace, and the AWS Region.<br>
Create the Kubernetes StorageClass for the Provisioner.
``` 
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: s3-buckets [1]
provisioner: aws-s3.io/bucket [2]
parameters:
  region: us-west-1 [3]
  secretName: s3-bucket-owner [4]
  secretNamespace: s3-provisioner [5]
reclaimPolicy: Delete [6]
``` 
[1] Name of the StorageClass, this will be referenced in the User ObjectBucketClaim. <br>
[2] Provisioner name <br>
[3] AWS Region that the StorageClass will serve <br>
[4] Name of the bucket owner Secret created above <br>
[5] Namespace where the Secret will exist <br>
[6] reclaimPolicy (Delete or Retain) indicates if the bucket can be deleted when the OBC is deleted.<br>
NOTE: the absence of the bucketName Parameter key in the storage class indicates this is a new bucket and its name is based on the bucket name fields in the OBC.
``` 
# kubectl create -f storageclass-greenfield.yaml
storageclass.storage.k8s.io/s3-buckets created
``` 
3. User Creates ObjectBucketClaim
An ObjectBucketClaim follows the same concept as a PVC, in that it is a request for Object Storage, the user doesn't need to concern him/herself with the underlying storage, just that they need access to it. The user will work with the cluster/storage administrator to get the proper StorageClass needed and will then request access via the OBC.
Create the ObjectBucketClaim.
``` 
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: myobc [1]
  namespace: s3-provisioner [2]
spec:
  generateBucketName: mybucket-prefix- [3]
  bucketName: my-awesome-bucket [4]
  storageClassName: s3-buckets [5]
  additionalConfig:
    additionalProperties: test[6]
 ``` 
[1] Name of the OBC <br>
[2] Namespace of the OBC <br>
[3] Name prepended to a random string used to generate a bucket name. It is ignored if bucketName is defined <br>
[4] Name of new bucket which must be unique across all AWS regions, otherwise an error occurs when creating the bucket. If present, this name overrides generateName <br>
[5] StorageClass name <br>
[6] set any object name here <br>
```diff
- NOTE: Only remain either generateBucketName or bucketName in this document, otherwise error will occur.<br>
- BucketName must be globally unique.
```
Then use the following command
```
# kubectl create -f obc-brownfield.yaml
objectbucketclaim.objectbucket.io/myobc created
```

### Results and Recap
Let's pause for a moment and digest what just happened. After creating the OBC, and assuming the S3 provisioner is running, we now have the following Kubernetes resources: . a global ObjectBucket (OB) which contains: bucket endpoint info (including region and bucket name), a reference to the OBC, and a reference to the storage class. Unique to S3, the OB also contains the bucket Amazon Resource Name (ARN).Note: there is always a 1:1 relationship between an OBC and an OB. . a ConfigMap in the same namespace as the OBC, which contains the same endpoint data found in the OB. . a Secret in the same namespace as the OBC, which contains the AWS key-pairs needed to access the bucket.
And of course, we have a new AWS S3 Bucket which you should be able to see via the AWS Console.
ObjectBucket<br>
The installation status can be confirmed in two ways:
1. Login in your AWS account to confirm whether new bucket has been created.
2. using command line to check thether new bucket has been created.
```
 $ cd .aws/
 $ vi config
 (change AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY to your AWS account)
 $ aws s3 ls
```
### Uninstalling
If you have installed minikube using the direct download method, follow the below steps to uninstall minikube completely from your system <br>
- In the shell, type in 
```
$ minikube delete 
```
to delete the minikube cluster. <br>
- Uninstall the minikube package using 
```
$ brew uninstall minikube 
```
- Remove the directory containing the minikube configuration 
```
$ rm -rf ~/.minikube
```
### Useful commands
Check log status
```
$ kubectl log aws-s3-provisioner-deployment-94987d57-dsjxc -n s3-provisioner
```
Check deployment status
```
$ kubectl get deployment -n s3-provisioner
```
Check pod status
```
$ kubectl get pod -n s3-provisioner
```

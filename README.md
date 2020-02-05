# Velero
This document describes how Velero works with Hyperstore
### Introduction
Velero (formerly Heptio Ark) gives you tools to back up and restore your Kubernetes cluster resources and persistent volumes. You can run Velero with a cloud provider or on-premises. Velero lets you:<br>

Take backups of your cluster and restore in case of loss.<br>
Migrate cluster resources to other clusters.<br>
Replicate your production cluster to development and testing clusters.<br>
Velero consists of:<br>
A server that runs on your cluster<br>
A command-line client that runs locally<br>
### Prerequisites
1. Access to a Kubernetes cluster, version 1.7 or later. Note: restic support requires Kubernetes version 1.10 or later, or an earlier version with the mount propagation feature enabled.<br>
For example,install Minikube in your local machine
```
 brew install minikube
 minikube start --kubernetes-version v1.17.0
```
2. A DNS server on the cluster <br>
You can modify the config-map of coreDNS
```
 kubectl edit cm coredns -n kube-system 
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
3. Create a bucket named velero in Hyperstore.

### Installing
1. Create a Velero-specific credentials file (credentials-velero) in your local directory:
```
[default]
aws_access_key_id = [hyperstore aws_access_key_id]
aws_secret_access_key = [hyperstore ws_secret_access_key]
```
You can get [hyperstore aws_access_key_id] and [hyperstore ws_secret_access_key] from [user]-[Security Credentials]-[Service Information] in Hyperstore.<br>
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
### Back up
1. First confirm the current namespaces in K8s and you can find a namespace named nginx-example
``` 
kubectl get ns
``` 
2. Create a backup for any object that matches the app=nginx label selector:
``` 
velero backup create nginx-backup --selector app=nginx
``` 
After running this command, a folder named backups is expected to be shown in Hyperstore velero bucket.
![Image workflow](https://github.com/hyu-cloudian/Velero/blob/master/backup_and_restore.png)

3. Simulate a disaster
``` 
kubectl delete namespace nginx-example
``` 
If you run get namespaces again, you can find the namespace named nginx-example has been deleted.
``` 
kubectl get ns
``` 
### Restore
1.Run:
```
velero restore create --from-backup nginx-backup
```
After runing this command, a folder named restores will appear in Hyperstore velero bucket.
Then confirm the namespaces again, you will find the namespace named nginx-example has been restored
```
kubectl get ns
```
![Image workflow](https://github.com/hyu-cloudian/Velero/blob/master/command.png)
 

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
### Back up
1. Create a backup for any object that matches the app=nginx label selector:
``` 
velero backup create nginx-backup --selector app=nginx
``` 
2. Simulate a disaster
``` 
kubectl delete namespace nginx-example
``` 
### Restore
Run:
```
velero restore create --from-backup nginx-backup
```
Run:
```
velero restore get
```

## Create kubernet clusters on AWS 

before using this quick start sample script, make sure you set up following instruction:
* Pre-setups (aws eksctl, kubectl, aws ctl) 
https://docs.aws.amazon.com/eks/latest/userguide/setting-up.html

* VPC and subnets (monitor on AWS cloudformation): pay attention to the role creating and policies
https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html
https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html
https://docs.aws.amazon.com/vpc/latest/userguide/vpc-policy-examples.html#vpc-public-subnet-iam

* EKS cluster: pay attention to the role creating and policies
https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html

I use user account to do the work, so I created customized policy for my account permission. Here is the IAM json I used for this porject 
(see in the json file in the repo)
* custome inline policy: myAmazonEKSploicy
* custome inline policy: EKS-cloudformation
* custome inline policy: EKS-kubernets


After attaching required policy read the quickstart and run the sample config 
refer to: https://docs.aws.amazon.com/eks/latest/userguide/quickstart.html

remember to change the create namespace command as:
`$ kubectl create namespace defualt --save-config`

# Instructions for Labelstudio EKS practice after installing labelstudio
For user who installed labelstudio and want to bound it to EKS service.

If you want to re-set the pvc and ingress, you can also refer to the following parts:
* Define storage class before install: **Define a Storage Class**
  Create the storage class pg2 (or any other name) then install labelstudio
  If the pod is still pending and pvc is still not bounding check: **Update PVC to Use the Storage Class**
    
* connect labelstudio to load balancer controller: **create ingress resource**
  


## Set up persistent volume claim for labelstudio 

### get the status of the pvc that labelstudio created
`$ kubectl get pvc -n default`
if not set the pvc the result looks like this: 
```
NAME                            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-labelstudio-postgresql-0   Pending                                                     <unset>                 13h
labelstudio-ls-pvc              Pending                                                     <unset>                 13h
```
`data-labelstudio-postgresql-0` and `labelstudio-ls-pvc` are your pvc names

The Pending status of your Persistent Volume Claims (PVCs) indicates that Kubernetes is unable to bind the PVCs to a Persistent Volume (PV). This could be due to several reasons, such as:
1. No Storage Class Defined: If a Storage Class is not defined, Kubernetes might not know how to provision the storage.
2. Insufficient Resources: There might not be enough resources available to satisfy the storage request.
3. Misconfiguration: The PVC might be configured incorrectly.

### Define a Storage Class
If your PVCs do not specify a storageClassName, 
Kubernetes will attempt to use the default storage class. 
However, if no default storage class is configured or 
if your cluster setup requires a specific Storage Class, you need to define one.
Create a Storage Class, for example, using the gp2 EBS volume type:

storage-class.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
reclaimPolicy: Retain
volumeBindingMode: Immediat
```
#### apply storagaclass 
`$ kubectl apply -f storage-class.yaml`

### Update PVC to Use the Storage Class
Update your PVC manifests to use the defined gp2 Storage Class
pvc for labelstudio main instance (labelstudio-ls-pvc)
example pvc-labelstudio.yaml (replace pvc name with your own pvc name)
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: labelstudio-ls-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp2 
```

pvc for labelstudio datastorage pvc (data-labelstudio-postgresql-0) 
example pvc-postgresql.yaml (replace pvc name with your own pvc name)
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-labelstudio-postgresql-0
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp2 
```

`$ kubectl apply -f pvc-labelstudio.yaml`
`$ kubectl apply -f pvc-postgresql.yaml`

#### Verify the PVC and PV Status
`$ kubectl get pvc -n default`
`$ kubectl get pvc`

#### Delete pod if they are still not bound (Just in case)
With the PVCs bound, Label Studio pods should be able to start correctly. 
If they are still in a Pending state, redeploy the application:
example code: 
`$ kubectl delete pod labelstudio-ls-app-7d4657df77-xbp2w -n default `
`$ kubectl delete pod labelstudio-postgresql-0 -n default `
This will allow Kubernetes to reschedule the pods, 
which should now work because the storage is properly set up.

#### Monitor pods 
`$ kubectl get pods -n default `

check pod events sample command:
<pod-name> are the pod names get from command ` kubectl get pods -n default`
`$ kubectl describe pod <pod-name> -n default`
`$ kubectl describe pod <pod-name> -n default`

#### check pod log if need 
If the pods still face issues, check the pod descriptions and logs for more information:
`$ kubectl describe pod <pod-name> -n default`  
`$ kubectl logs <pod-name> -n default `

## Set up access link use AWS load balancer controller
labelstudio default setting is forward port to localhost:8080 
but we want other user can access the labelstudio 

### turn up the security variable 
```$ helm upgrade labelstudio heartex/label-studio \
--set global.extraEnvironmentVars.SSRF_PROTECTION_ENABLED="true" \
--namespace default ```

#### get the app's name that deploy on kubernet 
`$ kubectl get deployments -n default`

output sample: 
``` NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
labelstudio-ls-app   1/1     1            1           41h
```

#### resart the app make sure the variable is set:
```$ kubectl rollout restart deployment labelstudio-ls-app -n default
deployment.apps/labelstudio-ls-app restarted
```

#### confirm by running the flowing command 
`$ kubectl describe pod -n default | grep SSRF_PROTECTION_ENABLED `



### install AWS loadbalancer (for user installed labelstudio as first step)

```$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --namespace kube-system \
    --set clusterName=web-quickstart \
    --set serviceAccount.create=false \
    --set region=${CLUSTER_REGION} \
    --set vpcId=${CLUSTER_VPC} \
    --set serviceAccount.name=aws-load-balancer-controller
```

output sample: 
``` 
NAME: aws-load-balancer-controller
LAST DEPLOYED: Thu Aug 15 12:54:43 2024
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed! 
```

#### Ensure AWS Load Balancer Controller is Installed and Running:

`$ kubectl get pods -n kube-system `

output sample: 
```
NAME                                           READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-f7bbccbd7-fgjpl   1/1     Running   0          27m
aws-load-balancer-controller-f7bbccbd7-j5p8z   1/1     Running   0          27m
aws-node-j89j9                                 2/2     Running   0          39h
aws-node-zkdwc                                 2/2     Running   0          39h
coredns-858457f4f6-8szb9                       1/1     Running   0          39h
coredns-858457f4f6-jn5t4                       1/1     Running   0          39h
ebs-csi-controller-5f6f8c8f9b-954cl            6/6     Running   0          39h
ebs-csi-controller-5f6f8c8f9b-zjxpl            6/6     Running   0          39h
ebs-csi-node-4g88d                             3/3     Running   0          39h
ebs-csi-node-g7dsw                             3/3     Running   0          39h
kube-proxy-fb7fn                               1/1     Running   0          39h
kube-proxy-q9hcf                               1/1     Running   0          39h
```


### create ingress resource 

sample content in the labelstudio-ingress.yaml file:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: labelstudio-ingress
  namespace: default  # Replace with your namespace if it's different
  annotations:
    kubernetes.io/ingress.class: "alb"
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: labelstudio-ls-app  # Replace with your service name
              port:
                number: 80  
```

#### apply ingress resource 
`$ kubectl apply -f labelstudio-ingress.yaml `

#### Check the Ingress Status
the external address (DNS name) is assigned by the AWS Load Balancer: 

`$ kubectl get ingress labelstudio-ingress -n default `




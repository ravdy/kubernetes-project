### How to position tracker connects with Database.

position-tracker cant able to connect with database until release 2. to Enable this we need to use release 3 image. in release 3 configuration file updated to connect with database. Below is the link for configuration file. 

https://github.com/ravdy/k8s-fleetman/blob/master/microservice-source-code/release3/k8s-fleetman-position-tracker/src/main/resources/application-production-microservice.properties

Also we need create MongoDB pod. 

Update the manifest file 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue
spec:
  selector:
    matchLabels:
      app: queue
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: queue
    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-simulator
spec:
  selector:
    matchLabels:
      app: position-simulator
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: position-simulator
    spec:
      containers:
      - name: position-simulator
        image: richardchesterwood/k8s-fleetman-position-simulator:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-tracker
spec:
  selector:
    matchLabels:
      app: position-tracker
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: position-tracker
    spec:
      containers:
      - name: position-tracker
        image: richardchesterwood/k8s-fleetman-position-tracker:release3
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: richardchesterwood/k8s-fleetman-api-gateway:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
```

Once the above pod deployed position-tracker try to connect with the database and failed as we hadn't created the database pod yet. 

So create a new manifest file for MongoDB listens on 27017.  So we should open that port in the database. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.5-jessie

---
kind: Service
apiVersion: v1
metadata:
  name: fleetman-mongodb
spec:
  selector:
    app: mongodb
  ports:
    - name: mongoport
      port: 27017
  type: ClusterIP

```

After this, it starts tracking the history in the MongoDB if the position tracker still it contains the data. but if the MongoDB dies then we last the data. to overcome this, we need to store data in the persistent storage. 

## Persistent Volumes

Adding pv and pvc to the to the MongoDB 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.5-jessie
        volumeMounts:
          - name: mongo-persistent-storage
            mountPath: /data/db
      volumes:
        - name: mongo-persistent-storage
          # pointer to the configuration of HOW we want the mount to be implemented
          persistentVolumeClaim:
            claimName: mongo-pvc
---
kind: Service
apiVersion: v1
metadata:
  name: fleetman-mongodb
spec:
  selector:
    app: mongodb
  ports:
    - name: mongoport
      port: 27017
  type: ClusterIP
```

`aws eks update-kubeconfig --region us-east-1 --name xample-dev`

```yaml
# What do want?
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  storageClassName: mylocalstorage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
# How do we want it implemented
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-storage
spec:
  storageClassName: mylocalstorage
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/some-one/mangodb/"
    type: DirectoryOrCreate

```

to use EBS we need to install below drivers 

**you need to use the correct region in Step 1, and in each step you must replace YourClusterName with.. your cluster name!**

Step 1:
`eksctl utils associate-iam-oidc-provider --region=YOUR-REGION --cluster=YourClusterNameHere --approve`

example

```yaml
unix@vdesktop-4:~/terraform_eks/test$ eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=xample-dev -
-approve
2024-04-23 08:24:54 [ℹ]  will create IAM Open ID Connect provider for cluster "xample-dev" in "us-east-1"
2024-04-23 08:24:55 [✔]  created IAM Open ID Connect provider for cluster "xample-dev" in "us-east-1"
```

Step 2:
`eksctl create iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster xample-dev --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy --approve  --role-only  --role-name AmazonEKS_EBS_CSI_DriverRole`

Example

```yaml
unix@vdesktop-4:~/terraform_eks/test$ eksctl create iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster xample-dev --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy --approve  --r
ole-only  --role-name AmazonEKS_EBS_CSI_DriverRole
2024-04-23 08:26:07 [ℹ]  1 iamserviceaccount (kube-system/ebs-csi-controller-sa) was included (based on the include/exclude rules)
2024-04-23 08:26:07 [!]  serviceaccounts in Kubernetes will not be created or modified, since the option --role-only is used
2024-04-23 08:26:07 [ℹ]  1 task: { create IAM role for serviceaccount "kube-system/ebs-csi-controller-sa" }
2024-04-23 08:26:07 [ℹ]  building iamserviceaccount stack "eksctl-xample-dev-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa"
2024-04-23 08:26:07 [ℹ]  deploying stack "eksctl-xample-dev-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa"
2024-04-23 08:26:08 [ℹ]  waiting for CloudFormation stack "eksctl-xample-dev-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa"

2024-04-23 08:26:39 [ℹ]  waiting for CloudFormation stack "eksctl-xample-dev-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa"
```

Step 3:
`eksctl create addon --name aws-ebs-csi-driver --cluster YourClusterNameHere --service-account-role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/AmazonEKS_EBS_CSI_DriverRole --force`

Example

```yaml
unix@vdesktop-4:~/terraform_eks/test$ eksctl create addon --name aws-ebs-csi-driver --cluster xample-dev --service-account-role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/AmazonEKS_EBS_CSI_DriverRole --force
2024-04-23 08:27:33 [ℹ]  Kubernetes version "1.29" in use by cluster "xample-dev"
2024-04-23 08:27:34 [ℹ]  using provided ServiceAccountRoleARN "arn:aws:iam::671835412446:role/AmazonEKS_EBS_CSI_DriverRole"
2024-04-23 08:27:34 [ℹ]  creating addon
```

### EBS as a disk

```yaml
# What do want?
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  storageClassName: cloud-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 7Gi
---
# How do we want it implemented
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cloud-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

Nowe we need to change service from NodePort to ClusterIP or LoadBalancer. And no change in the workload.yaml file

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp

spec:
  # This defines which pods are going to be represented by this Service
  # The service becomes a network endpoint for either other services
  # or maybe external users to connect to (eg browser)
  selector:
    app: webapp

  ports:
    - name: http
      port: 80
      nodePort: 30080

  type: LoadBalancer
---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-queue

spec:
  # This defines which pods are going to be represented by this Service
  # The service becomes a network endpoint for either other services
  # or maybe external users to connect to (eg browser)
  selector:
    app: queue

  ports:
    - name: http
      port: 8161
      nodePort: 30010

    - name: endpoint
      port: 61616

  type: NodePort

---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-position-tracker

spec:
  # This defines which pods are going to be represented by this Service
  # The service becomes a network endpoint for either other services
  # or maybe external users to connect to (eg browser)
  selector:
    app: position-tracker

  ports:
    - name: http
      port: 8080

  type: ClusterIP

---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-api-gateway

spec:
  # This defines which pods are going to be represented by this Service
  # The service becomes a network endpoint for either other services
  # or maybe external users to connect to (eg browser)
  selector:
    app: api-gateway

  ports:
    - name: http
      port: 8080
      nodePort: 30020

  type: NodePort

```

### Readiness and Liveness Probes

To test readiness probs   change replicas from 1 to 3 in the api-gateway microservice 

```yaml
while true; do curl 3.90.228.118:30080/api; echo; done
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  replicas: 3
  template: # template for the pods
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: richardchesterwood/k8s-fleetman-api-gateway:performance
        readinessProbe:
          httpGet:
            path: /
            port: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        resources: 
          requests: 
            memory: 200Mi
            cpu: 50m
```

Now Liveness comes in to the picture when we have readiness in the picture. Actually Livenss restarts the container incase of any failure. 

### Horizontal Pod Autoscaling

now we can test the Horizontal pod autoscaling. we can test this on API Proxy microservice. To enable this we need to use api proxy image with  “performance” tag. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: richardchesterwood/k8s-fleetman-api-gateway:performance
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        resources: 
          requests: 
            memory: 200Mi
            cpu: 50m
```

We are creating autoscaling for deployment not for pod. for this we should create a hpa yaml file. same can be generated with help of a kubectl command. 

```yaml
kubectl autoscale deployment api-gateway --cup-percent 400 --min 1 --max 4 

```

### Ingress controller

Install nginx ingress controller using helm

install helm 

```jsx
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh 

chmod 700 get_helm.sh 

./get_helm.sh
```

Add the NGINX Ingress repository

```yaml
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install the NGINX Ingress Controller:

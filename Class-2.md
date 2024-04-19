# Setup 5 microservices 
### List of microservices
As part of this project, we are going to provision 5 microservices

1. Web service - Access the application
2. API Proxy
3. Position Tracker
4. Position Simulator
5. ActiveMQ
### webapp microservice as a pod 
Let's create a webapp pod 1st

```sh
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp

spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5
```

We can experiment it with “release0” and “release0-5”
### webapp service
to access this application we need service. below is the service manifest file 

```sh
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp

spec:
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      nodePort: 30080
  type: NodePort
```

now we can access this application from the browsers. but make sure that you have opened the 30080 port in the security group. 
### webapp microservice as a deployment 
in the current case, we are using the resource type as pod. If the pod is terminated our application goes down. to overcome this we can use the “deployment” resource. So adjust configuration like below 

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp

spec:
  selector:
    matchLabels:
      app: webapp

  replicas: 1
  template:
    metadata:
      labels:
        app: webapp

    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5
```

As we just changed the resource type for pod we can use same deployment. make sure your labels and selector are matching in the manifest files. 
### ActiveMQ
Now lets add another service to webapp that is ActiveMQ

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp

spec:
  selector:
    matchLabels:
      app: webapp

  replicas: 1
  template:
    metadata:
      labels:
        app: webapp

    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue

spec:
  selector:
    matchLabels:
      app: queue

  replicas: 1
  template:
    metadata:
      labels:
        app: queue

    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release1
```
### ActiveMQ service
We are using ActiveMQ service type as NodePort. it uses 2 ports, i.e 8161 and 61616. Later we are going to changes these to ClusterIP  

```sh
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp

spec:
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      nodePort: 30080
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-queue

spec:
  selector:
    app: queue
  ports:
    - name: http
      port: 8161
      nodePort: 30010
    - name: endpoint
      port: 61616
  type: NodePort
```
### Positon Simulator
Now we are adding another microservice that is position simulator (ps) this need environment variable please add this to the position simulator configuration 

> - name: SPRING_PROFILES_ACTIVE  
          value: production-microservice
> 

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp

spec:
  selector:
    matchLabels:
      app: webapp

  replicas: 1
  template:
    metadata:
      labels:
        app: webapp

    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue

spec:
  selector:
    matchLabels:
      app: queue

  replicas: 1
  template:
    metadata:
      labels:
        app: queue

    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release1

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
  template:
    metadata:
      labels:
        app: position-simulator

    spec:
      containers:
      - name: position-simulator
        image: richardchesterwood/k8s-fleetman-position-simulator:release1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
```

For Position Simulator (ps) service isn't required. So we can proceed with the next service 

### Position Tracker

We need to add a new microservice. That is the Position tracker (pt). For this also we need env variables

> - name: SPRING_PROFILES_ACTIVE  
          value: production-microservice
> 

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp

spec:
  selector:
    matchLabels:
      app: webapp

  replicas: 1
  template:
    metadata:
      labels:
        app: webapp

    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue

spec:
  selector:
    matchLabels:
      app: queue

  replicas: 1
  template:
    metadata:
      labels:
        app: queue

    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release1

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
  template:
    metadata:
      labels:
        app: position-simulator

    spec:
      containers:
      - name: position-simulator
        image: richardchesterwood/k8s-fleetman-position-simulator:release1
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
        image: richardchesterwood/k8s-fleetman-position-tracker:release1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
```

Let's add a service for the position tracker. 

```sh
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp

spec:
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      nodePort: 30080
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-queue

spec:
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
  selector:
    app: position-tracker

  ports:
    - name: http
      port: 8080
      nodePort: 30020

  type: NodePort
```

### API Proxy

Now let's add API proxy service

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp

spec:
  selector:
    matchLabels:
      app: webapp

  replicas: 1
  template:
    metadata:
      labels:
        app: webapp

    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue

spec:
  selector:
    matchLabels:
      app: queue

  replicas: 1
  template:
    metadata:
      labels:
        app: queue

    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release1

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
  template:
    metadata:
      labels:
        app: position-simulator

    spec:
      containers:
      - name: position-simulator
        image: richardchesterwood/k8s-fleetman-position-simulator:release1
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
        image: richardchesterwood/k8s-fleetman-position-tracker:release1
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
        image: richardchesterwood/k8s-fleetman-api-gateway:release1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
```

lets add a service for API proxy microservice 

```sh
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp

spec:
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      nodePort: 30080
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-queue

spec:
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
  selector:
    app: api-gateway

  ports:
    - name: http
      port: 8080
      nodePort: 30020

  type: NodePort
```

Now lets release the version2 as version one is unable to track vehicles route. developers fixed in the version2 so we need to update the take as “relase2” in all microservice. 

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp

spec:
  selector:
    matchLabels:
      app: webapp

  replicas: 1
  template:
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

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue

spec:
  selector:
    matchLabels:
      app: queue

  replicas: 1
  template:
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
  template:
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
        image: richardchesterwood/k8s-fleetman-position-tracker:release2
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
```

Next class we will be covering concepts like 

why do we need database

How to use PV and PVC 

How to store data in persistent volumes etc.

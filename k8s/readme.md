# Setting up AKS k8s Cluster
* [Refer here](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough) for setting up AKS azure CLI

# K8s Specifications

* Pod specification for Openmrscore

```
---
apiversion : v1
kind : Pod
metadata :
  name : openmrscore
  labels :
    app : openmrscore
    version : 2.4.0
spec :
  containers:
    - name : openmrscore 
      image : sunilkumardasu/openmrs-core2.4.0
      Ports :
        - containerport : 8080
          protocal : tcp
           
```

* Pod specification for mysql

```
---
apiversion : v1
kind : Pod
metadata :
  name : mysql
  labels :
    app: db
    ver: 5.6
spec :
  containers:
    - name : mysql 
      image : mysql:5.6
      Ports :
        - containerport : 3306
          protocal : tcp
      env:
       - name: 'MYSQL_ROOT_PASSWORD'
         value: 'password'
       - name: 'MYSQL_DATABASE'
         value: 'openmrs'
       - name: 'MYSQL_USER'
         value: 'sunildasu'
       - name: 'MYSQL_PASSWORD'
         value: 'sunildasu'  

```
* Replication Controller for openmrs will look like

```
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: openmrs-rc
  labels:
    app: openmrs
    ver: 2.4.0
spec:
  replicas: 3
  selector:
    app: openmrscore
    ver: 2.4.0
  template:
    metadata:
      labels:
        app: openmrscore
        ver: 2.4.0
    spec:
      containers:
        - name: openmrscore
          image: sunilkumardasu/openmrs-core:2.4.0
          ports:
            - containerPort: 8080
              protocol: TCP

```
* Deployment of the Openmrs will look like

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openmrs-deploy
spec:
  minReadySeconds: 10
  replicas: 3
  selector:
    matchLabels:
      app: openmrscore
      ver: 2.4.0
  strategy:
    rollingUpdate:
      maxSurge: 35%
      maxUnavailable: 30%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: openmrscore
        ver: 2.4.0
    spec:
      containers:
        - name: openmrscore
          image: sunilkumardasu/openmrs-core:2.4.0
          ports:
            - containerPort: 8080
              protocol: TCP
```

* While Writing mysql, we need to persist data in the case of azure [Refer](https://docs.microsoft.com/en-Us/azure/aks/azure-disks-dynamic-pv)

* Deployment for mysql in azure

```
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-managed-disk
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: default
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deploy
spec:
  minReadySeconds: 10
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.6
          ports:
            - containerPort: 3306
              protocol: TCP
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: mysqlvolume
		  env:
			- name: MYSQL_DATABASE
			  value: 'openmrs'
			- name: MYSQL_USER
			  value: 'sunildasu'
			- name: MYSQL_PASSWORD
			  value: 'sunildasu'
			- name: MYSQL_ROOT_PASSWORD
			  value: 'password'
      volumes:
        - name: mysqlvolume
          persistentVolumeClaim:
            claimName: azure-managed-disk
```

# How to enable communications between pods

* [Refer](https://github.com/shaikkhajaibrahim/openmrs-core/tree/master/k8s) for k8s spec yaml files.

* Create Service with loadbalancer for openmrs

```
---
apiVersion: v1
kind: Service
metadata:
  name: openmrssvc
spec:
  selector:
    app: openmrscore
    ver: 2.4.0
  type: LoadBalancer
  ports:
    - port: 8080
```
* Create a Service with Cluster Ip for mysql

```
---
apiVersion: v1
kind: Service
metadata:
  name: mysqlsvc
spec:
  selector:
    app: mysql    
  type: ClusterIP
  ports:
    - port: 3306

```

* Apply the services to the k8s cluster

```
kubectl apply -f openmrs-svc.yml
kubectl apply -f mysql-svc.yml

```
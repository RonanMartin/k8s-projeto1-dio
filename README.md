# Kubernetes

In the development of this project, the images of the containers and services needed in Kubernetes were created so that the production was ready for production.

## **How it was done?**

The base of the project consists of the ready-made files that we found in the [backend](https://github.com/RonanMartin/k8s-projeto1-dio/tree/main/backend) and [frontend](https://github.com/RonanMartin/k8s-projeto1-dio/tree/main/frontend) folders, minus the dockerfile that we created.
The database was also created through a dockerfile and was separated inside the [database](https://github.com/RonanMartin/k8s-projeto1-dio/tree/main/database) folder. For this, later we will see the information that will be necessary for sql.

Outside the folders we have the following files:

**deployment.yml**: To upload the backend and the database
**script.bat**: To deploy from a script for ease in case changes need to be made  
**services.yml**: To upload services, such as load balancer or database connection

### **Cluster and application architecture**

The frontend will be outside the cluster and the backend will be inside the cluster. We must have a Load Balancer that will make the connection with the backend, while the MySQL will be an internal connection.

### **Database**

**database**: meubanco

**table**: mensagens

| id          | int     |
| ----------- | ------- |
| nome        | varchar |
| email       | varchar |
| comentaraio | varchar |

## **Backend: dockerfile**

The dockerfile will encapsulate in a container the backend files, so run php.

`FROM php:7.4-apache` To upload container image using php as base and also using apache

`WORKDIR /var/www/html` To specify the default folder

`COPY index.php /var/www/html/`

`COPY conexao.php /var/www/html/` To specify the files to be copied to the container

```
RUN apt-get update && apt-get install -y \

        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libpng-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install mysqli
```

To RUN some important features like mysqli so we can make the connection

`EXPOSE 80`

## **Database: dockerfile**

In this file will be two environment variables that are important for the connection, which are the **Password** and the **Name** of the database:

```
FROM mysql:5.7

WORKDIR /var/lib/mysql/

ENV MYSQL_ROOT_PASSWORD=Senha123

ENV MYSQL_DATABASE=meubanco

ADD sql.sql /docker-entrypoint-initdb.d

EXPOSE 3306
```

As per the **ADD** above, we need to create the `sql.sql` file. As our example, very simple:

```
CREATE TABLE mensagens (
    id int,
    nome varchar(50),
    email varchar(50),
    comentario varchar (100)
);
```

From this moment we already have the complete images, and we can run `docker build` and `docker push`. However, let's create a script so that all files are executed at the same time. **Advantage:** If we got something wrong, we can just change the code and run again.

## **Script**

For **Windows** the script must have the `.bat` extension, and for **Linux** it must have the `.sh` extension

Instead of running command by command in the terminal, we'll put the commands in the script

`docker build -t ronanmartin/projeto-backend:1.0 backend/.` Here we describe the address where the image is on docker hub

`docker build -t ronanmartin/projeto-database:1.0 database/.` This one is inside the database folder

`docker push ronanmartin/projeto-backend:1.0`

`docker push ronanmartin/projeto-database:1.0` To push the images

Now we need to think about what we need to have the cluster running. We have to go up the deployments and also the services.

`kubectl apply -f ./services.yml`

`kubectl apply -f ./deployment.yml` But we still don't have the files, because first we need to understand some concepts. Later we will see the YML files.

## **Data Persistence in Kubernetes Clusters**

### PV and PVC

Data persistence is all about storage management. When we upload a Pod to the database, we need the data to be saved elsewhere, so that if the Pod falls, we don't lose the data.
PersistentVolume (PV) will indicate where is the data volume in which we will save the information. This PV is usually created by the Cluster administrator.
For a Pod to be able to use PersistentVolume, we need to create a PersistentVolumeClaim (PVC) to bind to the PV. The PVC will be the Pod's request, or deployment, to a given PV, indicating information such as size and specific access mode to this PV.

Important: There are 3 standard means of access, they are:

**ReadWriteOnce:** The volume can be mounted read-write by a single node. Only Pods present on the node can write and read volume information.
**ReadOnlyMany:** The volume can be mounted read-only by many nodes.
**ReadWriteMany:** The volume can be mounted read-write by many nodes. However, this format is not accepted by the Compute Engine of GCP, so for the ReadWriteMany case we have to have an external NFS server.

## **Deployment.yml**

This deployment must have persistent data, so we will use de GPC (Google Cloud Platform) Storage Class, and because of that it will not be necessary to create a PV first.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-dados
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard-rwo
```

In the database specs, the image must match the image we are building for the database.
If the persistent volume already has data, there may be a conflict when uploading it. To avoid this, we add the following argument `- "--ignore-db-dir=lost+found"` to ignore the **lost+found** directory if there is anything there. This way it will go up normally without problems.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: ronanmartin/projeto-database:1.0
        args:
        - "--ignore-db-dir=lost+found"
        imagePullPolicy: Always
        name: mysql
        ports:
        - containerPort: 3306
          name: mysql

        volumeMounts:
        - name: mysql-dados
          mountPath: /var/lib/mysql/
      volumes:
      - name: mysql-dados
        persistentVolumeClaim:
          claimName: mysql-dados
```

`imagePullPolicy: Always` It is important that always download the image again when the deploy is executed.
`mountPath: /var/lib/mysql/` Here is the local where the datas will be mounted.
`- name: mysql-dados` It is also important that we provide the correct persistent volume name.

After that, we arrive at the last part of our deployment, the backend part, where in this case we will generate 6 replicas. It is important to remember that the name must correspond to the name of the image that will be used in the background.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php
  labels:
    app: php
spec:
  replicas: 6
  selector:
    matchLabels:
      app: php
  template:
    metadata:
      labels:
        app: php
    spec:
      containers:
      - name: php
        image: ronanmartin/projeto-backend:1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

## **Services.yml**

Now we need to create the database connection services so that our backend can communicate with the database, and we need a Load Balace. As we are going to connect the load balancer with our backend, let's do this on port 80.

And we need to create the database connection. Note that we are giving the connection name as `name: mysql-connection`, the same name we put in our backend **conexao.php** as servername. Also take care of the application name which must match the application name we have in our deployment.yml file.

```
apiVersion: v1
kind: Service
metadata:
  name: php
spec:
  selector:
    app: php
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-connection
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
```

Now all that we need to do is run the script.

`.\script.bat` So initially it will generate the images, then it will **push** the images, then it will apply the deploy, then the services. You can follow through the log.

`kubectl.exe get services` To verify that the services have been activated correctly.

`kubectl.exe get pod` To check if the pods are running.

And finally, checking the services, get the IP and insert it in the javascript file (js.js), in the field `url: " ",`. Ready our application is running, ready to receive and archive the messages in the database!

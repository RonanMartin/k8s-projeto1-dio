# Kubernetes

In the development of this project, the images of the containers and services needed in Kubernetes were created so that the production was ready for production.

## **How it was done?**

The base of the project consists of the ready-made files that we found in the [backend](https://github.com/RonanMartin/k8s-projeto1-dio/tree/main/backend) and [frontend](https://github.com/RonanMartin/k8s-projeto1-dio/tree/main/frontend) folders, minus the dockerfile that we also created.
The database was also created through a dockerfile and was separated inside the [database](https://github.com/RonanMartin/k8s-projeto1-dio/tree/main/database) folder. For this, later we will see the information that will be necessary for sql.

Outside the folders we have the following files:

**deployment.yml**: To upload the backend and the database
**script.bat**: To deploy from a script for ease in case changes need to be made  
**services.yml**: To upload services, such as load balancer or database connection

### **Cluster and application architecture**

The frontend will be outside the cluster and the backend will be inside the cluster. We must have a Load Balancer that will make the connection with the backend, while the MySQL will be an internal connection.

### **Database**

database: meubanco
table: mensagens

| id          | int     |
| ----------- | ------- |
| nome        | varchar |
| email       | varchar |
| comentaraio | varchar |

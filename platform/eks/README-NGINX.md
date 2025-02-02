# Deploying IBM Operational Decision Manager with NGINX Ingress Controller on Amazon EKS

The aim of this complementary documentation is to explain how to replace the **AWS Load Balancer Controller** usage with an **NGINX Ingress Controller**.

## Prerequisites

You must have created an EKS cluster and set up your environment by following step 1 of [Deploying IBM Operational Decision Manager on Amazon EKS](README.md#1-prepare-your-environment-40-min).

## Provision an NGINX Ingress Controller

You can replace the [Provision an AWS Load Balancer Controller](README.md#c-provision-an-aws-load-balancer-controller) step by provisioning an NGINX Ingress Controller with the following commands.

```console
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-odm-nginx ingress-nginx/ingress-nginx
```

For more information, refer to the [ingress-nginx readme](https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx#install-chart).

If your AWS subnets have been well tagged by using:

```
Key: kubernetes.io/cluster/<cluster-name> | Value: shared
Key: kubernetes.io/role/elb | Value: 1
```

my-odm-ginx should have an available `External-IP` when you run the command:

```console
kubectl get service my-odm-nginx-ingress-nginx-controller
```

You can then go back to the [main documentation](README.md#2-prepare-your-environment-for-the-odm-installation-25-min).

## Install an ODM release with NGINX Ingress Controller

During the helm install, you just have to replace  **eks-values.yaml** with **eks-nginx-values.yaml** that contains the relevant Ingress annotations:
`kubernetes.io/ingress.class: nginx` and `nginx.ingress.kubernetes.io/backend-protocol: https`

To install ODM with the AWS RDS PostgreSQL database created in [step 3](README.md#3-create-an-rds-database-20-min):

- Get the [eks-rds-nginx-values.yaml](./eks-rds-nginx-values.yaml) file and replace the following keys:
  - `<RDS_DB_ENDPOINT>`: your database server endpoint (of the form: `<INSTANCE_NAME>.xxxxxxxx.<REGION>.rds.amazonaws.com`)
  - `<RDS_DATABASE_NAME>`: the initial database name defined when creating the RDS database

```bash
helm install mycompany ibmcharts/ibm-odm-prod --version 22.2.0 -f eks-rds-nginx-values.yaml
```

>NOTE: If you prefer to install ODM for prototyping (not for production) with the ODM PostgreSQL internal database:
>
>- Get the [eks-nginx-values.yaml](./eks-nginx-values.yaml) file:
>
>```bash
>helm install mycompany ibmcharts/ibm-odm-prod --version 22.2.0 -f eks-nginx-values.yaml
>```

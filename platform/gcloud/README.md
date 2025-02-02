# Deploying IBM Operational Decision Manager on Google GKE

This project demonstrates how to deploy an IBM® Operational Decision Manager (ODM) clustered topology using the [container-native load balancer of GKE](https://cloud.google.com/blog/products/containers-kubernetes/container-native-load-balancing-on-gke-now-generally-available).

The ODM services will be exposed using the Ingress provided by the ODM on Kubernetes Helm chart.
This deployment implements Kubernetes and Docker technologies.
Here is the Google Cloud home page: https://cloud.google.com

<img width="1000" height="560" src='./images/architecture.png'/>

The ODM on Kubernetes Docker images are available in the [IBM Entitled Registry](https://www.ibm.com/cloud/container-registry). The ODM Helm chart is available in the [IBM Helm charts repository](https://github.com/IBM/charts).

## Included components

The project comes with the following components:

- [IBM Operational Decision Manager](https://www.ibm.com/docs/en/odm/8.11.1?topic=operational-decision-manager-certified-kubernetes-8110)
- [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine)
- [Google Cloud SQL for PostgreSQL](https://cloud.google.com/sql)
- [IBM License Service](https://github.com/IBM/ibm-licensing-operator)

## Tested environment

The commands and tools have been tested on macOS and Linux.

## Prerequisites

First, install the following software on your machine:
- [gcloud CLI](https://cloud.google.com/sdk/gcloud)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm v3](https://helm.sh/docs/intro/install/)

Then, perform the following tasks:

1. Create a Google Cloud account by connecting to the Google Cloud Platform [console](https://console.cloud.google.com/). When prompted to sign in, create a new account by clicking **Create account**.

2. [Create a Google Cloud project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)

3. [Manage the associated billing](https://cloud.google.com/billing/docs/how-to/modify-project#confirm_billing_is_enabled_on_a_project).

Without the relevant billing level, some Google Cloud resources will not be created.

> NOTE:  Prerequisites and software supported by ODM 8.11 are listed on [the Detailed System Requirements page](https://www.ibm.com/software/reports/compatibility/clarity-reports/report/html/softwareReqsForProduct?deliverableId=2D28A510507B11EBBBEA1195F7E6DF31&osPlatforms=AIX%7CLinux%7CMac%20OS%7CWindows&duComponentIds=D002%7CS003%7CS006%7CS005%7CC006&mandatoryCapIds=30%7C1%7C13%7C25%7C26&optionalCapIds=341%7C47%7C9%7C1%7C15).

## Steps to deploy ODM on Kubernetes from Google GKE

<!-- TOC depthfrom:3 depthto:3 -->

- [Prepare your GKE instance 30 min](#prepare-your-gke-instance-30-min)
- [Create the Google Cloud SQL PostgreSQL instance 10 min](#create-the-google-cloud-sql-postgresql-instance-10-min)
- [Prepare your environment for the ODM installation 10 min](#prepare-your-environment-for-the-odm-installation-10-min)
- [Manage a digital certificate 2 min](#manage-a-digital-certificate-2-min)
- [Install the ODM release 10 min](#install-the-odm-release-10-min)
- [Access ODM services](#access-odm-services)
- [Track ODM usage with the IBM License Service](#track-odm-usage-with-the-ibm-license-service)

<!-- /TOC -->

### 1. Prepare your GKE instance (30 min)

Refer to the [GKE quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart) for more information.

#### a. Log into Google Cloud

After installing the `gcloud` tool, use the following command line:

```
gcloud auth login <ACCOUNT>
```

#### b. Create a GKE cluster

There are several [types of clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/types-of-clusters).
In this article, we chose to create a [regional cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-regional-cluster).
Regions and zones (used below) can be listed respectively with `gcloud compute regions list` and `gcloud compute zones list`.

- Set the project (associated to a billing account):

  ```
  gcloud config set project <PROJECT_ID>
  ```

- Set the region:

  ```
  gcloud config set compute/region <REGION (ex: europe-west1)>
  ```

- Set the zone:

  ```
  gcloud config set compute/zone <ZONE (ex: europe-west1-b)>
  ```

- Create a cluster and [enable autoscaling](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-autoscaler). Here, we start with 6 nodes (16 max):

  ```
  gcloud container clusters create <CLUSTER_NAME> \
    --release-channel=regular --cluster-version=1.24 \
    --enable-autoscaling --num-nodes=6 --total-min-nodes=1 --total-max-nodes=16
  ```

  > If you get a red warning about a missing gke-gcloud-auth-plugin, install it with `gcloud components install gke-gcloud-auth-plugin` and enable it for each kubectl command with `export USE_GKE_GCLOUD_AUTH_PLUGIN=True` ([more information](https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke)).

  > NOTE: You can also create your cluster from the Google Cloud Platform using the **Kubernetes Engine** > **Clusters** panel and clicking the **Create** button
  > <img width="1000" height="300" src='./images/create_cluster.png'/>

#### c. Set up your environment

- Create a kubeconfig to connect to your cluster:
  ```
  gcloud container clusters get-credentials <CLUSTER_NAME>
  ```

  > NOTE: You can also retrieve the command line to configure `kubectl` from the Google Cloud Console using the **Kubernetes Engine** > **Clusters** panel and clicking **Connect** on the dedicated cluster.
  > <img width="1000" height="300" src='./images/connection.png'/>

- Check your environment

  If your environment is set up correctly, you should be able to get the cluster information by running the following command:
  ```
  kubectl cluster-info
  ```

### 2. Create the Google Cloud SQL PostgreSQL instance (10 min)

#### a. Create the database instance

We will use the Google Cloud Platform console to create the database instance.

- Go to the [SQL context](https://console.cloud.google.com/sql), and then click the **CREATE INSTANCE** button
- Click **Choose PostgreSQL**
  - Instance ID: ``<YourInstanceName>``
  - Password: ``<PASSWORD>`` - Take note of this password.
  - Database version: `PostgreSQL 13`
  - Region: ``<REGION>`` (must be the same as the cluster for the communication to be optimal between the database and the ODM instance)
  - Keep **Multiple zones** for Zonal availability to the highest availability
  - Expand **Show customization option** and expand **Connections**
    - As *Public IP* is selected by default, in Authorized networks, click the **ADD NETWORK** button, put a name and add *0.0.0.0/0* for Network, then click **DONE**.
      > NOTE: It is not recommended to use a public IP. In a production environment, you should use a private IP.
- Click **CREATE INSTANCE**

After the database instance is created, you can drill on the SQL instance overview to retrieve needed information to connect to this instance, like the IP address and the connection name. Take note of the **Public IP address**.

<img width="1000" height="630" src='./images/database_overview.png'/>

> NOTE: A default *postgres* database is created with a default *postgres* user. You can change the password of the postgres user in the **Users** panel by selecting the *postgres* user, and then using the **Change password** menu:
> <img width="1000" height="360" src='./images/database_changepassword.png'/>

#### b. Create the database secret for Google Cloud SQL PostgreSQL

To secure access to the database, you must create a secret that encrypts the database user and password before you install the Helm release.

```
kubectl create secret generic <ODM_DB_SECRET> \
  --from-literal=db-user=<USERNAME> \
  --from-literal=db-password=<PASSWORD> 
```

Where:
- `<ODM_DB_SECRET>` is the secret name
- `<USERNAME>` is the database username (default is *postgres*)
- `<PASSWORD>` is the database password (PASSWORD set during the PostgreSQL instance creation above)

### 3. Prepare your environment for the ODM installation (10 min)

To get access to the ODM material, you need an IBM entitlement key to pull the images from the IBM Entitled Registry.

#### a. Retrieve your entitled registry key

- Log in to [MyIBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary) with the IBMid and password that are associated with the entitled software.

- In the Container software library tile, verify your entitlement on the **View library** page, and then go to **Get entitlement key** to retrieve the key.

#### b. Create a pull secret by running a kubectl create secret command.

```
kubectl create secret docker-registry <REGISTRY_SECRET> \
        --docker-server=cp.icr.io \
        --docker-username=cp \
        --docker-password="<API_KEY_GENERATED>" \
        --docker-email=<USER_EMAIL>
```

Where:

* `<REGISTRY_SECRET>` is the secret name.
* `<API_KEY_GENERATED>` is the entitlement key from the previous step. Make sure you enclose the key in double-quotes.
* `<USER_EMAIL>` is the email address associated with your IBMid.

> NOTE:  The `cp.icr.io` value for the docker-server parameter is the only registry domain name that contains the images. You must set the docker-username to `cp` to use an entitlement key as docker-password.

Take note of the secret name so that you can set it for the *image.pullSecrets* parameter when you run a helm install command of your containers.  The *image.repository* parameter will later be set to `cp.icr.io/cp/cp4a/odm`.

#### c. Add the public IBM Helm charts repository

```
helm repo add ibmcharts https://raw.githubusercontent.com/IBM/charts/master/repo/ibm-helm
helm repo update
```

#### d. Check you can access ODM charts

```
helm search repo ibm-odm-prod
NAME                  	CHART VERSION   APP VERSION     DESCRIPTION
ibmcharts/ibm-odm-prod	22.2.0          8.11.1.0        IBM Operational Decision Manager
```

### 4. Manage a digital certificate (2 min)

#### a. (Optional) Generate a self-signed certificate

In this step, you will generate a certificate to be used by the GKE load balancer.

If you do not have a trusted certificate, you can use OpenSSL and other cryptography and certificate management libraries to generate a certificate file and a private key to define the domain name and to set the expiration date. The following command creates a self-signed certificate (`.crt` file) and a private key (`.key` file) that accept the domain name *mycompany.com*. The expiration is set to 1000 days:

```
openssl req -x509 -nodes -days 1000 -newkey rsa:2048 -keyout mycompany.key \
        -out mycompany.crt -subj "/CN=mycompany.com/OU=it/O=mycompany/L=Paris/C=FR"
```

#### b. Create a TLS secret with these keys

```
kubectl create secret tls mycompany-crt-secret --key mycompany.key --cert mycompany.crt
```

The certificate must be the same as the one you used to enable TLS connections in your ODM release. For more information, see [Server certificates](https://www.ibm.com/docs/en/odm/8.11.1?topic=servers-server-certificates) and [Working with certificates and SSL](https://docs.oracle.com/cd/E19830-01/819-4712/ablqw/index.html).

### 5. Install the ODM release (10 min)

#### a. Install an ODM Helm release

The ODM services will be exposed with an Ingress that uses the previously created `mycompany` certificate.
It automatically creates an HTTPS GKE load balancer. We will disable the ODM internal TLS as it is not needed.

- Get the [gcp-values.yaml](./gcp-values.yaml) file and replace the following keys:

  - `<REGISTRY_SECRET>`: the name of the secret containing the IBM Entitled Registry key
  - `<ODM_DB_SECRET>`: the name of the secret containing the database user and password
  - `<DB_ENDPOINT>`: the database IP
  - `<DATABASE_NAME>`: the database name (default is postgres)

  > NOTE: You can configure the driversUrl parameter to point to the appropriate version of the Google Cloud SQL PostgreSQL driver. For more information, refer to the [Cloud SQL Connector for Java](https://github.com/GoogleCloudPlatform/cloud-sql-jdbc-socket-factory#cloud-sql-connector-for-java) documentation.

- Install the chart from IBM's public Helm charts repository:

    ```
    helm install <release> ibmcharts/ibm-odm-prod --version 22.2.0 -f gcp-values.yaml
    ```

  > NOTE: You might prefer to access ODM components through the NGINX Ingress controller instead of using the IP addresses. If so, please follow [these instructions](README_NGINX.md).

#### b. Check the topology

Run the following command to check the status of the pods that have been created:

```
kubectl get pods
NAME                                                   READY   STATUS    RESTARTS   AGE
<release>-odm-decisioncenter-***                       1/1     Running   0          20m
<release>-odm-decisionrunner-***                       1/1     Running   0          20m
<release>-odm-decisionserverconsole-***                1/1     Running   0          20m
<release>-odm-decisionserverruntime-***                1/1     Running   0          20m
```

#### c. Check the Ingress and the GKE LoadBalancer

To get the status of the current deployment, go to the [Kubernetes Engine / Services & Ingress Panel](https://console.cloud.google.com/kubernetes/ingresses) in the console.

The Ingress remains in the state *Creating ingress* for several minutes until the pods are up and running, and the backend gets in a healthy state.

<img width="1000" height="308" src='./images/ingress_creating.png'/>

You can also check the [load balancer status](https://console.cloud.google.com/net-services/loadbalancing/list/loadBalancers).
It provides information about the backend using the service health check.

<img width="1000" height="352" src='./images/loadbalancer.png'/>

In the Ingress details, you should get a *HEALTHY* state on all backends.
This panel also provides some logs on the load balancer activity.
When the Ingress shows an OK status, all ODM services can be accessed.

<img width="1000" height="517" src='./images/ingress_details.png'/>

#### d. Create a Backend Configuration for the Decision Center Service

Sticky session is needed for Decision Center. The browser contains a cookie which identifies the user session that is linked to a unique container.
The ODM on Kubernetes Helm chart has a [clientIP](https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-ipvs) for the Decision Center session affinity. Unfortunately, GKE does not use it automatically.
You will not encounter any issue until you scale up the Decision Center deployment.

A configuration that uses [BackendConfig](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-features#session_affinity) is needed to manage session affinity at the load balancer level.

- Create the [Decision Center Backend Config](decisioncenter-backendconfig.yaml):

  ```
  kubectl create -f decisioncenter-backendconfig.yaml
  ```

- Annotate the Decision Center Service with this GKE Backend Config:

  ```
  kubectl annotate service <release>-odm-decisioncenter \
          cloud.google.com/backend-config='{"ports": {"9453":"dc-backendconfig"}}'
  ```

  As soon as GKE manages Decision Center session affinity at the load balancer level, you can check the ClientIP availability below the Decision Center Network Endpoint Group configuration from the Google Cloud Console in the Load Balancer details.

  <img width="1000" height="353" src='./images/dc_sessionaffinity.png'/>

### 6. Access ODM services

In a real enterprise use case, to access the mycompany.com domain name, you have to deal with [Google Managed Certificate](https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs) and [Google Cloud DNS](https://cloud.google.com/dns).

In this trial, we use a self-signed certificate. So, there is no extra charge like certificate and domain purchase.
We only have to manage a configuration to simulate the mycompany.com access.

- Get the EXTERNAL-IP with the command line:

  ```
  kubectl get ingress <release>-odm-ingress -o jsonpath='{.status.loadBalancer.ingress[].ip}'
  ```

- Edit your /etc/hosts file and add the following entry:

  ```
  <EXTERNAL-IP> mycompany.com
  ```

- You can now access all ODM services with the following URLs:

  | SERVICE NAME | URL | USERNAME/PASSWORD
  | --- | --- | ---
  | Decision Server Console | https://mycompany.com/res | odmAdmin/odmAdmin
  | Decision Center | https://mycompany.com/decisioncenter | odmAdmin/odmAdmin
  | Decision Center REST-API | https://mycompany.com/decisioncenter-api | odmAdmin/odmAdmin
  | Decision Server Runtime | https://mycompany.com/DecisionService | odmAdmin/odmAdmin
  | Decision Runner | https://mycompany.com/DecisionRunner | odmAdmin/odmAdmin

  > NOTE:You can also click the Ingress routes accessible from the Google Cloud console under the [Kubernetes Engine/Services & Ingress Details Panel](https://console.cloud.google.com/kubernetes/ingresses).
  > <img width="1000" height="532" src='./images/ingress_routes.png'/>

### 7. Track ODM usage with the IBM License Service

This section explains how to track ODM usage with the IBM License Service.

#### a. Create an NGINX Ingress controller

- Add the official stable repository:

    ```
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    ```

- Use Helm to deploy the NGINX Ingress controller:

    ```
    helm install nginx-ingress ingress-nginx/ingress-nginx
    ```

#### b. Install the IBM License Service

Follow the **Installation** section of the [Manual installation without the Operator Lifecycle Manager (OLM)](https://www.ibm.com/docs/en/cpfs?topic=software-manual-installation-without-operator-lifecycle-manager-olm)

> NOTE: Make sure you do not follow the instantiation part!

#### c. Create the IBM Licensing instance

Get the [licensing-instance.yaml](./licensing-instance.yaml) file and run the following command:

```
kubectl create -f licensing-instance.yaml
```

> NOTE: You can find more information and use cases on [this page](https://www.ibm.com/docs/en/cpfs?topic=software-configuration).

#### d. Retrieving license usage

After a couple of minutes, the Ingress configuration is created and you will be able to access the IBM License Service by retrieving the URL with the following command:

```
export LICENSING_URL=$(kubectl get ingress ibm-licensing-service-instance -n ibm-common-services -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/ibm-licensing-service-instance
export TOKEN=$(kubectl get secret ibm-licensing-token -o jsonpath={.data.token} -n ibm-common-services |base64 -d)
```

You can access the `http://${LICENSING_URL}/status?token=${TOKEN}` URL to view the licensing usage or retrieve the licensing report .zip file by running the following command:

```
curl -v "http://${LICENSING_URL}/snapshot?token=${TOKEN}" --output report.zip
```

If your IBM License Service instance is not running properly, refer to this [troubleshooting page](https://www.ibm.com/docs/en/cpfs?topic=software-troubleshooting).

## Troubleshooting

If your ODM instances are not running properly, refer to [our dedicated troubleshooting page](https://www.ibm.com/docs/en/odm/8.11.1?topic=8110-troubleshooting-support).

## Getting Started with IBM Operational Decision Manager for Containers

Get hands-on experience with IBM Operational Decision Manager in a container environment by following this [Getting started tutorial](https://github.com/DecisionsDev/odm-for-container-getting-started/blob/master/README.md).

# License

[Apache 2.0](/LICENSE)

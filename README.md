<p align="center">
<img src="/src/frontend/static/icons/Hipster_HeroLogoMaroon.svg" width="300" alt="Online Boutique" />
</p>


**Online Boutique** is a cloud-first microservices demo application.  The application is a
web-based e-commerce app where users can browse items,
add them to the cart, and purchase them.

Google uses this application to demonstrate the use of technologies like
Kubernetes, GKE, Istio, Stackdriver, and gRPC. This application
works on any Kubernetes cluster, like Google
Kubernetes Engine (GKE). It’s **easy to deploy with little to no configuration**.

## Anthos Information
### What is Anthos?
* Anthos is a Google Cloud provided tool for managing multiple clusters, known as fleets. It is considered a ‘step up’ from Kubernetes as it allows for enterprises to control and scale multiple clusters in tandem, instead of just one.
* Benefits of using Anthos:
  * Automatic scaling.
  * Fleet-wide networking features. Instead of managing the networks each cluster and containers directly, Anthos provides the ability to control traffic throughout the fleet.
  * Identity Management (IAM), providing authentication and user control for fleet workloads and users.
  * Observability features, such as health monitoring, resource utilization, and security.
  *Multi-cloud support, fleet can be on-premises, in the cloud, or both.

### What is Istio?
Istio is the open-source 'backbone' or framework that powers Anthos. It is a service mesh that allows organizations to observe and control how microservices interface and distribute data between eachother.
* Pilot - Service Discovery/traffic management (Discovery)
* Citadel - Service-to-service and end-user authentication (Certificates)
* Galley - Configuration validation/ingestion/processing (Configuration)
![image](https://github.com/coradora/microservices-demo-ecu/assets/78966342/de08730f-2019-4790-97da-9272b1ded74f)
    * Image source: [https://istio.io/v1.6/docs/ops/deployment/architecture/](https://istio.io/v1.6/docs/ops/deployment/architecture/)

 
## Quickstart (GKE Anthos Tutorial)

1. Open the [Google Cloud Shell](https://console.cloud.google.com/cloudshell=true).
2. Create a new project
```sh
 gcloud projects create ecu-anthos-demo
 ```
3. Verify the project was created and copy its project ID.
```sh
 gcloud projects list

PROJECT_ID: ecu-anthos-demo
NAME: ecu-anthos-demo
PROJECT_NUMBER: #####
 ```
4. Clone the repository.

 ```sh
 git clone https://github.com/coradora/microservices-demo-ecu
 cd microservices-demo-ecu/
 ```

5. Set the Google Cloud project and region and ensure the Google Kubernetes Engine API is enabled, replacing <PROJECT_ID> with the ID obtained in step 2a.
   
 ```sh
 export PROJECT_ID=<PROJECT_ID>
 export ZONE="us-central1-a"
 export CLUSTER_NAME="onlineboutique"
 gcloud config set project $PROJECT_ID
 gcloud services enable container.googleapis.com \
   --project=${PROJECT_ID}
 ```
  
6. Confirm the services have been enabled for your project.

 ```sh
 gcloud services list --enabled --project=${PROJECT_ID}
 ```

7. Provision a GKE Cluster
 
Create a GKE cluster with at least 4 nodes, machine type `e2-standard-4`, [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity), and the [Kubernetes Gateway API resources](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways):


```sh
gcloud container clusters create ${CLUSTER_NAME} \
  --project=${PROJECT_ID} \
  --zone=${ZONE} \
  --machine-type=e2-standard-4 \
  --num-nodes=4 \
  --workload-pool ${PROJECT_ID}.svc.id.goog \
  --gateway-api "standard"
```

8. Provision managed `Anthos Service Mesh` via Fleet feature API
  ```sh
  # Enable ASM and Fleet APIs
  gcloud services enable mesh.googleapis.com --project ${PROJECT_ID}
  
  # Enable ASM on the project's Fleet
  gcloud container fleet mesh enable --project ${PROJECT_ID}
  
  # Register GKE cluster with Fleet
  gcloud container fleet memberships register ${CLUSTER_NAME} \
      --gke-cluster ${ZONE}/${CLUSTER_NAME} \
      --enable-workload-identity
  
  FLEET_PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format 'value(projectNumber)')
  # Apply mesh_id label to clusters that should be added to the service mesh
  gcloud container clusters update --project ${PROJECT_ID} ${CLUSTER_NAME} \
      --zone ${ZONE} --update-labels="mesh_id=proj-$FLEET_PROJECT_NUMBER"
  
  # Enable managed Anthos Service Mesh on the cluster
  gcloud container fleet mesh update --project ${PROJECT_ID} \
      --management automatic \
      --memberships ${CLUSTER_NAME}
  
  # Enable sidecar injection for Kubernetes namespace where workload is deployed
  kubectl label namespace default istio-injection- istio.io/rev=asm-managed --overwrite
  ```
Note: You can ignore any label "istio-injection" not found errors. The istio-injection=enabled annotation would also work but ASM prefers revision labels.

9. Add Istio Manifest using Kustomize
  ```sh
  cd kustomize/
  kustomize edit add component components/service-accounts
  kustomize edit add component components/service-mesh-istio
  ```
  
  This will update the `kustomize/kustomization.yaml` file which could be similar to:
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - base
  components:
  - components/service-accounts
  - components/service-mesh-istio
  ```

9. Deploy Online Boutique to the cluster.

  ```sh
  kubectl apply -k
  ```

10. Wait for the pods to be ready.

   ```sh
   kubectl get pods
   ```

   After a few minutes, you should see the Pods in a `Running` state:

   ```
  NAME                                     READY   STATUS    RESTARTS   AGE
  adservice-85dcbf69d9-l2j95               1/2     Running   0          39s
  cartservice-7558dc659b-zxrwn             2/2     Running   0          39s
  checkoutservice-7c76547c-cf72l           2/2     Running   0          39s
  currencyservice-65bdb74d87-cfk5w         2/2     Running   0          39s
  emailservice-647bf84659-v7h22            2/2     Running   0          39s
  frontend-7564657f6f-7rvfx                2/2     Running   0          38s
  istio-gateway-istio-589c949445-cr86n     1/1     Running   0          35s
  loadgenerator-6fb8595bb6-p5vl6           2/2     Running   0          38s
  paymentservice-6c788bbc9-gn2wq           2/2     Running   0          37s
  productcatalogservice-6fb8967c89-fhh9t   2/2     Running   0          37s
  recommendationservice-7967ff57-fp277     2/2     Running   0          37s
  redis-cart-76b9545755-2rj2c              1/1     Running   0          9m34s
  shippingservice-5454849755-7hc69         2/2     Running   0          36s
   ```

11. Access the web frontend in a browser using the frontend's external IP.

   ```sh
   kubectl get service frontend-external | awk '{print $4}'
   ```

   Visit `http://EXTERNAL_IP` in a web browser to access your instance of Online Boutique.

12. Navigate to [https://console.cloud.google.com/kubernetes/list/overview&cloudshell=true](https://console.cloud.google.com/kubernetes/list/overview).
  NOTE: You may need to enable GKE Enterprise to view the next bullets.
  * On the left hand navigation menu, click on "Clusters" and click "onlineboutique"
    * Select the Nodes category, and click on any of the Nodes under the Nodes section.
    * You should see a list of Pods. These are the services running within the k8s cluster.
  * On the left hand navigation menu, click on "Gateways, Services, & Ingress" underneath Networking. 
    * Select the Services category. Note that each service has its own Endpoint IP, as well as the load balancer we accessed in Step 11 and the Istio gateway. These Endpoints are how pods within a cluster communicate with eachother, and the Istio gateway controls & monitors traffic going in and out of the cluster.
    * ![image](https://github.com/coradora/microservices-demo-ecu/assets/78966342/6d52f1f0-7d7f-45a4-937e-ae97ce8f16ab)

  * On the left hand navigation menu, select "Service Mesh"
    * You should see a list of all available services running within a cluster, as well as the Istio gateway.

14. Once you are done with it, delete the GKE cluster.

   ```sh
   gcloud container clusters delete online-boutique \
     --project=${PROJECT_ID} --region=${REGION}
   ```

   Deleting the cluster may take a few minutes.

## Service Mesh

![image](https://github.com/coradora/microservices-demo-ecu/assets/78966342/cfd3791f-2451-44b8-8988-41ef152b6f8d)


## Architecture

**Online Boutique** is composed of 11 microservices written in different
languages that talk to each other over gRPC.

[![Architecture of
microservices](/docs/img/architecture-diagram.png)](/docs/img/architecture-diagram.png)

Find **Protocol Buffers Descriptions** at the [`./protos` directory](/protos).

| Service                                              | Language      | Description                                                                                                                       |
| ---------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [frontend](/src/frontend)                           | Go            | Exposes an HTTP server to serve the website. Does not require signup/login and generates session IDs for all users automatically. |
| [cartservice](/src/cartservice)                     | C#            | Stores the items in the user's shopping cart in Redis and retrieves it.                                                           |
| [productcatalogservice](/src/productcatalogservice) | Go            | Provides the list of products from a JSON file and ability to search products and get individual products.                        |
| [currencyservice](/src/currencyservice)             | Node.js       | Converts one money amount to another currency. Uses real values fetched from European Central Bank. It's the highest QPS service. |
| [paymentservice](/src/paymentservice)               | Node.js       | Charges the given credit card info (mock) with the given amount and returns a transaction ID.                                     |
| [shippingservice](/src/shippingservice)             | Go            | Gives shipping cost estimates based on the shopping cart. Ships items to the given address (mock)                                 |
| [emailservice](/src/emailservice)                   | Python        | Sends users an order confirmation email (mock).                                                                                   |
| [checkoutservice](/src/checkoutservice)             | Go            | Retrieves user cart, prepares order and orchestrates the payment, shipping and the email notification.                            |
| [recommendationservice](/src/recommendationservice) | Python        | Recommends other products based on what's given in the cart.                                                                      |
| [adservice](/src/adservice)                         | Java          | Provides text ads based on given context words.                                                                                   |
| [loadgenerator](/src/loadgenerator)                 | Python/Locust | Continuously sends requests imitating realistic user shopping flows to the frontend.                                              |

## Features

- **[Kubernetes](https://kubernetes.io)/[GKE](https://cloud.google.com/kubernetes-engine/):**
  The app is designed to run on Kubernetes (both locally on "Docker for
  Desktop", as well as on the cloud with GKE).
- **[gRPC](https://grpc.io):** Microservices use a high volume of gRPC calls to
  communicate to each other.
- **[Istio](https://istio.io):** Application works on Istio service mesh.
- **[Cloud Operations (Stackdriver)](https://cloud.google.com/products/operations):** Many services
  are instrumented with **Profiling** and **Tracing**. In
  addition to these, using Istio enables features like Request/Response
  **Metrics** and **Context Graph** out of the box. When it is running out of
  Google Cloud, this code path remains inactive.
- **[Skaffold](https://skaffold.dev):** Application
  is deployed to Kubernetes with a single command using Skaffold.
- **Synthetic Load Generation:** The application demo comes with a background
  job that creates realistic usage patterns on the website using
  [Locust](https://locust.io/) load generator.

## Development

See the [Development guide](/docs/development-guide.md) to learn how to run and develop this app locally.

## Demos featuring Online Boutique

- [Anthos Service Mesh Workshop: Lab Guide](https://codelabs.developers.google.com/codelabs/anthos-service-mesh-workshop)

---

This is not an official Google project.

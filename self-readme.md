### Detailed Notes on "Demo of E-Commerce Three-Tier Application on AWS EKS"

---
![Deploy an E Commerce Three Tier application on AWS EKS _ 8 Services and 2 Databases 10-15 screenshot (1)](https://github.com/user-attachments/assets/10679d98-2278-4857-b7ae-5f407fc63742)

#### **Introduction to Three-Tier Architecture**
1. **Overview**: 
   - The Three-Tier Architecture model is a design pattern used in application development.
   - It consists of three distinct layers: Presentation, Logic (Business), and Data.
   - Commonly used in various applications like e-commerce platforms, social media, and more.

2. **Components of Three-Tier Architecture**:
   - **Presentation Layer (Frontend)**: The user interface, responsible for displaying information to the user and handling user inputs.
   - **Logic Layer (Backend)**: Handles the application logic, processes user inputs, interacts with the data layer, and sends responses to the presentation layer.
   - **Data Layer (Database)**: Manages data storage and retrieval.

---

#### **Demo Project: Robot Shop**

1. **Project Overview**:
   - The demo project, "Robot Shop," simulates an e-commerce platform selling robots and AI products.
   - Uses multiple microservices, each written in different programming languages.
   - Components include RabbitMQ for messaging, Redis for in-memory data storage, and a comprehensive frontend.

2. **Project Features**:
   - **Registration and Login**: Users can register and log in.
   - **Product Catalog**: Displays various robots with images, descriptions, ratings, and pricing.
   - **Shopping Cart**: Users can add products to the cart and proceed to checkout.
   - **Shipping and Payments**: Integration options for payment gateways and shipping details.

---

#### **Microservices Architecture**
![Deploy an E Commerce Three Tier application on AWS EKS _ 8 Services and 2 Databases 25-9 screenshot](https://github.com/user-attachments/assets/6a91948e-a95e-40f9-819e-f403e3aad4d3)

1. **Benefits of Microservices**:
   - **Modularity**: Each service can be developed, deployed, and scaled independently.
   - **Technology Diversity**: Different services can use different technologies/languages based on the requirements.
   - **Fault Isolation**: Issues in one service don't necessarily affect others.

2. **Microservices in Robot Shop**:
  - **Catalog Service**: Handles the product catalog.
  - **Ratings Service**: Manages product ratings.
  - **Cart Service**: Manages the shopping cart.
  - **Payment Service**: Manages payment processing.
  - **Shipping Service**: Handles shipping information.
  - **User Service**: Manages user registration and login.
  - **Dispatch service**: for handeling disptach
  - **web service**: 

3. **Databases**: - Different databases are used for various services (it is not that we have to use mongo db and mysql for specific thing yha sirf smjhane ke liye hai dono mein se koi 1 bhi use kr skte the dono chijo ke liye but alg alg)
  - **MySQL**: Used for catalog(product information) and ratings data.
  - **MongoDB**: Used to store user data.
  - **Redis**: In-memory data store used for the shopping cart data for quick access and session persistence.(jisse agar user kuch cart mein add kre to agar wo baad me aaye to usse wo fir se cart mein mile)

---

#### Technical know how - **Deploying on AWS EKS**

1. **Setting Up EKS Cluster**:
   - The demo project is deployed on AWS EKS (Elastic Kubernetes Service).
   - Prerequisites include `eksctl`, `kubectl`, and AWS CLI.
   - Command to create an EKS cluster: 
    ```
    eksctl create cluster --name demo-cluster-three-tier-1 --region us-east-1
    ```
 follow EKS folder for further steps..

2. **Kubernetes Resources**:
   - **Deployments**: Manages stateless applications.
   - **StatefulSets**: Manages stateful applications like databases.
   - **Services**: Exposes microservices within the cluster and to the outside world.

3. **Helm Charts**:
   - Used for managing Kubernetes applications manifests.
   - Simplifies the deployment and management of complex applications by using Helm charts.

4. **for Redis as statefulset**:
![Deploy an E Commerce Three Tier application on AWS EKS _ 8 Services and 2 Databases 33-40 screenshot](https://github.com/user-attachments/assets/4775d136-0c04-4431-abeb-7233c5162b64)
![Deploy an E Commerce Three Tier application on AWS EKS _ 8 Services and 2 Databases 43-0 screenshot](https://github.com/user-attachments/assets/bd636365-b167-49bd-8720-3a374ad11439)


    **1. Redis as a StatefulSet**
    - Redis is commonly deployed as a StatefulSet.yaml(not as deployment.yaml) on Kubernetes because it is an in-memory data store, and persistent storage/volume is required to retain data. refer path for redis StatefulSet.YAML..  K8s/helm/templates/redis-statefulset.yaml
    - StatefulSets ensure that each instance(pod) of Redis has a unique identity(remain constant even during pod restarts or rescheduling) and stable storage(PV).

    **2. Persistent Volume (PV)**
    - Redis requires a persistent volume(storage) in our case because we are on AWS we mostly use Elastic Block Store (EBS) or Elastic File System (EFS) and For this example, we are using EBS as the persistent volume.

    **3. Persistent Volume Claim (PVC) and Storage class**
    - **Persistent Volume Claim (PVC):** 
        - A request for storage by a user/application. When a PVC is created, Kubernetes looks for a suitable PV that matches the request or uses a Storage Class to dynamically provision one.
        - while creating PVC, specify the desired Storage Class and storage requirements.

               ```yaml
               apiVersion: v1
               kind: PersistentVolumeClaim
               metadata:
               name: redis-pvc
               spec:
               accessModes:
                  - ReadWriteOnce
               storageClassName: ebs-sc
               resources:
                  requests:
                     storage: 10Gi
               ```

    - **Storage Class:** (define the type and parameters of storage for dynamic provisioning)
        - A Storage Class in Kubernetes defines the type of storage (e.g., EBS, EFS) and parameters for dynamically provisioning storage.
        - When a PVC references a Storage Class, that Storage Class is used to provision/create the storage/PV (e.g., EBS volume on AWS).

               ```yaml
               apiVersion: storage.k8s.io/v1
               kind: StorageClass
               metadata:
                  name: ebs-sc
               provisioner: ebs.csi.aws.com
               parameters:
                  type: gp2
                  fsType: ext4
               reclaimPolicy: Retain
               volumeBindingMode: WaitForFirstConsumer
               ```

    **4. Automatic Provisioning with EBS CSI contoller Plugin****
        - When a PVC is created with a reference to the ebs-sc Storage Class, the EBS CSI contoller Plugin detects the PVC automatically and use SC info to provisions an EBS volume and attaches it to the Redis StatefulSet.
        **ebs csi Controller Action:**
        - The CSI (Container Storage Interface) provisioner/controller watches for PVCs.
        - When a PVC is created, the controller detects it and looks at the specified Storage Class(for type and parameters). which SC to use is specified in PVC.
        - The controller then provisions a PV based on the Storage Class parameters.
        - For example, the `ebs.csi.aws.com` provisioner would interact with AWS to create an EBS volume.
        - A new PV is created and bound to the PVC

      **PV Creation and Binding:** The provisioned `PV` is now bound to the `PVC`, making the storage available to the application.(below will be auto generated resource)

               ```yaml
               apiVersion: v1
               kind: PersistentVolume
               metadata:
               name: pv-redis
               spec:
               capacity:
                  storage: 10Gi
               accessModes:
                  - ReadWriteOnce
               persistentVolumeReclaimPolicy: Retain
               storageClassName: ebs-sc
               csi:
                  driver: ebs.csi.aws.com
                  volumeHandle: <volume-id>
               ```
6. **Install chart Helm on K8s cluster**: Deploy application using helm on k8s cluster after all the steps from EKS folder.

---

#### **High-Level Design Considerations**
![Deploy an E Commerce Three Tier application on AWS EKS _ 8 Services and 2 Databases 19-1 screenshot](https://github.com/user-attachments/assets/8bff4a3e-a6d1-444e-914f-ab7a31939dd8)

1. **Workflow Components**:
   - **User Workflow**: 
        1. **User Registration/Login**: Users can register or log in to the platform.
        2. **Catalog Access**: Users can browse different categories and view products.
        3. **Product Interaction**: Users can view product details, ratings, and pricing.
        4. **Shopping Cart**: Items can be added to the cart and reviewed.
        5. **Checkout**: Users can proceed to checkout, enter shipping details, do payment to confirm orders and order-id is generated after successful purchase.
   - **Admin/Application Workflow**: Includes management of product listings, order processing, and user data.

2. **Data Flow and Interaction**:
   - User interactions with the frontend trigger API calls to the backend.
   - Backend services process these requests and interact with the databases.
   - Responses are sent back to the frontend for user display.

---

#### **Best Practices and Additional Insights**

1. **Microservices Best Practices**:
   - Design services to be stateless where possible.
   - Use consistent APIs for communication between services.
   - Implement logging and monitoring for better visibility and debugging.

2. **Security Considerations**:
   - Secure communication between services using HTTPS.
   - Use IAM roles and policies to manage access to AWS resources.
   - Implement authentication and authorization mechanisms.

3. **Scaling and Performance**:
   - Use auto-scaling features of EKS to handle varying loads.
   - Optimize database queries and use caching (e.g., Redis) to improve response times.
   - Regularly monitor and analyze application performance metrics.

---

#### **Conclusion**

The "Robot Shop" demo project serves as an excellent learning tool for understanding the Three-Tier Architecture and microservices on AWS EKS. By leveraging different programming languages, databases, and Kubernetes resources, it provides a comprehensive view of building and deploying modern, scalable applications.

---

These notes provide a structured overview of the key concepts and practices related to deploying a three-tier e-commerce application on AWS EKS. They can be used as a reference for both learning and implementing similar projects.


--------------------------

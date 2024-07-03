# Multi-Container-Pod

# Deploying Two Containers as a Web Application in a Kubernetes Pod

In this guide, we'll walk through deploying two containers (one running a Node.js application on port 3003 and another running an Angular application on port 80) in a single pod in a Kubernetes cluster. We'll also expose these applications using a NodePort service.

# POD -> K8S 

Pods are Kubernetes Objects that are the basic unit for running our containers inside our Kubernetes cluster. In fact, Pods are the smallest object of the Kubernetes Model.Kubernetes uses pods to run an instance of our application and a single pod represents a single instance of that application. We can scale out our application horizontally by adding more Pod replicas.

Pods can contain a single or multiple containers as a group, that share the same resources within that Pod (storage, network resources, namespaces). Pods typically have a 1-1 mapping with containers, but in more advanced situations, we may run multiple containers in a Pod.Pods are epheremeral resources, meaning that Pods can be terminated at any point and then restarted on another node within our Kubernetes cluster. They live and die, but Pods will never come back to life.

# Single/Multi-Container -> POD 

Running a single container in a Pod is a common use case. Here, the Pod acts as a wrapper around the single container and Kubernetes manages the Pods rather than the containers directly.

We can also run multiple containers in a Pod. Here, the Pod wraps around an application that's composed of multiple containers and share resources.
If we need to run multiple containers within a single Pod, it's recommended that we only do this in cases where the containers are tightly coupled.

# Multi Container Design Patterns
• Sidecar Pattern – The sidecar pattern uses a second container in a Pod to provide supplementary functionality for the primary container. For example, if you had a container running a web server like nginx, you could have a side car container that periodically pulls new content from a remote data source and makes that content available to the web server container.
    
• Ambassador Pattern – The ambassador pattern uses a second container in a Pod to proxy incoming network traffic and then route that traffic to the primary Pod. For example, if traffic into your Pod is via TLS you may use an ambassador container to act as the TLS termination point and then pass the unencrypted traffic onto the primary container via plain HTTP.
    
• Adapter Pattern – The adapter pattern uses a second container to perform some kind of  transformation of data coming from the primary container. An example is a container that takes logs data produced by the primary Pod and performs some kind of transform before making those logs available outside the Pod

# Prerequisites:
1. A Kubernetes cluster - Microk8s
2. kubectl configured to interact with your cluster
3. Docker installed to build container images

# Step 1: Create Docker Images

First, we need Docker images for the Node.js and Angular applications.

# Node.js Dockerfile (`Dockerfile.nodejs`):

```
# Use the official Node.js image
FROM node:14

# Create and change to the app directory
WORKDIR /usr/src/app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application source code
COPY . .

# Expose port 3003
EXPOSE 3003

# Start the application
CMD ["node", "app.js"]
```

# Angular Dockerfile (`Dockerfile.angular`):

```
# Use the official Node.js image to build the Angular app
FROM node:14 as build

# Create and change to the app directory
WORKDIR /usr/src/app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application source code
COPY . .

# Build the Angular app
RUN npm run build

# Use the official Nginx image to serve the app
FROM nginx:alpine

# Copy the build output to replace the default Nginx contents
COPY --from=build /usr/src/app/dist/your-angular-app /usr/share/nginx/html

# Expose port 80
EXPOSE 80
```

# Build and tag the Docker images:

```
docker build -t udhaya21/logmanagement-ui  .
docker build -t sambathkumarj/web  .
```

# Push the images to a container registry:

```
docker push udhaya21/logmanagement-ui
docker push sambathkumarj/web
```

# Step 2: Create Kubernetes Manifests

Now, we'll create Kubernetes manifests for deploying the pod and the service.


Let's break this down a bit. To create Kubernetes objects using YAML, we need to set values for the following fields.

apiVersion - This defines the Kubernetes API version that we want to use in this YAML file. You can read more about API versioning in Kubernetes here.

kind - This defines what kind of Kubernetes object we want to create.

metadata - This is data that helps us uniquely identify the object that we want to create. Here we can provide a name for our app, as well as apply labels to our object.

spec - This defines the state that we want or our object. The format that we use for spec. For our Pod file, we have provided information about the containers that we want to host on our Pod.

# Pod Manifest (`pod.yaml`):

```
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    app: web-app
spec:
  restartPolicy: Never
  containers:
  - name: frontend
    image: udhaya21/logmanagement-ui
    ports:
    - containerPort: 3003
  - name: netflix
    image: sambathkumarj/web
    ports:
    - containerPort: 80
```

# Service Manifest (`service.yaml`):

```
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  type: NodePort
  ports:
  - name: frontend
    port: 3003
    targetPort: 3003
    nodePort: 30081  # Specify a nodePort or let Kubernetes assign one
  - name: netflix
    port: 80
    targetPort: 80
    nodePort: 30082  # Specify a nodePort or let Kubernetes assign one

```

# Step 3: Deploy to Kubernetes

Apply the manifests to your Kubernetes cluster:

```
microk8s kubectl apply -f pod.yaml
microk8s kubectl apply -f svc.yaml
```

# Step 4: Access the Applications

Once the pod and service are deployed, you can access the applications via the NodePort on your cluster's nodes.

```
microk8s kubectl logs multi-container-pod -c frontend 
```

![multi-cont-pod-netflix](https://github.com/sambathkumarj/multi-container-pod-k8s/assets/42794636/da4552c4-362b-455f-a46d-632b5c3bec44)


```
microk8s kubectl logs multi-container-pod -c netflix
```
![multi-cont-pod-frotend](https://github.com/sambathkumarj/multi-container-pod-k8s/assets/42794636/28099009-ce03-403f-a548-65e416ac4de4)


For the Node.js application, open your browser or use `curl` to access:

```
http://<node-ip>:30081
```

![frontend](https://github.com/sambathkumarj/multi-container-pod-k8s/assets/42794636/14037b55-5546-4264-8b8a-0891b8ae2f92)


For the Angular application, open your browser or use `curl` to access:

```
http://<node-ip>:30082
```

![netflix](https://github.com/sambathkumarj/multi-container-pod-k8s/assets/42794636/eb52e138-0239-4e3c-a684-e59aec3c07dc)

In this guide, we've demonstrated how to deploy two containers in a single Kubernetes pod and expose them via a NodePort service. This setup allows you to run multiple applications together and access them externally through specific ports. This approach can be extended and customized based on your application's requirements.

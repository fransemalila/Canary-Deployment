# Kubernetes Canary Deployments Example

This document demonstrates how to perform a canary deployment in Kubernetes. A canary deployment is a strategy where a new version of an application is rolled out to a small subset of users before being fully deployed. This allows you to test the new version in a real-world environment and identify any potential issues before they impact all users.

## Prerequisites

*   A working Kubernetes cluster.
*   `kubectl` command-line tool configured to connect to your cluster.

## Step 1: Creating the Initial Deployment (Blue)

1.  **Create an `initial-deployment.yaml` file with the following content:**

    This deployment defines the initial (or "blue") version of the `random-generator` application, using the `k8spatterns/random-generator:1.0` image. It creates three replicas and labels them with `app: random-generator` and `version: initial`.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: random-generator-blue
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: random-generator
          version: initial
      template:
        metadata:
          labels:
            app: random-generator
            version: initial
        spec:
          containers:
          - image: k8spatterns/random-generator:1.0
            name: random-generator
            ports:
            - containerPort: 8080
            readinessProbe:
              httpGet:
                path: /info
                port: 8080
              initialDelaySeconds: 5
    ```

2.  **Apply the deployment definition:**

    ```bash
    kubectl apply -f initial-deployment.yaml
    ```

3.  **Verify the pods are created and running:**

    ```bash
    kubectl get pods -l app=random-generator,version=initial
    ```

    Wait until all three pods transition to the `Running` status.

## Step 2: Exposing the Pods with a Service

1.  **Create a `service.yaml` file with the following content:**

    This service creates a `ClusterIP` service named `random-generator` that selects pods with the label `app: random-generator`.  It routes traffic to port 8080 of the selected pods.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: random-generator
    spec:
      selector:
        app: random-generator
      ports:
        - protocol: TCP
          port: 80
          targetPort: 8080
    ```

2.  **Apply the service definition:**

    ```bash
    kubectl apply -f service.yaml
    ```

3.  **Test the service:**

    You can use a temporary pod with `curl` to test the service:

    ```bash
    kubectl run tmp --image=alpine/curl:3.14 --restart=Never -it --rm -- curl random-generator.default.svc.cluster.local
    ```

    This command creates a temporary pod, runs `curl` to access the service, and then deletes the pod.  The output should be the response from the `random-generator` application version 1.0.

## Step 3: Creating the New Deployment (Canary/Green)

1.  **Create a `new-deployment.yaml` file with the following content:**

    This deployment defines the new ("green" or canary) version of the `random-generator` application, using the `k8spatterns/random-generator:2.0` image.  It creates only *one* replica and labels it with `app: random-generator` and `version: new`.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: random-generator-new
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: random-generator
          version: new
      template:
        metadata:
          labels:
            app: random-generator
            version: new
        spec:
          containers:
          - image: k8spatterns/random-generator:2.0
            name: random-generator
            ports:
            - containerPort: 8080
            readinessProbe:
              httpGet:
                path: /info
                port: 8080
              initialDelaySeconds: 5
    ```

2.  **Apply the deployment definition:**

    ```bash
    kubectl apply -f new-deployment.yaml
    ```

3.  **Verify the pod is created and running:**

    ```bash
    kubectl get pods -l app=random-generator,version=new
    ```

    Wait until the pod transitions to the `Running` status.

## Step 4: Routing Traffic to Initial or New Pods

The service `random-generator` is configured to select all pods with the label `app: random-generator`. *Crucially, both the old and new deployments use this label.* Therefore, the service will now distribute traffic across *both* the `version: initial` and `version: new` pods. Since there are three pods running version 1.0 and one pod running version 2.0, roughly 25% of traffic will be routed to the new version.

1.  **Test the service again:**

    ```bash
    kubectl run tmp --image=alpine/curl:3.14 --restart=Never -it --rm -- curl random-generator.default.svc.cluster.local
    ```

    Run this command repeatedly. You should see that some requests return the response from version 1.0 and some return the response from version 2.0. This demonstrates that the traffic is being routed to both sets of pods.

## Post-Canary Deployment

*   **Monitor the canary deployment:**  Carefully monitor the performance and behavior of the canary deployment. Check logs, metrics, and user feedback to identify any potential issues.
*   **If everything is working as expected:** Gradually increase the number of replicas in the `random-generator-new` deployment and decrease the number of replicas in the `random-generator-blue` deployment until all traffic is routed to the new version.
*   **If problems are detected:**  Roll back the canary deployment by scaling down the `random-generator-new` deployment and scaling up the `random-generator-blue` deployment.

## Considerations for Production Canary Deployments

*   **Traffic Management:** For more sophisticated traffic routing (e.g., routing based on user agent or geographic location), consider using a service mesh like Istio or Linkerd, or an ingress controller with advanced routing capabilities (e.g., Nginx Ingress Controller).
*   **Automated Rollbacks:** Implement automated rollbacks based on metrics and alerts to quickly revert to the previous version if problems are detected.
*   **Feature Flags:**  Consider using feature flags to enable or disable features in the canary deployment.
*   **Session Affinity (Sticky Sessions):**  If your application requires session affinity, configure the service or ingress controller to route requests from the same user to the same pod.
*   **Observability:**  Ensure that you have comprehensive monitoring and logging in place to track the performance and behavior of both the old and new versions of your application.
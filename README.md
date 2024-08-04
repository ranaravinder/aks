### Application Design and Build

#### 1. Define, Build, and Modify Container Images

##### Building a Container Image
**Example: Simple Node.js Application**

- **Directory Structure:**
  ```
  app/
  ├── Dockerfile
  ├── package.json
  └── server.js
  ```

- **`server.js` (Node.js server):**
  ```javascript
  const express = require('express');
  const app = express();
  const port = 3000;

  app.get('/', (req, res) => res.send('Hello World!'));

  app.listen(port, () => console.log(`App running on port ${port}`));
  ```

- **`package.json` (Dependencies):**
  ```json
  {
    "name": "simple-node-app",
    "version": "1.0.0",
    "main": "server.js",
    "dependencies": {
      "express": "^4.17.1"
    },
    "scripts": {
      "start": "node server.js"
    }
  }
  ```

- **`Dockerfile` (Build Instructions):**
  ```Dockerfile
  FROM node:14
  WORKDIR /app
  COPY package.json /app
  RUN npm install
  COPY . /app
  EXPOSE 3000
  CMD ["npm", "start"]
  ```

- **Build the Docker Image:**
  ```bash
  docker build -t simple-node-app:1.0 .
  ```

##### Modifying a Container Image
- **Tag the Image:**
  ```bash
  docker tag simple-node-app:1.0 myrepo/simple-node-app:1.0
  ```

- **Push the Image to a Docker Registry:**
  ```bash
  docker push myrepo/simple-node-app:1.0
  ```

#### 2. Understand Jobs and CronJobs

##### Job Example
**Run a Batch Job to Calculate Pi:**
- **`job.yaml`:**
  ```yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: pi-job
  spec:
    template:
      spec:
        containers:
        - name: pi
          image: perl
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
        restartPolicy: Never
    backoffLimit: 4
  ```

- **Create the Job:**
  ```bash
  kubectl apply -f job.yaml
  ```

##### CronJob Example
**Schedule a Job to Run Every Minute:**
- **`cronjob.yaml`:**
  ```yaml
  apiVersion: batch/v1
  kind: CronJob
  metadata:
    name: hello-cronjob
  spec:
    schedule: "*/1 * * * *"
    jobTemplate:
      spec:
        template:
          spec:
            containers:
            - name: hello
              image: busybox
              command: ["echo", "Hello, Kubernetes!"]
            restartPolicy: OnFailure
  ```

- **Create the CronJob:**
  ```bash
  kubectl apply -f cronjob.yaml
  ```

#### 3. Multi-Container Pod Design Patterns

##### Sidecar Container Example
**Logging Sidecar for an Application:**
- **`sidecar-pod.yaml`:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: sidecar-pod
  spec:
    containers:
    - name: main-app
      image: my-app-image
    - name: logging-sidecar
      image: busybox
      command: ["sh", "-c", "tail -f /var/log/app.log"]
      volumeMounts:
      - name: log-volume
        mountPath: /var/log
    volumes:
    - name: log-volume
      emptyDir: {}
  ```

- **Create the Pod:**
  ```bash
  kubectl apply -f sidecar-pod.yaml
  ```

##### Init Container Example
**Initialize Application Data Before Main Container Starts:**
- **`init-container-pod.yaml`:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: init-container-pod
  spec:
    initContainers:
    - name: init-myservice
      image: busybox
      command: ['sh', '-c', 'echo "Initializing data..."; sleep 5']
    containers:
    - name: main-app
      image: my-app-image
  ```

- **Create the Pod:**
  ```bash
  kubectl apply -f init-container-pod.yaml
  ```

#### 4. Persistent and Ephemeral Volumes

##### Persistent Volumes (PV) and PersistentVolumeClaims (PVC)
**Define and Use Persistent Storage:**
- **`pv.yaml`:**
  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-demo
  spec:
    capacity:
      storage: 1Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    hostPath:
      path: "/mnt/data"
  ```

- **`pvc.yaml`:**
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: pvc-demo
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  ```

- **`pod-with-pvc.yaml`:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-with-pvc
  spec:
    containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo 'Hello, Kubernetes!' > /mnt/data/hello.txt; sleep 3600"]
      volumeMounts:
      - mountPath: /mnt/data
        name: storage
    volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-demo
  ```

- **Create the Resources:**
  ```bash
  kubectl apply -f pv.yaml
  kubectl apply -f pvc.yaml
  kubectl apply -f pod-with-pvc.yaml
  ```

##### Ephemeral Volumes
**Using an emptyDir Volume:**
- **`emptydir-pod.yaml`:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: emptydir-demo
  spec:
    containers:
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'echo "Hello from emptyDir" > /data/hello.txt; sleep 3600']
      volumeMounts:
      - mountPath: /data
        name: temp-storage
    volumes:
    - name: temp-storage
      emptyDir: {}
  ```

- **Create the Pod:**
  ```bash
  kubectl apply -f emptydir-pod.yaml
  ```

These examples cover defining, building, and modifying container images, as well as utilizing Kubernetes resources like Jobs, CronJobs, multi-container pod design patterns, and different volume types.


###  Application Deployment

### Deployment Strategies with Kubernetes

#### 1. Blue/Green Deployment
**Blue/Green Deployment** is a technique that reduces downtime and risk by running two identical production environments, only one of which (blue or green) serves live production traffic at any time.

##### Example:
1. **Create Blue Deployment (v1):**
   - **`blue-deployment.yaml`:**
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: blue-deployment
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: myapp
           version: blue
       template:
         metadata:
           labels:
             app: myapp
             version: blue
         spec:
           containers:
           - name: myapp
             image: myapp:1.0
     ```

   - **Apply Blue Deployment:**
     ```bash
     kubectl apply -f blue-deployment.yaml
     ```

2. **Create Green Deployment (v2):**
   - **`green-deployment.yaml`:**
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: green-deployment
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: myapp
           version: green
       template:
         metadata:
           labels:
             app: myapp
             version: green
         spec:
           containers:
           - name: myapp
             image: myapp:2.0
     ```

   - **Apply Green Deployment:**
     ```bash
     kubectl apply -f green-deployment.yaml
     ```

3. **Switch Traffic to Green Deployment:**
   - **`service.yaml`:**
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: myapp-service
     spec:
       selector:
         app: myapp
         version: green  # Switch from blue to green
       ports:
       - protocol: TCP
         port: 80
         targetPort: 8080
     ```

   - **Apply Service:**
     ```bash
     kubectl apply -f service.yaml
     ```

4. **Remove Blue Deployment:**
   ```bash
   kubectl delete -f blue-deployment.yaml
   ```

#### 2. Canary Deployment
**Canary Deployment** is a technique to reduce the risk of introducing a new software version in production by slowly rolling out the change to a small subset of users before releasing it to everyone.

##### Example:
1. **Create Canary Deployment (v2) with Less Replicas:**
   - **`canary-deployment.yaml`:**
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: canary-deployment
     spec:
       replicas: 1  # Fewer replicas for canary
       selector:
         matchLabels:
           app: myapp
           version: canary
       template:
         metadata:
           labels:
             app: myapp
             version: canary
         spec:
           containers:
           - name: myapp
             image: myapp:2.0
     ```

   - **Apply Canary Deployment:**
     ```bash
     kubectl apply -f canary-deployment.yaml
     ```

2. **Update Service to Include Both Versions:**
   - **`service-canary.yaml`:**
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: myapp-service
     spec:
       selector:
         app: myapp
       ports:
       - protocol: TCP
         port: 80
         targetPort: 8080
       # Canary deployment with different version
       selector:
         app: myapp
         version: canary  # Switch to canary version
     ```

   - **Apply Service:**
     ```bash
     kubectl apply -f service-canary.yaml
     ```

3. **Monitor and Gradually Increase Canary Replicas:**
   - If canary version is stable, increase replicas:
     ```bash
     kubectl scale deployment canary-deployment --replicas=3
     ```

4. **Decommission Old Version:**
   ```bash
   kubectl delete -f blue-deployment.yaml
   ```

#### 3. Rolling Updates
**Rolling Updates** allow updating the application version without downtime by incrementally updating instances with the new version.

##### Example:
1. **Create Initial Deployment:**
   - **`deployment.yaml`:**
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: myapp-deployment
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: myapp
       template:
         metadata:
           labels:
             app: myapp
         spec:
           containers:
           - name: myapp
             image: myapp:1.0
     ```

   - **Apply Deployment:**
     ```bash
     kubectl apply -f deployment.yaml
     ```

2. **Update Deployment Image:**
   - **`deployment-update.yaml`:**
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: myapp-deployment
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: myapp
       template:
         metadata:
           labels:
             app: myapp
         spec:
           containers:
           - name: myapp
             image: myapp:2.0
     ```

   - **Apply Update:**
     ```bash
     kubectl apply -f deployment-update.yaml
     ```

3. **Monitor Rolling Update:**
   ```bash
   kubectl rollout status deployment/myapp-deployment
   ```

4. **Rollback if Needed:**
   ```bash
   kubectl rollout undo deployment/myapp-deployment
   ```

#### 4. Using Helm to Deploy Existing Packages

**Helm** is a package manager for Kubernetes, which helps in defining, installing, and upgrading complex Kubernetes applications.

##### Example:
1. **Install Helm:**
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```

2. **Add Helm Repository:**
   ```bash
   helm repo add stable https://charts.helm.sh/stable
   helm repo update
   ```

3. **Search for a Package:**
   ```bash
   helm search repo nginx
   ```

4. **Install a Package:**
   ```bash
   helm install my-nginx stable/nginx-ingress
   ```

5. **Upgrade a Release:**
   ```bash
   helm upgrade my-nginx stable/nginx-ingress
   ```

6. **Uninstall a Release:**
   ```bash
   helm uninstall my-nginx
   ```

These steps cover common deployment strategies like blue/green and canary, performing rolling updates, and using Helm to deploy existing packages. If you need more details or specific examples, feel free to ask!


###   Application Observability and Maintenance

### Kubernetes Advanced Topics: Realistic Examples

#### 1. Understanding API Deprecations

API deprecations in Kubernetes are important to track for maintaining and upgrading your clusters. The Kubernetes API evolves over time, and old versions of APIs are eventually deprecated and removed.

##### Example: Checking for Deprecated APIs

- **Check for Deprecated APIs in Current Resources:**
  ```bash
  kubectl get ingresses.v1beta1.extensions
  ```
  If you see resources listed, they are using deprecated API versions.

- **Update Deprecated APIs:**
  For example, if Ingress is using `v1beta1`, you should update it to use `v1`:
  - **Deprecated `v1beta1` Ingress:**
    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: example-ingress
    spec:
      rules:
      - host: example.com
        http:
          paths:
          - path: /
            backend:
              serviceName: example-service
              servicePort: 80
    ```

  - **Updated `v1` Ingress:**
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: example-ingress
    spec:
      rules:
      - host: example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-service
                port:
                  number: 80
    ```

- **Use `kubectl deprecations` Plugin:**
  ```bash
  kubectl krew install deprecations
  kubectl deprecations
  ```

#### 2. Implementing Probes and Health Checks

Kubernetes uses liveness, readiness, and startup probes to determine the health of containers.

##### Example: Adding Probes to a Pod

- **`probes.yaml`:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp
  spec:
    containers:
    - name: myapp-container
      image: myapp:1.0
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 3
        periodSeconds: 3
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
  ```

- **Apply the Probes:**
  ```bash
  kubectl apply -f probes.yaml
  ```

#### 3. Using Provided Tools to Monitor Kubernetes Applications

##### Example: Prometheus and Grafana

1. **Install Prometheus and Grafana Using Helm:**
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update

   helm install prometheus prometheus-community/prometheus
   helm install grafana grafana/grafana
   ```

2. **Access Grafana Dashboard:**
   ```bash
   kubectl port-forward svc/grafana 3000:80
   ```
   Open [http://localhost:3000](http://localhost:3000) and log in with the default credentials (`admin/admin`).

3. **Configure Data Sources and Dashboards in Grafana:**
   - Add Prometheus as a data source.
   - Import pre-built dashboards for Kubernetes metrics.

#### 4. Utilizing Container Logs

Logs are essential for debugging and monitoring applications running in containers.

##### Example: Viewing Logs

- **Get Logs of a Running Pod:**
  ```bash
  kubectl logs myapp
  ```

- **Get Logs of a Specific Container:**
  ```bash
  kubectl logs myapp -c myapp-container
  ```

- **Stream Logs:**
  ```bash
  kubectl logs -f myapp
  ```

#### 5. Debugging in Kubernetes

Kubernetes provides several tools and commands to debug issues in the cluster.

##### Example: Debugging a Pod

1. **Describe Pod:**
   ```bash
   kubectl describe pod myapp
   ```

2. **Exec Into a Running Container:**
   ```bash
   kubectl exec -it myapp -- /bin/sh
   ```

3. **Debugging with `kubectl debug`:**
   ```bash
   kubectl debug myapp --image=busybox --target=myapp-container -it
   ```
   This will attach a debugging container to your pod.

4. **Checking Events:**
   ```bash
   kubectl get events
   ```

5. **Using `kubectl port-forward` for Debugging:**
   Forward a port to access an application locally for debugging:
   ```bash
   kubectl port-forward myapp 8080:80
   ```
   Access [http://localhost:8080](http://localhost:8080) to interact with the application running in the pod.

These examples provide a comprehensive overview of handling API deprecations, implementing health checks, monitoring Kubernetes applications, utilizing logs, and debugging within a Kubernetes environment. If you have specific scenarios or additional questions, feel free to ask!

###  Application Environment, Configuration and Security

### Advanced Kubernetes Concepts

#### 1. Custom Resource Definitions (CRDs)

CRDs allow you to extend Kubernetes with your own resource types. This lets you define custom resources that can be managed like native Kubernetes resources.

##### Example: Defining and Using a CRD

1. **Define a CRD:**
   - **`crd.yaml`:**
     ```yaml
     apiVersion: apiextensions.k8s.io/v1
     kind: CustomResourceDefinition
     metadata:
       name: crontabs.stable.example.com
     spec:
       group: stable.example.com
       versions:
       - name: v1
         served: true
         storage: true
         schema:
           openAPIV3Schema:
             type: object
             properties:
               spec:
                 type: object
                 properties:
                   cronSpec:
                     type: string
                   image:
                     type: string
                   replicas:
                     type: integer
       scope: Namespaced
       names:
         plural: crontabs
         singular: crontab
         kind: CronTab
         shortNames:
         - ct
     ```

   - **Apply the CRD:**
     ```bash
     kubectl apply -f crd.yaml
     ```

2. **Create an Instance of the Custom Resource:**
   - **`my-crontab.yaml`:**
     ```yaml
     apiVersion: stable.example.com/v1
     kind: CronTab
     metadata:
       name: my-new-cron-object
     spec:
       cronSpec: "* * * * */5"
       image: my-awesome-cron-image
       replicas: 1
     ```

   - **Apply the Custom Resource:**
     ```bash
     kubectl apply -f my-crontab.yaml
     ```

3. **Verify the Custom Resource:**
   ```bash
   kubectl get crontabs
   ```

#### 2. Authentication, Authorization, and Admission Control

Kubernetes uses several mechanisms to authenticate and authorize users and service accounts, and to control and validate resource creation.

##### Authentication
- **Supported Methods:**
  - X.509 Client Certificates
  - Static Token File
  - Bootstrap Tokens
  - OpenID Connect Tokens
  - Webhook Token Authentication

##### Authorization
- **Role-Based Access Control (RBAC):**
  - **`role.yaml`:**
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      namespace: default
      name: pod-reader
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "watch", "list"]
    ```

  - **`rolebinding.yaml`:**
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: read-pods
      namespace: default
    subjects:
    - kind: User
      name: jane
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role
      name: pod-reader
      apiGroup: rbac.authorization.k8s.io
    ```

  - **Apply RBAC Resources:**
    ```bash
    kubectl apply -f role.yaml
    kubectl apply -f rolebinding.yaml
    ```

##### Admission Control
Admission controllers are plugins that govern and enforce how the cluster is used.

- **Example: Using Pod Security Policies (deprecated in Kubernetes v1.25):**
  - **`psp.yaml`:**
    ```yaml
    apiVersion: policy/v1beta1
    kind: PodSecurityPolicy
    metadata:
      name: restricted
    spec:
      privileged: false
      allowPrivilegeEscalation: false
      requiredDropCapabilities:
      - ALL
      runAsUser:
        rule: MustRunAsNonRoot
      seLinux:
        rule: RunAsAny
      fsGroup:
        rule: MustRunAs
        ranges:
        - min: 1
          max: 65535
      supplementalGroups:
        rule: MustRunAs
        ranges:
        - min: 1
          max: 65535
      volumes:
      - 'configMap'
      - 'emptyDir'
      - 'projected'
      - 'secret'
      - 'downwardAPI'
      - 'persistentVolumeClaim'
    ```

  - **Apply Pod Security Policy:**
    ```bash
    kubectl apply -f psp.yaml
    ```

#### 3. Resource Requirements, Limits, and Quotas

Defining resource requests and limits ensures that pods run with the appropriate amount of CPU and memory.

##### Example: Resource Requests and Limits

- **`resource-limits.yaml`:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: resource-demo
  spec:
    containers:
    - name: resource-demo-container
      image: busybox
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      command: ["sh", "-c", "echo Hello Kubernetes! && sleep 3600"]
  ```

- **Apply Resource Limits:**
  ```bash
  kubectl apply -f resource-limits.yaml
  ```

##### Example: Resource Quotas

- **`resource-quota.yaml`:**
  ```yaml
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: example-quota
    namespace: default
  spec:
    hard:
      pods: "10"
      requests.cpu: "4"
      requests.memory: "2Gi"
      limits.cpu: "8"
      limits.memory: "4Gi"
  ```

- **Apply Resource Quota:**
  ```bash
  kubectl apply -f resource-quota.yaml
  ```

#### 4. ConfigMaps

ConfigMaps store configuration data in key-value pairs.

##### Example: Creating and Using a ConfigMap

- **Create ConfigMap from Literal Values:**
  ```bash
  kubectl create configmap game-config --from-literal=player=admin --from-literal=level=1
  ```

- **Create ConfigMap from a File:**
  - **`game-config.yaml`:**
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: game-config
    data:
      player: "admin"
      level: "1"
    ```

  - **Apply ConfigMap:**
    ```bash
    kubectl apply -f game-config.yaml
    ```

- **Using ConfigMap in a Pod:**
  - **`pod-with-configmap.yaml`:**
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: game-pod
    spec:
      containers:
      - name: game-container
        image: busybox
        env:
        - name: PLAYER
          valueFrom:
            configMapKeyRef:
              name: game-config
              key: player
        - name: LEVEL
          valueFrom:
            configMapKeyRef:
              name: game-config
              key: level
        command: ["sh", "-c", "echo $PLAYER; echo $LEVEL; sleep 3600"]
    ```

  - **Apply Pod:**
    ```bash
    kubectl apply -f pod-with-configmap.yaml
    ```

#### 5. Creating & Consuming Secrets

Secrets store sensitive information such as passwords, OAuth tokens, and SSH keys.

##### Example: Creating and Using a Secret

- **Create Secret from Literal Values:**
  ```bash
  kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secretpassword
  ```

- **Create Secret from a File:**
  - **`secret.yaml`:**
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: db-secret
    type: Opaque
    data:
      username: YWRtaW4=  # base64 encoded 'admin'
      password: c2VjcmV0cGFzc3dvcmQ=  # base64 encoded 'secretpassword'
    ```

  - **Apply Secret:**
    ```bash
    kubectl apply -f secret.yaml
    ```

- **Using Secret in a Pod:**
  - **`pod-with-secret.yaml`:**
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: secret-pod
    spec:
      containers:
      - name: secret-container
        image: busybox
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        command: ["sh", "-c", "echo $DB_USERNAME; echo $DB_PASSWORD; sleep 3600"]
    ```

  - **Apply Pod:**
    ```bash
    kubectl apply -f pod-with-secret.yaml
    ```

#### 6. ServiceAccounts

ServiceAccounts provide identities for processes running in Pods.

##### Example: Creating and Using a ServiceAccount

- **Create ServiceAccount:**
  - **`serviceaccount.yaml`:**
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: my-service-account
    ```

  - **Apply ServiceAccount:**
    ```bash
    kubectl apply -f

###  Services and Networking

### Networking in Kubernetes: NetworkPolicies, Services, and Ingress

#### 1. NetworkPolicies

NetworkPolicies allow you to control the traffic flow to and from Kubernetes resources.

##### Example: Basic NetworkPolicy

- **Deny All Traffic Except from a Specific Namespace:**
  - **`networkpolicy.yaml`:**
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: deny-all
      namespace: default
    spec:
      podSelector: {}
      policyTypes:
      - Ingress
      - Egress
      ingress:
      - from:
        - namespaceSelector:
            matchLabels:
              name: allowed-namespace
    ```

  - **Apply NetworkPolicy:**
    ```bash
    kubectl apply -f networkpolicy.yaml
    ```

#### 2. Services

Services in Kubernetes expose applications running on Pods to be accessible within or outside the cluster.

##### Example: Exposing an Application via a Service

1. **Deploy an Application:**
   - **`deployment.yaml`:**
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: myapp
     spec:
       replicas: 2
       selector:
         matchLabels:
           app: myapp
       template:
         metadata:
           labels:
             app: myapp
         spec:
           containers:
           - name: myapp
             image: nginx
             ports:
             - containerPort: 80
     ```

   - **Apply Deployment:**
     ```bash
     kubectl apply -f deployment.yaml
     ```

2. **Create a Service:**
   - **`service.yaml`:**
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: myapp-service
     spec:
       selector:
         app: myapp
       ports:
       - protocol: TCP
         port: 80
         targetPort: 80
       type: LoadBalancer
     ```

   - **Apply Service:**
     ```bash
     kubectl apply -f service.yaml
     ```

3. **Verify Service:**
   ```bash
   kubectl get services
   ```

   The output should show `myapp-service` with an external IP if using a cloud provider or with a ClusterIP otherwise.

#### 3. Ingress

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster.

##### Example: Using Ingress to Expose an Application

1. **Deploy an Ingress Controller:**
   (Assuming you are using NGINX Ingress Controller)
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
   ```

2. **Create an Ingress Resource:**
   - **`ingress.yaml`:**
     ```yaml
     apiVersion: networking.k8s.io/v1
     kind: Ingress
     metadata:
       name: myapp-ingress
       annotations:
         nginx.ingress.kubernetes.io/rewrite-target: /
     spec:
       rules:
       - host: myapp.example.com
         http:
           paths:
           - path: /
             pathType: Prefix
             backend:
               service:
                 name: myapp-service
                 port:
                   number: 80
     ```

   - **Apply Ingress Resource:**
     ```bash
     kubectl apply -f ingress.yaml
     ```

3. **Verify Ingress:**
   ```bash
   kubectl get ingress
   ```

   Make sure that DNS for `myapp.example.com` points to the Ingress controller's external IP.

#### Troubleshooting Access to Applications

1. **Check Pod and Service Status:**
   ```bash
   kubectl get pods
   kubectl get services
   ```

2. **Describe Resources for Details:**
   ```bash
   kubectl describe pod <pod-name>
   kubectl describe service <service-name>
   ```

3. **Check Ingress Controller Logs:**
   ```bash
   kubectl logs -n ingress-nginx <ingress-controller-pod>
   ```

4. **Test Connectivity:**
   - **From within the cluster:**
     ```bash
     kubectl exec -it <pod-name> -- curl http://myapp-service
     ```
   - **From outside the cluster:**
     ```bash
     curl http://<external-ip>  # if using LoadBalancer
     curl http://myapp.example.com  # if using Ingress with proper DNS
     ```

5. **Common Issues:**
   - **Service Not Exposed:**
     Ensure that the service type is correctly set (`ClusterIP`, `NodePort`, or `LoadBalancer`).
   - **Ingress Rules Not Applied:**
     Ensure that the Ingress resource has correct rules and paths.
   - **DNS Issues:**
     Verify that the DNS entry points to the correct external IP.

These examples provide a comprehensive overview of using NetworkPolicies, Services, and Ingress in Kubernetes. 
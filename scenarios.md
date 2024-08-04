
## Problem Statement

1. **Persistent Volume Claim**: Create a Persistent Volume Claim (PVC) named `jekyll-site` in the `development` namespace with:
   - Storage request of 1Gi
   - Access mode `ReadWriteMany`
   - Using storage class `local-storage`

2. **Role and RoleBinding**: Create a Role and a RoleBinding in the `development` namespace:
   - **Role**: Named `developer-role` with all permissions for `pods`, `persistentvolumeclaims`, and `services`.
   - **RoleBinding**: Named `developer-rolebinding` binding the `developer-role` to the user `martin`.

3. **Kubeconfig Update**:
   - Add user `martin` with client key and certificate.
   - Create a new context called `developer` with `user = martin` and `cluster = kubernetes`.
   - Set this context as the current context.

4. **Persistent Volume**: Ensure the `PersistentVolume` named `jekyll-site` matches the PVC request and has:
   - Capacity of 1Gi
   - Access mode `ReadWriteMany`
   - Storage class `local-storage`
   - Path `/site` on the node

5. **Pod Configuration**: Create a Pod named `jekyll` with:
   - An `initContainer` named `copy-jekyll-site` using image `kodekloud/jekyll` to run the command `jekyll new /site`.
   - A main container named `jekyll` using image `kodekloud/jekyll-serve`.
   - Both `initContainer` and main container using a volume named `site` that is bound to the PVC `jekyll-site`.

---

### Solution

#### 1. Persistent Volume Claim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: development
  name: jekyll-site
spec:
  accessModes: 
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
```

#### 2. Role and RoleBinding

**Role:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
  - apiGroups: [""]
    resources: ["pods", "persistentvolumeclaims", "services"]
    verbs: ["*"]
```

**RoleBinding:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: 
  namespace: development
  name: developer-rolebinding
subjects: 
  - kind: User
    name: "martin"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

#### 3. Update Kubeconfig

**User Configuration:**

```yaml
apiVersion: v1
kind: Config
users:
  - name: martin
    user:
      client-key: /root/martin.key
      client-certificate: /root/martin.crt
```

**Context Configuration:**

```yaml
apiVersion: v1
kind: Config
contexts:
  - name: developer
    context:
      cluster: kubernetes
      user: martin
```

**Set Context:**

```bash
kubectl config use-context developer
```

#### 4. Persistent Volume (PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jekyll-site
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  storageClassName: local-storage
  hostPath:
    path: /site
```

#### 5. Pod Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: jekyll
  labels:
    run: jekyll
spec:
  volumes:
    - name: site
      persistentVolumeClaim:
        claimName: jekyll-site
  initContainers:
    - name: copy-jekyll-site
      image: kodekloud/jekyll
      command: [ "jekyll", "new", "/site" ]
      volumeMounts:
        - name: site
          mountPath: /site
  containers:
    - name: jekyll
      image: kodekloud/jekyll-serve
      volumeMounts:
        - name: site
          mountPath: /site
```

### Summary

1. **Create PVC**: Define the PVC with storage and access requirements.
2. **Define Role and RoleBinding**: Grant necessary permissions to the user `martin`.
3. **Update kubeconfig**: Add the user and context, and set it as current.
4. **Create Persistent Volume**: Ensure it matches the PVC request.
5. **Deploy Pod**: Configure the Pod with initContainers and main containers, and use the PVC.

Apply these configurations using `kubectl apply -f <filename>.yaml` for each YAML file. Verify the setup with `kubectl get` and `kubectl describe` commands.

---




### **Problem Statement and Configuration Overview**

#### **Problem Statement**

You are encountering several issues when trying to initialize a Kubernetes cluster with `kubeadm`. The issues include:
1. Ports being in use, indicating that some components might already be running or not properly cleaned up.
2. Configuration files for Kubernetes components already existing, suggesting a previous initialization attempt.
3. Directories and files that are not empty or already present, which could be causing conflicts.

Additionally, you are trying to list the images used by Kubernetes components and verify their versions and configurations. You also need to ensure that Kubernetes components are correctly set up and running.

#### **Current State of Kubernetes Configuration**

1. **Kubeadm Config Images List**

   ```sh
   k8s.gcr.io/kube-apiserver:v1.23.13
   k8s.gcr.io/kube-controller-manager:v1.23.13
   k8s.gcr.io/kube-scheduler:v1.23.13
   k8s.gcr.io/kube-proxy:v1.23.13
   k8s.gcr.io/pause:3.6
   k8s.gcr.io/etcd:3.5.1-0
   k8s.gcr.io/coredns/coredns:v1.8.6
   ```

   **Note:** The remote version is much newer (v1.25.3), but you're using `stable-1.23` due to the fallback.

2. **Error Messages from Kubeadm Init**

   ```sh
   [preflight] Some fatal errors occurred:
        [ERROR Port-10259]: Port 10259 is in use
        [ERROR Port-10257]: Port 10257 is in use
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: /etc/kubernetes/manifests/kube-controller-manager.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-etcd.yaml]: /etc/kubernetes/manifests/etcd.yaml already exists
        [ERROR Port-10250]: Port 10250 is in use
        [ERROR Port-2379]: Port 2379 is in use
        [ERROR Port-2380]: Port 2380 is in use
        [ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
   ```

   These errors suggest that ports and files/directories expected by `kubeadm` are already in use or present.

3. **Kubeadm Commands**

   - `kubeadm init --cluster-dns=100.64.0.10` was attempted, but failed due to pre-flight errors.
   - `kubeadm upgrade apply v1.8.6 --feature-gates=CoreDNS=true` was run, but this should typically be used for upgrading rather than initialization.

4. **File Listings**

   - **/etc/kubernetes/**

     ```sh
     total 36K
     -rw------- 1 root root 5.6K Oct 15 08:31 admin.conf
     -rw------- 1 root root 5.6K Oct 15 08:31 controller-manager.conf
     -rw------- 1 root root 2.0K Oct 15 08:31 kubelet.conf
     drwxr-xr-x 1 root root 4.0K Oct 15 08:58 manifests
     drwxr-xr-x 3 root root 4.0K Oct 15 08:31 pki
     -rw------- 1 root root 5.5K Oct 15 08:31 scheduler.conf
     ```

   - **/etc/kubernetes/pki**

     ```sh
     total 60K
     -rw-r--r-- 1 root root 1.3K Oct 15 08:31 apiserver.crt
     -rw-r--r-- 1 root root 1.2K Oct 15 08:31 apiserver-etcd-client.crt
     -rw------- 1 root root 1.7K Oct 15 08:31 apiserver-etcd-client.key
     -rw------- 1 root root 1.7K Oct 15 08:31 apiserver.key
     -rw-r--r-- 1 root root 1.2K Oct 15 08:31 apiserver-kubelet-client.crt
     -rw------- 1 root root 1.7K Oct 15 08:31 apiserver-kubelet-client.key
     -rw-r--r-- 1 root root 1.1K Oct 15 08:31 ca.crt
     -rw------- 1 root root 1.7K Oct 15 08:31 ca.key
     drwxr-xr-x 2 root root 4.0K Oct 15 08:31 etcd
     -rw-r--r-- 1 root root 1.1K Oct 15 08:31 front-proxy-ca.crt
     -rw------- 1 root root 1.7K Oct 15 08:31 front-proxy-ca.key
     -rw-r--r-- 1 root root 1.1K Oct 15 08:31 front-proxy-client.crt
     -rw------- 1 root root 1.7K Oct 15 08:31 front-proxy-client.key
     -rw------- 1 root root 1.7K Oct 15 08:31 sa.key
     -rw------- 1 root root  451 Oct 15 08:31 sa.pub
     ```

   - **/etc/kubernetes/manifests/**

     ```sh
     total 16K
     -rw------- 1 root root 2.2K Oct 15 08:31 etcd.yaml
     -rw------- 1 root root 3.8K Oct 15 08:58 kube-apiserver.yaml
     -rw------- 1 root root 3.3K Oct 15 08:31 kube-controller-manager.yaml
     -rw------- 1 root root 1.5K Oct 15 08:31 kube-scheduler.yaml
     ```

5. **Docker Containers**

   ```sh
   CONTAINER ID        IMAGE
   3dc02fb5f0a8        k8s.gcr.io/pause:3.6
   5b60f872b5fa        37c6aeb3663b
   d3d172ba190e        56c5af1d00b5
   ec72da1648e6        k8s.gcr.io/pause:3.6
   c0eecf38d0f9        df29c0a4002c
   2229813b4815        weaveworks/weave-npc
   e3b720ec9110        e03484a90585
   ded98bc59571        k8s.gcr.io/pause:3.6
   db3f6fd95d97        k8s.gcr.io/pause:3.6
   dc5786b0df60        25f8c7f3da61
   9a80080e2e10        k8s.gcr.io/pause:3.6
   cff972d02330        k8s.gcr.io/pause:3.6
   51e136c16208        k8s.gcr.io/pause:3.6
   ```

6. **Running Kubelet Process**

   ```sh
   root        3708  0.0  0.0 4236676 108476 ?      Ssl  08:31   2:12 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.6
   ```

### **Recommendations**

1. **Resolve Port Conflicts:**
   - Check which processes are using the conflicting ports (e.g., `10257`, `10259`, `10250`, `2379`, `2380`) and stop them if necessary.

2. **Clean Up Existing Configurations:**
   - If you need to reinitialize the cluster, remove existing Kubernetes manifest files and configurations:
     ```sh
     sudo rm -f /etc/kubernetes/manifests/*.yaml
     sudo rm -rf /var/lib/etcd
     ```

3. **Verify and Restart Docker Containers:**
   - Ensure that all necessary Docker containers are running and configured correctly. Restart Docker if needed:
     ```sh
     sudo systemctl restart docker
     ```

4. **Re-initialize Kubernetes:**
   - After resolving the above issues, attempt to reinitialize Kubernetes using `kubeadm init`.

5. **Check Kubelet Status:**
   - Ensure that the kubelet process is running correctly and that its configuration is up to date.

6. **Verify Configuration Files:**
   - Make sure that configuration files are correct and

 do not contain errors.

By following these steps, you should be able to resolve the issues with your Kubernetes cluster initialization and configuration. 


Here's a breakdown of the commands and configurations you've executed:

### **1. Kubernetes Cluster Initialization**

You ran the following command to initialize the Kubernetes cluster:

```sh
kubeadm init --apiserver-advertise-address=172.16.16.100 --pod-network-cidr=192.168.0.0/16 --ignore-preflight-errors=all
```

- **`--apiserver-advertise-address=172.16.16.100`**: Sets the IP address for the API server.
- **`--pod-network-cidr=192.168.0.0/16`**: Configures the CIDR for the pod network.
- **`--ignore-preflight-errors=all`**: Ignores all pre-flight errors, which can be useful if you are confident that the issues are non-critical.

### **2. Deploy Calico Network**

You deployed the Calico network plugin using:

```sh
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

- This command sets up Calico as the network plugin for your cluster.

### **3. Expose Deployment**

You exposed the `vote-deployment` in various ways:

1. **Service with NodePort and TCP Exposure:**

   ```sh
   kubectl expose deployment vote-deployment --tcp=5000:80 --node-port=31000 --name=vote-service
   ```

   - **`--tcp=5000:80`**: Exposes port 80 of the deployment on port 5000.
   - **`--node-port=31000`**: Exposes the service on node port 31000.

2. **Service with NodePort and Overriding API Version:**

   ```sh
   kubectl expose --type=NodePort deployment vote-deployment --port 80 --name vote-service --overrides '{ "apiVersion": "v1","spec":{"ports": [{"port":80,"protocol":"TCP","targetPort":80,"nodePort":30080}]}}'
   ```

   - **`--type=NodePort`**: Exposes the service on a NodePort.
   - **`--port 80`**: Service port.
   - **`--nodePort=30080`**: Exposes the service on node port 30080.

3. **Simple Service Exposure:**

   ```sh
   kubectl expose deployment vote-deployment --name=vote-service
   ```

   - Exposes the deployment with default settings, typically using a ClusterIP service.

### **Summary and Next Steps**

- **Cluster Initialization**: You successfully initialized the cluster. Ensure that all components are running and check the cluster status with `kubectl get nodes`.

- **Network Plugin**: Calico is deployed as the network plugin. Verify its status with:
  ```sh
  kubectl get pods -n kube-system
  ```

- **Service Exposure**: You have exposed the `vote-deployment` using various methods. Check the services and their assigned ports with:
  ```sh
  kubectl get services
  ```

Make sure to test the services to verify that they are accessible on the specified ports. 


The commands you’ve provided are used to extract and export values related to the Istio ingress gateway service, which is often used for managing incoming traffic to services in a Kubernetes cluster. Here’s a breakdown of each command:

### **1. Export Ingress Host**

```sh
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

- **Purpose**: Retrieves the external IP address of the Istio ingress gateway.
- **Command Explanation**: 
  - `kubectl -n istio-system get service istio-ingressgateway`: Gets details about the `istio-ingressgateway` service in the `istio-system` namespace.
  - `-o jsonpath='{.status.loadBalancer.ingress[0].ip}'`: Extracts the IP address from the service's status.
  - `export INGRESS_HOST=`: Sets the retrieved IP address to the `INGRESS_HOST` environment variable.

### **2. Export Ingress Port**

```sh
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
```

- **Purpose**: Retrieves the NodePort for the HTTP2 port of the Istio ingress gateway.
- **Command Explanation**: 
  - `kubectl -n istio-system get service istio-ingressgateway`: Gets details about the `istio-ingressgateway` service.
  - `-o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'`: Extracts the NodePort for the `http2` port.
  - `export INGRESS_PORT=`: Sets the retrieved NodePort to the `INGRESS_PORT` environment variable.

### **3. Export Secure Ingress Port**

```sh
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```

- **Purpose**: Retrieves the NodePort for the HTTPS port of the Istio ingress gateway.
- **Command Explanation**: 
  - `kubectl -n istio-system get service istio-ingressgateway`: Gets details about the `istio-ingressgateway` service.
  - `-o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'`: Extracts the NodePort for the `https` port.
  - `export SECURE_INGRESS_PORT=`: Sets the retrieved NodePort to the `SECURE_INGRESS_PORT` environment variable.

### **Usage**

After running these commands, you will have the external IP address and ports for the Istio ingress gateway stored in the environment variables `INGRESS_HOST`, `INGRESS_PORT`, and `SECURE_INGRESS_PORT`. You can use these variables to configure or test your applications' access through the ingress gateway. 

For example, to check if your services are accessible through the ingress, you could use the following `curl` commands:

```sh
curl http://$INGRESS_HOST:$INGRESS_PORT
curl https://$INGRESS_HOST:$SECURE_INGRESS_PORT
```

These commands will help you verify that the ingress gateway is properly routing traffic to your services.
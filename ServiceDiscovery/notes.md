# 🌐 Kubernetes Service Discovery: NodePort & ClusterIP

---

## 📌 Introduction to Service Discovery

In Kubernetes, **Pods are ephemeral**—they are created, destroyed, and rescheduled dynamically. Whenever a Pod is recreated, it receives a new, unpredictable internal IP address. Directly connecting to Pod IP addresses is therefore highly unreliable.

To solve this, Kubernetes provides **Services**. A Service acts as a stable networking abstraction and load balancer in front of a set of Pods. It routes traffic dynamically to any healthy Pod matching its **label selector**.

---

## 🚪 1. NodePort Service (External Access for Prototyping)

### 💡 Concept & How it Works
A **NodePort Service** exposes an application to external traffic outside the cluster.

- **Ephemeral Pod IPs**: Pod IP addresses change frequently when Pods restart. Directly accessing a Pod by its IP is brittle and unmaintainable.
- **Static Node Port Assignment**: A NodePort service allocates a static port directly on every Node in the cluster, typically in the range **`30000` to `32767`** (or `30000-32000`).
- **Traffic Routing**: When external traffic arrives at `localhost:<node-port>` (or `<node-ip>:<node-port>`), the NodePort service intercept this traffic, evaluates its **label selector** to find matching Pods, and forwards the request to the matching Pod's **`targetPort`**.

> ⚠️ **DevOps / Production Warning**: NodePort is intended **strictly for local prototyping and development** (e.g., Minikube, Docker Desktop). It is **not used in production** due to severe security risks (exposing high ports publicly) and scaling limitations (port collision and lack of advanced load balancing). In production, **LoadBalancer** or **Ingress** controllers are used instead.

### 🖼️ NodePort Service Flow Diagram

![Kubernetes NodePort Service Flow](./images/nodeport-flow.svg)

---

## 🔒 2. ClusterIP Service (Internal Pod-to-Pod Communication)

### 💡 Concept & How it Works
A **ClusterIP Service** is the default Kubernetes service type and is used exclusively for **internal pod-to-pod communication** inside the cluster network.

- **Internal Cluster IP & DNS**: Kubernetes assigns a stable, internal-only Virtual IP address (ClusterIP) to the Service, which also resolves automatically via internal Cluster DNS using the **Service Name** (e.g., `grade-submission-api-service`).
- **Request Flow**: A calling Pod (such as a frontend web portal) reaches the target application (such as a backend REST API) by issuing a request directly to the **Service Name** and **Service Port**. The ClusterIP service receives the request, queries its label selector, and load-balances the traffic to the destination Pod's **`targetPort`**.
- **Environment Variable Configuration**: The calling application is typically configured with an environment variable pointing directly to the service name (e.g., `API_URL=http://grade-submission-api-service:3000`). From the calling application's perspective, this abstracted DNS setup feels like a **direct connection**.

### 🖼️ ClusterIP Service Flow Diagram

![Kubernetes ClusterIP Service Flow](./images/clusterip-flow.svg)

---

## 📄 3. Manifest Architecture & YAML Files

Below are the two corresponding service manifests created for our application:

### 1. NodePort Service Manifest (`grade-submission-portal-service.yaml`)
Exposes the frontend portal on static node port `30000`, forwarding to `targetPort: 5001`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grade-submission-portal-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/component: frontend
    app.kubernetes.io/instance: grade-submission-portal
  ports:
    - port: 80
      targetPort: 5001
      nodePort: 30000
```

### 2. ClusterIP Service Manifest (`grade-submission-api-service.yaml`)
Exposes the backend API internally within the cluster on port `3000`, matching Pods labeled `component: backend`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grade-submission-api-service
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/component: backend
    app.kubernetes.io/instance: grade-submission-api
  ports:
    - port: 3000
      targetPort: 3000
```

---

## 🛠️ 4. Commands to Run, Observe & Verify Service Discovery

Use the following `kubectl` commands to deploy the Pods and Services, inspect how label selectors bind Endpoints, and test internal/external connectivity.

### 🚀 Step 1: Deploy Pods and Services

Apply all manifests in the `ServiceDiscovery` folder:

```bash
# Apply all Pods and Services in the directory
kubectl apply -f .

# Alternatively, apply manifests individually:
kubectl apply -f grade-submission-api-pod.yaml
kubectl apply -f grade-submission-portal-pod.yaml
kubectl apply -f grade-submission-api-service.yaml
kubectl apply -f grade-submission-portal-service.yaml
```

---

### 🔍 Step 2: Observe Active Resources & Service Bindings

#### A. List all running Pods (with IP address details):
```bash
kubectl get pods -o wide
```
*Expected Output*: Displays Pod status, individual assigned Pod IP addresses (e.g., `10.244.0.5`), and host node assignment.

#### B. List all active Services:
```bash
kubectl get services
# OR
kubectl get svc
```
*Expected Output*: Displays `grade-submission-api-service` with type `ClusterIP` on port `3000/TCP`, and `grade-submission-portal-service` with type `NodePort` mapping `80:30000/TCP`.

#### C. Inspect Service Endpoints (Label Selector Verification):
Kubernetes automatically populates **Endpoints** by matching Service label selectors to Pod labels.

```bash
# View all Endpoints mapped to Pod IPs
kubectl get endpoints
# OR
kubectl get ep

# Detailed view of ClusterIP Service Endpoints
kubectl describe svc grade-submission-api-service

# Detailed view of NodePort Service Endpoints
kubectl describe svc grade-submission-portal-service
```
*Key Observation*: Check the `Endpoints:` line in `kubectl describe`. It should show the exact IP address and target port of the target Pod. If `Endpoints: <none>`, the Service label selector does not match any running Pod's labels!

---

### 🧪 Step 3: Test and Verify Service Connectivity

#### A. Test NodePort Externally (Browser / Localhost):
Open your web browser or run `curl` to access the NodePort service from your workstation:

```bash
curl http://localhost:30000
```
*For Minikube users*:
```bash
minikube service grade-submission-portal-service --url
```

#### B. Test ClusterIP Internal Communication (Pod-to-Pod & DNS):
Verify internal DNS resolution and service discovery from inside the Portal Pod to the API ClusterIP Service:

```bash
# 1. Test HTTP connectivity using Service Name & Service Port
kubectl exec -it grade-submission-portal -- curl http://grade-submission-api-service:3000

# 2. Verify Cluster DNS resolution for the ClusterIP Service Name
kubectl exec -it grade-submission-portal -- nslookup grade-submission-api-service
```
*Key Observation*: `nslookup` returns the virtual ClusterIP assigned to `grade-submission-api-service`. The Portal Pod communicates seamlessly using the service hostname without ever knowing the underlying Pod's ephemeral IP!

---

### 🧹 Step 4: Cleanup Resources

Remove all created Pods and Services when testing is complete:

```bash
kubectl delete -f .
```

---

## 🧪 5. Quiz & Real-World Interview Questions & Answers

### 🧠 Quick Self-Assessment Quiz

<details>
<summary><b>Q1: Why is connecting directly to a Pod's internal IP address considered an anti-pattern in production?</b></summary>
<br/>
<b>Answer:</b> Pods are ephemeral. When a Pod restarts, crashes, or scales, Kubernetes assigns it a brand-new IP address. Connecting directly to Pod IPs causes connection failures whenever Pods are recreated. Services provide a stable Virtual IP and DNS name that survives Pod restarts.
</details>

<details>
<summary><b>Q2: What is the default range of ports allocated for NodePort services in Kubernetes?</b></summary>
<br/>
<b>Answer:</b> The default reserved NodePort range is <b><code>30000</code> to <code>32767</code></b>.
</details>

<details>
<summary><b>Q3: Is a ClusterIP service accessible from outside the Kubernetes cluster by default?</b></summary>
<br/>
<b>Answer:</b> <b>No.</b> ClusterIP services are assigned an internal virtual IP reachable only from within the cluster network (internal Pod-to-Pod communication).
</details>

<details>
<summary><b>Q4: What is the difference between `port` and `targetPort` in a Kubernetes Service manifest?</b></summary>
<br/>
<b>Answer:</b> 
<ul>
  <li><b><code>port</code></b>: The port number exposed by the Service object itself inside the cluster.</li>
  <li><b><code>targetPort</code></b>: The port number on the container inside the Pod where application traffic is actually forwarded.</li>
</ul>
</details>

<details>
<summary><b>Q5: What underlying Kubernetes object bridges a Service to matching Pod IPs?</b></summary>
<br/>
<b>Answer:</b> An <b>Endpoints</b> object (or EndpointSlice in newer K8s versions). Kubernetes continuously evaluates the Service's <code>selector</code> and populates Endpoints with the active IP addresses and target ports of healthy matching Pods.
</details>

---

### 💼 Top DevOps / Kubernetes Interview Questions & Answers

#### Q1: "How does Kubernetes Service Discovery work, and how does internal CoreDNS resolve Service names?"
> **Answer:** 
> When a Service is created, Kubernetes assigns it a stable ClusterIP address and CoreDNS automatically creates a DNS record for it in the format:
> `<service-name>.<namespace>.svc.cluster.local` (e.g. `grade-submission-api-service.default.svc.cluster.local`).
> Any Pod in the cluster can issue a request using short name (e.g. `http://grade-submission-api-service:3000`), which CoreDNS resolves to the ClusterIP. `kube-proxy` then intercepts the packet and routes it to an active Pod IP listed in the Service's Endpoints.

#### Q2: "Compare NodePort, ClusterIP, and LoadBalancer service types. When would you use each?"
> **Answer:**
> - **ClusterIP**: Default internal service type. Use for backend databases, APIs, or internal microservices that should never be exposed publicly.
> - **NodePort**: Opens a static high port (`30000-32767`) on every cluster node. Use for local development/testing (Minikube/Kind) or quick debugging. Not recommended for production.
> - **LoadBalancer**: Provisions an external Cloud Load Balancer (AWS ALB/NLB, GCP Cloud LB, Azure LB) that routes public traffic directly into the cluster. Standard choice for production public workloads.

#### Q3: "What does it mean when `kubectl describe service <name>` shows `Endpoints: <none>`? How do you troubleshoot and fix it?"
> **Answer:**
> `Endpoints: <none>` means the Service cannot find any healthy running Pods matching its label selector.
> **Troubleshooting Steps:**
> 1. Run `kubectl get pods --show-labels` and compare Pod labels against `kubectl get svc <service-name> -o yaml` -> `spec.selector`.
> 2. Ensure labels match **exact key-value pairs** (case-sensitive).
> 3. Verify Pod status is `Running` and passing its `readinessProbe` (unready Pods are stripped from Endpoints).

#### Q4: "How does traffic flow from an external client to a Pod when using a NodePort service?"
> **Answer:**
> 1. Client sends HTTP request to `http://<NodeIP>:30000`.
> 2. The host Node's kernel intercepts traffic on port 30000 via `kube-proxy` rules (iptables or IPVS).
> 3. `kube-proxy` selects a healthy target Pod IP from the Service Endpoints (performing load balancing across nodes if necessary).
> 4. Traffic is forwarded to the destination Pod's `containerPort` (e.g. port `5001`).

#### Q5: "What is `externalTrafficPolicy: Local` vs `Cluster` in a NodePort or LoadBalancer service?"
> **Answer:**
> - **`Cluster`** (Default): Traffic arriving at a node may be forwarded to a target Pod running on a *different* node (adds extra network hop, hides original client IP).
> - **`Local`**: Traffic arriving at a node is *only* routed to Pods running on that specific node. Preserves the original client source IP address and avoids extra network hops, but risks uneven traffic distribution if Pods aren't evenly distributed.

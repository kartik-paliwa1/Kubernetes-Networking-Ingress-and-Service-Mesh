# Kubernetes Networking, Ingress, and Service Mesh

## Overview

This project focuses on Kubernetes networking and traffic management used in real production systems.
The goal is to control how traffic flows between pods, how users access applications, and how services communicate securely inside the cluster.

This project covers network isolation, ingress routing, canary releases, and basic service mesh usage.

---

## Step 1: Verify Pod Networking

Check CNI pods.

```bash
kubectl get pods -n kube-system
```

Check pod IP addresses.

```bash
kubectl get pods -n dev -o wide
```

---

## Step 2: Test Pod-to-Pod Communication

Create a test pod.

```bash
kubectl run net-test \
--image=busybox \
--restart=Never \
--sleep 3600 \
-n dev
```

Enter the pod.

```bash
kubectl exec -it net-test -n dev -- sh
```

Test backend access.

```bash
wget -qO- http://backend-service
```

Exit.

```bash
exit
```

---

## Step 3: Apply Default Deny Network Policy

Create `deny-all.yaml`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: dev
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Apply policy.

```bash
kubectl apply -f deny-all.yaml
```

---

## Step 4: Allow Frontend to Backend Traffic

Create `allow-frontend.yaml`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: dev
spec:
  podSelector:
    matchLabels:
      app: backend-api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

Apply policy.

```bash
kubectl apply -f allow-frontend.yaml
```

---

## Step 5: Verify Network Policy

Enter test pod again.

```bash
kubectl exec -it net-test -n dev -- sh
```

Test backend access.

```bash
wget -qO- http://backend-service
```

Connection should fail.

---

## Step 6: Configure Ingress Routing

Create `ingress.yaml`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

Apply ingress.

```bash
kubectl apply -f ingress.yaml
```

Verify.

```bash
kubectl get ingress -n dev
```

---

## Step 7: Canary Deployment

Create second frontend deployment.

```bash
kubectl create deployment frontend-v2 --image=nginx -n dev
```

Expose service.

```bash
kubectl expose deployment frontend-v2 \
--name=frontend-v2-service \
--port=80 \
-n dev
```

Create `canary-ingress.yaml`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-canary
  namespace: dev
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-v2-service
            port:
              number: 80
```

Apply.

```bash
kubectl apply -f canary-ingress.yaml
```

---

## Step 8: Install Istio

Download Istio.

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
```

Install Istio.

```bash
./bin/istioctl install --set profile=demo -y
```

Verify installation.

```bash
kubectl get pods -n istio-system
```

---

## Step 9: Enable Sidecar Injection

Label namespace.

```bash
kubectl label namespace dev istio-injection=enabled
```

Restart deployments.

```bash
kubectl rollout restart deployment frontend -n dev
kubectl rollout restart deployment backend-api -n dev
```

Verify pods.

```bash
kubectl get pods -n dev
```

---

## Step 10: Apply Istio Traffic Policy

Create `destination-rule.yaml`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: frontend
  namespace: dev
spec:
  host: frontend-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

Apply.

```bash
kubectl apply -f destination-rule.yaml
```

---

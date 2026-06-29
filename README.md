# Exercise 9 – Prometheus Monitoring Failure

## Overview

This project demonstrates troubleshooting a **Prometheus Monitoring Failure** in an Amazon EKS cluster.

The incident simulates a real-world scenario where Prometheus is unable to scrape application metrics because of a **ServiceMonitor configuration mismatch**. The investigation includes reproducing the issue, identifying the root cause, fixing the configuration, and verifying the results.

---

# Objective

* Deploy Prometheus and Grafana using Helm
* Deploy a sample application
* Create a ServiceMonitor
* Reproduce a monitoring failure
* Investigate the root cause
* Fix the issue
* Verify monitoring

---

# Architecture

```
Application Pod
       │
       ▼
Kubernetes Service
       │
       ▼
ServiceMonitor
       │
       ▼
Prometheus
       │
       ▼
Grafana
```

---

# Technologies Used

* Amazon EKS
* Kubernetes
* Helm
* Prometheus
* Grafana
* ServiceMonitor
* AWS CLI
* kubectl

---

# Project Structure

```
Prometheus-Monitoring-Failure
│
├── manifests
│   ├── payment-app.yaml
│   └── servicemonitor.yaml
│
├── screenshots
│   ├── 01-prometheus-installed.png
│   ├── 02-payment-app-running.png
│   ├── 03-no-targets.png
│   ├── 04-target-down.png
│   └── 05-target-fixed.png
│
└── README.md
```

---

# Project Phases

## Phase 1 – Create EKS Cluster

Created an Amazon EKS cluster with managed worker nodes.

Verification:

```bash
kubectl get nodes
```

---

## Phase 2 – Install Prometheus Stack

Installed kube-prometheus-stack using Helm.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

kubectl create namespace monitoring

helm install monitoring prometheus-community/kube-prometheus-stack \
--namespace monitoring \
--timeout 20m \
--wait
```

Verification:

```bash
kubectl get pods -n monitoring
```

All monitoring components became healthy.

---

## Phase 3 – Deploy Sample Application

Deployed a sample payment service.

Verification:

```bash
kubectl get pods
kubectl get svc
```

---

## Phase 4 – Reproduce Failure

Created a ServiceMonitor with an incorrect port name.

Service:

```yaml
ports:
- name: prometheus
```

ServiceMonitor:

```yaml
endpoints:
- port: metrics
```

Because the ServiceMonitor searched for **metrics** while the Service exposed **prometheus**, Prometheus could not discover the target.

Observed Result:

```
No active targets
```

---

## Phase 5 – Investigation

Commands used during troubleshooting:

```bash
kubectl get servicemonitor

kubectl describe servicemonitor payment-service

kubectl get svc payment-service -o yaml

kubectl get endpoints payment-service

kubectl logs deployment/monitoring-kube-prometheus-operator -n monitoring
```

Investigation Findings:

* ServiceMonitor existed
* Service existed
* Endpoints existed
* Port names were different

---

# Root Cause

The ServiceMonitor referenced:

```yaml
port: metrics
```

The Service exposed:

```yaml
name: prometheus
```

Because Prometheus matches ports by **name**, it could not discover any scrape targets.

---

## Phase 6 – Fix

Updated the ServiceMonitor.

Changed

```yaml
port: metrics
```

to

```yaml
port: prometheus
```

Applied the updated manifest.

```bash
kubectl apply -f servicemonitor.yaml
```

---

## Phase 7 – Verification

After correcting the ServiceMonitor, Prometheus successfully discovered the target.

A new issue appeared:

```
404 Not Found
```

This occurred because the sample application used the standard **NGINX** image, which does not expose a `/metrics` endpoint.

Prometheus successfully reached the application, confirming that the ServiceMonitor configuration issue had been resolved.

---

# Incident Summary

| Component            | Status                            |
| -------------------- | --------------------------------- |
| Kubernetes Service   | ✅ Healthy                         |
| ServiceMonitor       | ✅ Healthy                         |
| Prometheus Discovery | ✅ Successful                      |
| Metrics Endpoint     | ❌ Missing (/metrics returned 404) |

---

# Lessons Learned

* How Prometheus discovers Kubernetes services
* How ServiceMonitors work
* Importance of matching Service port names
* Difference between target discovery failures and application metric failures
* Troubleshooting Prometheus target health
* Using Helm to deploy the Prometheus Operator stack

---

# Commands Used

```bash
kubectl get pods

kubectl get svc

kubectl get servicemonitor

kubectl describe servicemonitor

kubectl get endpoints

kubectl logs deployment/monitoring-kube-prometheus-operator -n monitoring

kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring

helm install monitoring prometheus-community/kube-prometheus-stack \
--namespace monitoring \
--timeout 20m \
--wait
```

---

# Screenshots

Include screenshots demonstrating:

* EKS cluster creation
* Prometheus installation
* Payment application deployment
* ServiceMonitor creation
* No Targets observed
* Target DOWN (404 Not Found)
* Updated ServiceMonitor
* Prometheus Targets page after the fix

---

# Conclusion

This exercise successfully reproduced a Prometheus monitoring incident caused by a ServiceMonitor configuration mismatch. Through systematic investigation, the incorrect port mapping was identified and corrected, allowing Prometheus to discover the application target. The final 404 response highlighted that successful target discovery and application metric exposure are separate concerns, emphasizing the importance of validating both monitoring configuration and application instrumentation.

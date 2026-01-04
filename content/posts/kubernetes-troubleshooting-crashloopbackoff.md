---
title: "Kubernetes Troubleshooting: Fixing the Dreaded CrashLoopBackOff"
date: 2025-06-15
summary: "A practical guide to debugging Kubernetes pods, analyzing logs, and fixing common container startup failures."
tags: ["Kubernetes", "DevOps", "Troubleshooting", "kubectl", "Debug"]
cover:
  image: "https://kubernetes.io/images/docs/pod-lifecycle.png" # Standard K8s Lifecycle image
  alt: "Kubernetes Pod Lifecycle"
  caption: "Understanding where pods fail in the lifecycle"
---

## The Scenario

You deploy your application to the cluster. The deployment says "Available: 0/1". You run `kubectl get pods`, and you see the most frustrating status in the cloud native world:

`NAME: my-app-84728  STATUS: CrashLoopBackOff  RESTARTS: 5`

This means your container started, crashed, restarted, crashed again, and now Kubernetes is backing off (waiting longer between restarts) to save resources. Here is my standard 4-step workflow to fix it.

## Step 1: The Logs Are Your First Responder

The first question is: *Did the application code actually run?*

If the app crashed because of a missing environment variable or a syntax error, standard output (`stdout`) will tell you.

```bash
# Check logs for the crashing pod
kubectl logs my-app-84728

# If the pod has restarted multiple times, check the PREVIOUS instance
kubectl logs my-app-84728 --previous
```
### Common Findings:

1. **`KeyError: 'DB_PASSWORD'`** -> You forgot to mount a Secret.

2. **`Connection Refused`** -> Your database isn't reachable or the service name is wrong.

## Step 2: The "Describe" Command
If `logs` are empty, the container might not even be starting. `kubectl describe` tells you what the **Node** thinks happened.

```bash
kubectl describe pod my-app-84728
```
Scroll to the bottom **Events** section. This is the "Black Box" recorder of Kubernetes.

Look for these specific errors:

1. **OOMKilled:** Your container used more memory than the limit allowed. Fix: *`Increase resources.limits.memory`*.

2. **MountVolume.SetUp failed:** The Azure Disk or AWS EBS volume is stuck on another node.

3. **ImagePullBackOff:** You typo'd the image name, or your ACR/ECR credentials aren't set up.

## Step 3: Check Your Probes
Sometimes the app starts fine, runs for 30 seconds, and then restarts. This is almost always a **Liveness Probe** failure.

If your application takes 60 seconds to warm up (e.g., Java Spring Boot), but your Liveness Probe starts checking after 10 seconds, Kubernetes assumes the app is dead and kills it.

**The Fix:** Check your deployment YAML and increase the `initialDelaySeconds`.
```hcl
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 60 # Give it time to breathe!
  periodSeconds: 10
```

## Step 4: The "Debug Container" Technique
If you are still stuck, you need to get inside the cluster. Since the container keeps crashing, you can't use `kubectl exec`.

Instead, launch a temporary "debug" container that shares the namespace but uses a stable image (like Alpine or Ubuntu) to poke around.

```bash
# Run a temporary curl/network tool in the same namespace
kubectl run -it --rm debug-pod --image=curlimages/curl -- sh

# Now you can test internal networking
curl http://my-database-service:5432
```

## Summary
`CrashLoopBackOff` is rarely a mystery; it's usually a configuration mismatch. By systematically checking Logs -> Events -> Resources -> Networking, you can isolate 99% of failures in minutes.
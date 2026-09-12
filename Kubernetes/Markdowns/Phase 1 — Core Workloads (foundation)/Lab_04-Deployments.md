# Kubernetes Hands-On Lab 04 — Deployments

## 4.1 Objectives

By the end of this lab, you should be able to:

- Create a Deployment using YAML
- Understand Deployment → ReplicaSet → Pod
- Scale a Deployment
- Update container images
- Perform rolling updates
- Monitor rollout status
- View rollout history
- Roll back a Deployment
- Pause and resume a rollout
- Troubleshoot a failed Deployment
- Perform common Deployment tasks quickly for the CKA

## 4.2 Architecture

```
                    Deployment
                        |
                        v
                   ReplicaSet
                        |
              +---------+---------+
              |         |         |
              v         v         v
            Pod-1     Pod-2     Pod-3
```

- The Deployment manages the ReplicaSet.
- The ReplicaSet maintains the desired number of Pods.

## 4.3 Prerequisites

You need:

```bash
kubectl version --client
kubectl get nodes
```

Verify that your cluster is available:

```bash
kubectl get nodes
```

Expected:

```
NAME           STATUS   ROLES           AGE   VERSION
control-plane  Ready    control-plane   ...   ...
worker         Ready    <none>          ...   ...
```

## 4.4 Lab 1 — Create a Deployment

Create the following file: [`deployment.yaml`](./deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f deployment.yaml
```

## 4.5 Verify the Deployment

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

Or:

```bash
kubectl get deploy,rs,pods
```

Expected relationship:

```
Deployment
    |
    +-- ReplicaSet
          |
          +-- nginx Pod
          +-- nginx Pod
          +-- nginx Pod
```

## 4.6 Inspect the Deployment

```bash
kubectl describe deployment nginx-deployment
```

Check:

- replicas
- selector
- image
- strategy
- available replicas
- conditions

## 4.7 Scale the Deployment

Scale from 3 replicas to 5:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Verify:

```bash
kubectl get pods
```

Then scale back:

```bash
kubectl scale deployment nginx-deployment --replicas=3
```

Verify:

```bash
kubectl get deployment nginx-deployment
```

## 4.8 Update the Container Image

Update NGINX:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

Check rollout:

```bash
kubectl rollout status deployment/nginx-deployment
```

Check Pods:

```bash
kubectl get pods
```

Check ReplicaSets:

```bash
kubectl get rs
```

You should see the Deployment create a new ReplicaSet.

## 4.9 Understand Rolling Updates

Check the Deployment:

```bash
kubectl describe deployment nginx-deployment
```

The default Deployment strategy is:

```yaml
strategy:
  type: RollingUpdate
```

You can explicitly define it — see [`deployment-rolling.yaml`](./deployment-rolling.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-rolling
spec:
  replicas: 4

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1

  selector:
    matchLabels:
      app: nginx-rolling

  template:
    metadata:
      labels:
        app: nginx-rolling

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment-rolling.yaml
```

Update the image:

```bash
kubectl set image deployment/nginx-rolling nginx=nginx:1.28
```

Watch the rollout:

```bash
kubectl rollout status deployment/nginx-rolling
```

In another terminal:

```bash
kubectl get pods -w
```

Observe the old Pods being replaced gradually.

## 4.10 Rollout History

Check the history:

```bash
kubectl rollout history deployment/nginx-deployment
```

You can inspect a particular revision:

```bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

## 4.11 Roll Back a Deployment

First check the current image:

```bash
kubectl get deployment nginx-deployment \
  -o=jsonpath='{.spec.template.spec.containers[0].image}'
```

Now update to another version:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:latest
```

Check:

```bash
kubectl rollout status deployment/nginx-deployment
```

Roll back:

```bash
kubectl rollout undo deployment/nginx-deployment
```

Verify:

```bash
kubectl rollout status deployment/nginx-deployment
```

Check the image:

```bash
kubectl get deployment nginx-deployment \
  -o=jsonpath='{.spec.template.spec.containers[0].image}'
```

## 4.12 Pause and Resume a Deployment

Pause:

```bash
kubectl rollout pause deployment/nginx-deployment
```

Make a change:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

Make another change:

```bash
kubectl set env deployment/nginx-deployment ENVIRONMENT=production
```

Resume:

```bash
kubectl rollout resume deployment/nginx-deployment
```

Check:

```bash
kubectl rollout status deployment/nginx-deployment
```

## 4.13 Break the Deployment

Now intentionally introduce a bad image:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:this-image-does-not-exist
```

Check:

```bash
kubectl get pods
```

You should see something similar to:

```
ImagePullBackOff
ErrImagePull
```

Investigate:

```bash
kubectl describe deployment nginx-deployment
```

Then:

```bash
kubectl get pods
```

Find the failing Pod and run:

```bash
kubectl describe pod <pod-name>
```

Check events:

```bash
kubectl get events --sort-by=.lastTimestamp
```

## 4.14 Recover the Deployment

Fix the image:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

Check:

```bash
kubectl rollout status deployment/nginx-deployment
```

Verify:

```bash
kubectl get deploy,rs,pods
```

## 4.15 Deployment YAML Modification Exercise

Modify `deployment.yaml` to:

- run 4 replicas
- use `nginx:1.28`
- expose container port 80
- add an environment variable named `ENVIRONMENT`
- set it to `dev`

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 4

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.28

          ports:
            - containerPort: 80

          env:
            - name: ENVIRONMENT
              value: "dev"
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get pods
```

## 4.16 CKA Practice Task

**Task**

Create a Deployment named:

```
webapp
```

Requirements:

- namespace: `default`
- replicas: `3`
- image: `nginx:1.28`
- container name: `nginx`
- container port: `80`

Then:

1. Verify all Pods are running.
2. Scale the Deployment to 5 replicas.
3. Change the image to `nginx:latest`.
4. Verify the rollout.
5. Roll back to the previous revision.

**Target Time**

5 minutes

Try it without looking at previous commands.

## 4.17 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What is the relationship between:

- Deployment
- ReplicaSet
- Pod

**Question 2**

Why does changing the container image create a new ReplicaSet?

**Question 3**

What happens when you scale a Deployment from 3 → 5?

**Question 4**

How would you troubleshoot:

- Deployment exists
- ReplicaSet exists
- Pods exist
- but Pods are not Ready

**Question 5**

How would you troubleshoot:

- `ImagePullBackOff`

**Question 6**

What is the difference between:

- `kubectl rollout undo`

and:

- `kubectl delete pod`

## 4.18 Useful Commands

```bash
kubectl get deployment
kubectl get deploy
kubectl describe deployment <name>

kubectl get rs
kubectl describe rs <name>

kubectl get pods
kubectl describe pod <name>

kubectl scale deployment <name> --replicas=5

kubectl set image deployment/<name> \
  <container>=<image>

kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout history deployment/<name> --revision=2
kubectl rollout undo deployment/<name>

kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>

kubectl get events --sort-by=.lastTimestamp
```

## 4.19 Cleanup

```bash
kubectl delete deployment nginx-deployment
kubectl delete deployment nginx-rolling
```

Or:

```bash
kubectl delete -f deployment.yaml
kubectl delete -f deployment-rolling.yaml
```

## 4.20 Lab Checklist

- [ ] Created Deployment using YAML
- [ ] Understood Deployment → ReplicaSet → Pod
- [ ] Scaled Deployment
- [ ] Updated container image
- [ ] Performed rolling update
- [ ] Viewed rollout history
- [ ] Rolled back Deployment
- [ ] Paused/resumed rollout
- [ ] Created an intentional failure
- [ ] Troubleshot ImagePullBackOff
- [ ] Recovered the Deployment
- [ ] Completed CKA task within 5 minutes

## Files for this lab

```
04-Deployments/
├── README.md
├── deployment.yaml
└── deployment-rolling.yaml
```

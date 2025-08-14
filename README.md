# 5G-Devops



✅ Step 1: Create a kind Cluster

kind create cluster --name hpa-demo

Wait for the cluster to be ready.
✅ Step 2: Install metrics-server

This is required for HPA to read CPU/memory metrics.
⚠️ metrics-server needs special config for kind due to TLS and networking

Run:

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

Then patch the deployment to allow insecure TLS (needed for kind):

kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'


Wait a few seconds and then confirm it's working:

kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods

If you see CPU/memory values — metrics-server is working.
✅ Step 3: Deploy a Sample App

Here’s an NGINX deployment that burns CPU artificially.
nginx-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
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
        image: k8s.gcr.io/hpa-example
        resources:
          limits:
            cpu: "500m"
          requests:
            cpu: "200m"
        ports:
        - containerPort: 80
       readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
 
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-deployment
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80


Apply it:

kubectl apply -f nginx-deployment.yaml

✅ Step 4: Create the HPA

Create an HPA that auto-scales the above deployment based on CPU usage.

kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=1 --max=10

View it:

kubectl get hpa

✅ Step 5: Generate Load to Trigger Scaling

Run a busybox pod to hammer the app:

kubectl run -i --tty load-generator --image=busybox /bin/sh

Inside the pod shell:

while true; do wget -q -O- http://nginx-deployment; done  # for nginx application

while true; do wget -q -O- https://fantastic-train-699pg6vjr9r5h477x-8989.app.github.dev/login; done # for my-app application



https://fantastic-train-699pg6vjr9r5h477x-8989.app.github.dev/login

This loop generates CPU load. Leave it running for a few minutes.

Then, in a new terminal, watch the HPA behavior:

kubectl get hpa -w

You should see the replica count increase based on CPU usage.
✅ Step 6: Clean Up

kind delete cluster --name hpa-demo

🔧 Optional: View Metrics

Use:

kubectl top pods
kubectl describe hpa
-----------------------------------------------------------------------------------------------------------------
If you're using autoscaling/v2 (which you are), there’s a default scale-down stabilization window of 300 seconds (5 minutes), meaning it won’t scale down immediately.

You can override it explicitly:

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpaautoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bankapp
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60


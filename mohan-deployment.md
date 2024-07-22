1. create a docker file:
```
sudo mkdir mohan-deployment
```
```
sudo nano Dockerfile
```
```
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 4499
CMD ["python", "app.py"]
```
save --> ctrl+s and ctrl+x

2. build a dockerfile
```
docker build -t wisecow -f Dockerfile .
```

3. Create a k8ns manifeast
```
sudo nano kubernetes-manifest.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app-deployment
  labels:
    app: python-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: python-app
  template:
    metadata:
      labels:
        app: python-app
    spec:
      containers:
      - name: python-app
        image: wisecow:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 4499
---
apiVersion: v1
kind: Service
metadata:
  name: python-app-service
spec:
  selector:
    app: python-app
  ports:
    - protocol: TCP
      port: 4499
      targetPort: 4499
  type: NodePort
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: python-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - python-app.local
    secretName: python-app-tls
  rules:
  - host: python-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: python-app-service
            port:
              number: 4499
```
4. start minikube and install ingress addon
```
minikube start
```
```
minikube addons enable ingress
```

5. Create a self signed TLS certificate
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=python-app.local/O=python-app.local"
```

6. create a secrets for TLS
```
kubectl create secret tls python-app-tls --key tls.key --cert tls.crt
```

7. add created Docker image to minikube env, since we dont have any container registry to pull image
```
minikube image load wisecow:latest
```

8. now execute the k8s manifeast file
```
kubectl apply -f kubernetes-manifest.yaml
```

9. check everything is working fine with below command(debug)
```
kubectl get deployments
kubectl describe deployment python-app-deployment
kubectl get pods
kubectl get services
kubectl get ingress
```
```
kubectl get service python-app-service -o yaml
```
```
kubectl get all -n default -o yaml
```
port forward of service(if required without ingress)
```
kubectl port-forward service/python-app-service 4499:4499
```

10. add ingress address to host file
```
minikube ip
```
```
sudo nano /etc/hosts
```
```
<minikube_ip> python-app.local
```

11. access the website using
```
curl -k https://python-app.local
```

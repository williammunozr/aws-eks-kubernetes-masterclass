# Kubernetes - Minikube

## Deploy the basic application on Minikube
```
# Create All Objects
kubectl apply -f minikube/

# List Pods
kubectl get pods

# List of Services
kubectl get svc

You should get: usermgmt-webapp-service

# Access the application locally
kubectl port-forward service/usermgmt-webapp-service 8080:80

# Access the application
http://localhost:8080

This is a learning environment
Username: admin101
Password: password101
```
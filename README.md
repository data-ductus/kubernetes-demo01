# Step 1. Setup the K8S cluster.


sudo su
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.79.10
on master

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubeadm join .... on agents

# Step 2. Setup overlay network

Show that nodes are "not ready".
wget https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml

args:
  --iface=enp0s8
kubectl apply -f kube-flannel.yml
Deploys DNS containers..
Nodes gets ready <none> no label yet

# Step 4. Deploy your first pod

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.15.3
    ports:
    - containerPort: 80

watch kubectl describe pods nginx

docker rm -f ID # on node

# Step 3. Make k8s access from cli outside (SKIP FOR NOW, SHOW VIA AKS INSTEAD)

kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

https://192.168.79.10:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system


kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}') -o yaml

create ca.crt file

  kubectl config set-cluster cfc --server=https://192.168.79.10:6443 --certificate-authority=./ca.crt --embed-certs=true
  kubectl config set-context cfc --cluster=cfc
  kubectl config set-credentials admin-user --token={token}
  kubectl config set-context cfc --user=admin-user
  kubectl config use-context cfc

# Step 5. Deploy replica set

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
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
        image: nginx:1.15.3
        ports:
        - containerPort: 80

delete container on node, see that it will replicate make sure they are up.

shutdown VM, show that pod reschedule to other VM

# Step 6. Deploy Deployment

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
        image: nginx:1.15.3
        ports:
        - containerPort: 80


kubectl edit nginx-deployment


resources:
  requests:
    memory: "64Mi"
    cpu: "0.25"
  limits:
    memory: "128Mi"
    cpu: "0.5"


kubectl rollout status deployment/nginx-deployment

Insufficient cpu

limits:
  memory: "128Mi"
  cpu: "0.1"

kubectl get rs

kubectl rollout history deployment/nginx-deployment

kubectl rollout undo deployment/nginx-deployment --to-revision=2

kubectl get rs

# Step 6. Create services

kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80


exec inside container
docker exec -ti 8996463d639f /bin/bash
apt-get update && apt-get install -y nslookup
nslookup nginx-service


# Step 7. Expose through nodeport


apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
   - port: 80
     targetPort: 80
     nodePort: 30000

 curl localhost:30000

kubectl edit nginx-deployment

# Step 8. Ingress

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system

---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
        - name: admin
          containerPort: 8080
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
  type: NodePort

---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik.dashboard.local
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web


---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: traefik.nginx.local
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: http

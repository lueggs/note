
> 以下以amazon linux 2 操作 

1. 在amazon ec2 中建立3台instance(1 master, 2 worker) , 
規格: t2.medium以上, storage 30G 以上
這3台instance需要用同一個安全原則

2. 建立完成, 開始建置主機

- 每一台主機都需執行
```shell!=
# 以系統管理者登入, 方便後續command執行
sudo su

# 安裝docker
yum install -y docker

# 安裝netstat 
yum install -y nc

# check required ports
nc 127.0.0.1 6443 -v

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 建立k8s yum repo, 依照自己需要的版本做調整
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Install kubelet, kubeadm and kubectl
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# Enable the kubelet service before running kubeadm:

sudo systemctl enable --now kubelet
```


3. in master

```shell=
#  產生token

kubeadm init <args>

# 如果token 過期了或忘記了
kubeadm token create --print-join-command

# 如果是非root user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 如果是root user
export KUBECONFIG=/etc/kubernetes/admin.conf
```

4. in each worker
```shell=

# join to master node
kubeadm token create --print-join-command

kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

```


5. in master

```shell=
# after worker joined successfully, check if worker nodes exist in master

kubectl get nodes

# install calico

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml

```


6. ec2 need permission to access ECR

    - Create an IAM Role with ECR access:
        1. Go to the IAM Console.
        2. Create a new role for EC2 service.
        3. Attach the policy AmazonEC2ContainerRegistryReadOnly (or AmazonEC2ContainerRegistryFullAccess if needed).
    - Attach the Role to Your EC2 Instances:
        4. In the EC2 Console, select your EC2 instance.
        5. Click Actions > Security > Modify IAM Role.
        6. Attach the role you just created.


7. k8s 需要建立 secret 提供給ecr , 才能pull image to k8s
```shell!=
# create secret

kubectl create secret docker-registry <your-secret-name> --docker-server=<your-ecr-address>.dkr.ecr.ap-northeast-1.amazonaws.com --docker-username=AWS --docker-password=$(aws ecr --region ap-northeast-1 get-login-password) --namespace=<default or your-namespace>

# verify

kubectl get secrets <your-secret-name>

kubectl get secrets --all-namespaces
```

8. 在deployment.yaml, 需要設定imagePullSecret 為 7.建立的secret 

```yaml!=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy1
  labels:
    app: deploy1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deploy1
  template:
    metadata:
      labels:
        app: deploy1
    spec:
      imagePullSecrets:
        - name: ecr-secret

```

9. service.yaml, 在還沒有使用ingress或istio 等gateway之前。可以先用NodePort 設定想要開放出去的port

```yaml=!

apiVersion: v1
kind: Service
metadata:
  name: deploy1-svc
spec:
  selector:
    app: deploy1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30001
  type: NodePort
  
```
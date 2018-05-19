# k8s-vagrant

k8s cluster using kubeadm, flannel on vagrant

based on https://qiita.com/MahoTakara/items/28cd766d0447140b7ae3

- servers
  - k8s-1 = 172.42.42.11: master
  - k8s-2, k8s-3, k8s-4 = 172.42.42.12~14: node
- docs
  - https://kubernetes.io/docs/setup/independent/install-kubeadm/
  - https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

## master

ssh master by `vagrant ssh k8s-1`.

- edit `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`
  - set node-ip

```
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local --node-ip=172.42.42.11"
```

- init kubeadm
  - `--pod-network-cirdr=10.244.0.0/16`: used by flannel, please see `kube-flannel.yml` later.
  - `--apiserver-advertise-address=172.42.42.11`: set apiserver to master node ip

```
> kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.42.42.11 --service-cidr=10.244.0.0/16
# if you want to reset data, please try `kubeadm reset`
> mkdir -p $HOME/.kube
> sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
> sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- setup flannel
```
> sysctl net.bridge.bridge-nf-call-iptables=1
> curl -O https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
...
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:c3:22:23 brd ff:ff:ff:ff:ff:ff
    inet 172.42.42.11/24 brd 172.42.42.255 scope global enp0s8
> # edit kube-flannel.yml
# command: [ "/opt/bin/flanneld", "--ip-masq", "--kube-subnet-mgr", "--iface=enp0s8" ]
# this is explained in https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/ as `Default NIC When using flannel as the pod network in Vagrant`
```

## node

ssh master by `vagrant ssh k8s-2`. (and `k8s-3`, `k8s-4`)

- enter `kubeadm init` result command

```
> kubeadm join 172.42.42.11:6443 --token pow256.2bad41dhvpgnv2nu --discovery-token-ca-cert-hash sha256:b43ef1517d163751e10661d63c2a656668509579031f1176aeca01e38c96fcec
```

- Now you can see node on your cluster.

```
vagrant@k8s-1:~$ kubectl get nodes
NAME      STATUS     ROLES     AGE       VERSION
k8s-1     Ready      master    1h        v1.10.2
k8s-2     Ready      <none>    32m       v1.10.2
k8s-3     Ready      <none>    14m       v1.10.2
k8s-4     NotReady   <none>    10s       v1.10.2
```

- labeling by following command

```
> for n in k8s-2 k8s-3 k8s-4; do kubectl label node $n node-role.kubernetes.io/node=; done
```

- deploy nginx server
  - add `ngins-deploy.yml`
  - `kubectl create -f nginx-deploy.yml`

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 6
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx-container
        image: nginx:alpine
        ports:
        - containerPort: 80
```

```
> kubectl get po -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-deployment-69878cbf8d-gsbg5   1/1       Running   0          33s       10.244.1.2   k8s-2
nginx-deployment-69878cbf8d-mkdbw   1/1       Running   0          33s       10.244.3.2   k8s-4
```

- create ClusterIP Service
  - create `nginx-cluster-ip-service.yml`
  - `kubectl create -f nginx-cluster-ip-service.yml`

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    app: nginx-web
  ports:
  - protocol: TCP
    port: 80
```

```
> kubectl get svc -o wide -w
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE       SELECTOR
...
nginx-cluster-ip-service   ClusterIP   10.244.250.202   <none>        80/TCP    17s       app=nginx-web
> curl 10.244.250.202 # get nginx resposne
```

- create NodePort Service
  - create `nginx-node-port-service.yml`
  - `kubectl apply -f nginx-node-port-service.yml`
  - then you can do `curl 172.42.42.12:31514`

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-service
spec:
  type: NodePort
  selector:
    app: nginx-web
  ports:
    - protocol: TCP
      port: 80
      nodePort: 31514
```

---
title: kubernetes on proxmox (2) - worker node & master node 설정
date: 2022-10-03 20:20:15
tags:
categories: "home lab"
canonical_url: "https://minpeter.xyz/blog/k8s-via-proxmox-2"
---

## 1. k8s-ctrlr & k8s-node 생성

앞에서 만들어둔 template을 이용하여 k8s-ctrlr과 k8s-node를 생성한다.
부팅 후 cloud-init 설정이 진행될때까지 잠시 기다려준다.

## 2. static ip 설정

k8s-ctrlr과 k8s-node의 static ip를 설정한다.
어김없이 netplan를 이용한다.  
상황에 따라서 다르겠지만 보통의 공유기의 보통의 단일 컴퓨터로 만들어진 proxmox에서는 거희 비슷할 것이다.  
파일 수정 전 혹시 모르니 백업을 해둔다.

```bash
sudo cp /etc/netplan/01-netcfg.yaml{,.bak}
```

/etc/netplan/50-cloud-init.yaml 파일을 열어서 아래와 같이 수정한다.

```yaml
version: 2
ethernets:
  eth0:
    addresses: [192.168.0.180/24]
    routes:
      - to: default
        via: 192.168.0.1
    nameservers:
      addresses: [192.168.0.120, 1.1.1.1]
```

나의 경우 192.168.0.120에서 pi.hole로 로컬 dns 서버를 운용중이기 때문에 이를 nameserver로 추가해준다.
k8s-ctrlr는 `192.168.0.180`, k8s-node는 `192.168.0.185`로 설정했다.
이제 적용해준다.

```bash
sudo netplan apply
```

## 3. containerd 설치

k8s-ctrlr과 k8s-node에 containerd를 설치한다.

```bash
sudo apt-get update && sudo apt-get install -y containerd
```

### 3.1. containerd 설정

아래 명령으로 기존 설정을 저장한다.

```bash
sudo mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

그 다음 /etc/containerd/config.toml 파일을 열어서 아래와 같이 수정한다.

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = false
```

위의 설정에서 `SystemdCgroup = false`를 `true`로 바꿔준다.

## 4. 기타 네트워크 설정

/etc/systcl.conf 파일을 열어서 아래와 같이 수정한다.

```bash
# Uncomment the next line to enable packet forwarding for IPv4
# sysctl -w net.ipv4.ip_forward=1
```

위와 동일한 부분을 찾아 2번째 줄에서 `#`을 제거해준다.

sudo vim /etc/modules-load.d/k8s.conf 파일에 다음 내용을 추가해준다.

```bash
br_netfilter
```

변경된 네트워크 설정을 적용해주기 위해서 재부팅한다.

## 5. 본격 kubernetes 설치

```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

윗 줄부터 키를 받고, kubernetes apt 저장소를 추가하고, kubernetes 패키지를 설치한다.
[공식 메뉴얼](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl) 변경이 확인되면 수정하도록 하겠다.

## 6. k8s-node를 탬플릿으로

k8s-node를 탬플릿으로 만들어서 원하는 수 만큼 worker를 만들 수 있게 해보자.

일단 아래 명령어는 machine-id로 인해 복제된 vm의 ip가 같아지는 문제를 해결하기 위한 것이다.

```bash
sudo cloud-init clean

sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
```

그 다음 전원을 끄고 web ui에서 Cover to Template을 눌러준다.

탬플릿으로 변경됬다면 원하는 수 만큼 worker를 만들 수 있다.
여기서는 k8s-node-1, k8s-node-2 이렇게 2개를 생성해주자.

## 8. vm 사양 변경

k8s-ctrlr, k8s-node-1, k8s-node-2의 사양을 변경해준다.
전부 전원을 끄고, web ui에서 사양을 변경해준다.
proxmox을 구동하고 있는 서버 사양에 따라서 변경해주면 되는데, k8s-ctrlr의 경우 2cpu, 4gb, k8s-node-1, k8s-node-2의 경우 2cpu, 2gb이 이상적이라고 한다.
만약 서버 사양이 부족하다면 전부 2cpu에 2gb로 설정해주자.
이게 최소사양이다.  
그리고 사양이 좋다고 해도 램은 증가시키기보다 worker를 더 생성하는 편이 좋다고 하는데.. 난 몰라도 되지 않을까?
설정이 완료됬다면 다시 전부 전원을 켜주자.
또한 탬플릿에서 복제한 vm의 경우 static ip가 풀려서 dhcp가 됬을텐데 원한다면 static ip로 변경해주자.

## 9. cluster 생성

k8s-ctrlr에 접속해서 kubeadm을 이용해서 클러스터를 생성해준다.

```bash
sudo kubeadm init --control-plane-endpoint=<vm-ip> \
--node-name <vm-hostname> --pod-network-cidr=10.244.0.0/16
```

나의 경우 <vm-ip>는 192.168.0.180, <vm-hostname>은 k8s-ctrlr이다.

그 다음 아래 명령어를 실행해준다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

다음 아래에 생성되는 join 명령어를 복사해 다른 노드에서 실행해준다.

```bash
kubeadm join 192.168.0.180:6443 --token ~~~~~~
```

만약 못찾겠다면 k8s-ctrlr에서 아래 명령어를 실행해준다.

```bash
kubeadm token create --print-join-command
```

### 9.1. pod network 생성

k8s-ctrlr에서 아래 명령어를 실행해준다.

```bash
kubectl apply -f \
https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

이제 모든 노드가 정상적으로 생성되었는지 확인해보자.

```bash
kubectl get nodes
```

아래와 같이 모든 노드가 Ready 상태가 되었다면 성공이다.

```bash
NAME         STATUS   ROLES                  AGE   VERSION
k8s-ctrlr    Ready    control-plane,master   10m   v1.21.1
k8s-node-1   Ready    <none>                 10m   v1.21.1
k8s-node-2   Ready    <none>                 10m   v1.21.1
```

클러스터 생성은 여기까지이다.

## kubernetes 이용하기

클러스터를 힘들게 생성했는데 이제 뭘 해야할까?
이제 kubernetes를 이용해서 웹서버를 배포해보자.

### 1. nginx pod 배포

nginx pod 하나를 만들어 배포해보자.
k8s-ctrlr에서 pod.yaml 파일에 다음과 같이 작성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-example
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: linuxserver/nginx
      ports:
        - containerPort: 80
          name: "nginx-http"
```

그리고 아래 명령어를 배포해준다.

```bash
kubectl apply -f pod.yaml
```

이제 pod가 정상적으로 생성되었는지 확인해보자.

```bash
kubectl get pods -o wide
```

k8s-node-1에서 10.244.1.2:80의 파드 네트워크에서 nginx가 실행되고 있는 것을 확인할 수 있다.

```
NAME            READY   STATUS    RESTARTS   AGE    IP           NODE         NOMINATED NODE   READINESS GATES
nginx-example   1/1     Running   0          3h8m   10.244.1.2   k8s-node-1   <none>
```

k8s-ctrlr, k8s-node-1, k8s-node-2 모두에서 다음 명령어로 접속를 확인할 수 있다.

```bash
curl 10.244.1.2:80
```

### 2. nginx service 생성

하지만 이렇게 pod를 직접 접근하는 것은 좋지 않다.
pod는 재시작되거나 삭제되면 ip가 바뀌기 때문이다.
또한 cluster 내부 접근은 어느 노드든 가능하지만 외부에서는 불가능하다.
이런 문제를 해결하기 위해 service를 생성해준다.
k8s-ctrlr에서 service-nodeport.yaml 파일에 다음과 같이 작성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-example
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      nodePort: 30080
      targetPort: nginx-http
  selector:
    app: nginx
```

그리고 아래 명령어를 배포해준다.

```bash
kubectl apply -f service-nodeport.yaml
```

이제 service가 정상적으로 생성되었는지 확인해보자.

```bash
kubectl get svc
```

다음과 같은 출력이라면 정상적으로 생성된 것이다.

```
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        3h19m
nginx-example   NodePort    10.109.7.138   <none>        80:30080/TCP   3h16m
```

k8s-ctrlr, k8s-node-1, k8s-node-2 모두에서 다음 명령어로 접속을 확인할 수 있다.

```bash
curl localhost:30080
```

또한 외부에서 클러스터 구성원중 어떤 노드의 아이피로 접속해도 접속이 가능하다.

```bash
curl 192.168.0.180:30080
curl 192.168.0.185:30080
curl 192.168.0.190:30080
```

여기서 aws의 경우 nodeport type이 아닌 loadbalancer type을 사용해 한개의 주소로 접속할 수 있지만 로컬에서는 loadbalancer type을 사용할 수 없기 때문에 nodeport type을 사용했다.

## 마무리

이번 포스팅에서는 kubernetes를 설치하고 nginx를 배포하는 방법을 알아보았다.
다음 포스팅에서는 kubernetes의 다른 기능들을 알아보고자 한다.
ingress, configmap, secret, volume, deployment, statefulset 등을 알아보고자 한다.
또한 가능하다면 jenkins를 이용한 CI/CD를 구축해보고자 한다.

## 참고자료

- [youtube/How to Build an Awesome Kubernetes Cluster using Proxmox Virtual Environment](https://youtu.be/U1VzcjCB_sY)
- [blog/How to Build an Awesome Kubernetes Cluster using Proxmox Virtual Environment](https://www.learnlinux.tv/how-to-build-an-awesome-kubernetes-cluster-using-proxmox-virtual-environment/)

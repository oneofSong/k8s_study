# kubeadm을 이용한 kubernetes 클러스터 구축

## 1. 설치 환경
- 1 master
    - ubuntu 16.04
- 2 worker
    - ubuntu 20.04
    - RTX 3090

- kubeadm version : 1.23.6


## 2. kubernetes 클러스터 세팅

### 1. 패키지 업데이트
```console
# su -
# apt-get update -y && apt-get upgrade -y 
```

### 2. Docker 설치

- 종속 패키지 설치
```console
# sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```


- gpg 키 등록
- docker GPG 키 등록
    
```console
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

- nvidia docker 
``` text
$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
&& curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
&& curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```


- Docker 설치

```text
# sudo apt-get update
# sudo apt-get install -y docker-ce docker-ce-cli containerd.io

## GPU version

# sudo apt-get update
# sudo apt-get install -y nvidia-docker2

```


- kubelet의 cgroup driver 설정
    - daemon.json이 없을 시 아래 명령어 실행, 있을 경우 `vim`을 활용

```console
# cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# sudo systemctl enable docker
# sudo systemctl daemon-reload
# sudo systemctl restart docker
```

- docker 설치 확인
    - `docker version` 등의 명령어로 동작 확인
    
- 참고 : docker 명령어 사용자 추가
    - `sudo usermod -aG docker $USER `
    
- docker private registry 실행 
    - master node나 별도의 서버에 구축. 하나의 노드에만 있으면 됨
    - 설치할 서버에서 아래 명령어를 실행(docker가 설치되어 있는 서버)
        - `docker run -dit --name docker-registry -p 5000:5000 registry`
- docker에 insecure registry 추가
    - kubernetes node에 모두 추가  
    - `vi /etc/docker/daemon.json`
        - `"insecure-registries" : ["<registry IP>:5000"]` 를 json에 추가
    - docker restart : `sudo service docker restart`


### 3. kubeadm, kubelet, kubectl 설치

- 패키지 update 및 종속 패키지 설치
```console
# sudo apt-get update
# sudo apt-get install -y apt-transport-https ca-certificates curl
```

- gpg 키 추가
```console 
# sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
# echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- 패키지 설치
```console
# sudo apt-get update
# sudo apt-get install -y kubelet=1.23.6-00 kubectl=1.23.6-00 kubeadm=1.23.6-00
```

- version 확인

``` text
# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.6", GitCommit:"ad3338546da947756e8a88aa6822e9c11e7eac22", GitTreeState:"clean", BuildDate:"2022-04-14T08:48:05Z", GoVersion:"go1.17.9", Compiler:"gc", Platform:"linux/amd64"}

# kubectl version
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.6", GitCommit:"ad3338546da947756e8a88aa6822e9c11e7eac22", GitTreeState:"clean", BuildDate:"2022-04-14T08:49:13Z", GoVersion:"go1.17.9", Compiler:"gc", Platform:"linux/amd64"}

# kubelet --version
Kubernetes v1.23.6

```

- 패키지 업데이트 보류
```console
# sudo apt-mark hold kubeadm kubelet kubectl
```
- swap off 하기
    - off 명령어 : swapoff -a
    - swap 상태 확인 : swapon -s
        - 출력 내용이 없으면 off 상태
        
### 4. kubernetes 클러스터 생성

#### 4-1. 마스터 노드


- kubeamd init
    - 명령어
        - ` # sudo kubeadm init --apiserver-advertise-address=<master node IP> --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=kube.example.org `
        - pod-network-cidr : 이후 추가할 에드온인 Flannel을 위해 추가. 반드시 ip는 `10.244.0.0/16`으로 설정
        - apiserver-cert-extra-sans : 이후 원격에서 kubectl을 사용하기 위해서 필요
        - apiserver-advertise-address : 마스터 노드의 API Server 주소를 설정할 때 사용하는 옵션, master 다중 노드 설정시에는 control-plane-endpoint를 사용하여 loadbalancer를 사용

    - 실행 결과

```text
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <master node IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<habsh value>
```
    
- kubectl 설정
```console
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config ## kubectl을 통해 kubernetes 클러스터 접근하기 위해 필요
# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


#### 4-2워커 노드

- 각 워커 노드에서 `kubeadm join`을 실행. `kube init` 실행 결과에서 나온 명령어를 실행

```
kubeadm join <Master Node IP>:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<habsh value>
```

- kubeadm init 시에 생성된 token은 1일 뒤에 삭제됨. 이후 추가를 위해서는 token을 새롭게 생성 하여 join을 실행

- token 생성 방법
    - `kubeadm token create`
- token 조회 방법
    - `kubeadm token list`    
- hash 조회 방법
    - `openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'`
    


## 3. kubernetes 추가 설정

### 3-1. flanel 세팅

- coredns가 동작하기 위해서는 network plugin이 필요
    - ```console
        # kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    ```
    - 결과 확인 `kubectl get pods -n kube-system`
    - ```console
    NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
    kube-system   coredns-5644d7b6d9-c5mn2         1/1     Running   0          8m8s
    kube-system   coredns-5644d7b6d9-s2rwt         1/1     Running   0          8m8s
    kube-system   etcd-ubuntu                      1/1     Running   0          7m1s
    kube-system   kube-apiserver-ubuntu            1/1     Running   0          7m1s
    kube-system   kube-controller-manager-ubuntu   1/1     Running   0          7m4s
    kube-system   kube-flannel-ds-amd64-hwdfq      1/1     Running   0          93s
    kube-system   kube-proxy-fsgmj                 1/1     Running   0          8m8s
    kube-system   kube-scheduler-ubuntu            1/1     Running   0          7m25s

### 3-2. nfs provisioner 세팅

- namespace 생성 및 설정
    - `kubectl create namespace nfs-provisioner`
    - `kubectl set-config`
- helm repo 추가
    - `helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/`
- helm isntall
    - nfs.server : nfs서버의 IP 주소
    - nfs.path : nfs 서버에서 세팅된 디렉토리 경로. 하위에 provision된 volume의 디렉토리가 생성됨
     
``` console
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=x.x.x.x \
--set nfs.path=/exported/path\
-n nfs-provisioner
```

## 참고. TroubleShooting
### 1. kubeadm init 에러 
- 발생 로그
``` text
[kubelet-check] it seems like the kubelet isn't running or healthy
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.
```

- 해결법 
    - kubelet log 확인
        - ` # journalctl -xeu kubelet`
    - /etc/docker/daemon.json 작성이 되어 있는지 확인, 없을 시 아래 명령어 실행
        - ```console
            # cat <<EOF | sudo tee /etc/docker/daemon.json

            {

              "exec-opts": ["native.cgroupdriver=systemd"],

              "log-driver": "json-file",

              "log-opts": {

                "max-size": "100m"

              },

              "storage-driver": "overlay2"

            }

            EOF
            # systemctl daemon-reload 
            # systemctl restart kubelet
            # 
            ```
    - 생성 후 kubeadm reset 및 init, 작동이 안될 시 리부팅 후 다시 실행
        - init 시 추가 명령어는 위에서 실행했던 명령과 동일하게 실행
        - ` # kubeadm reset `
        - ` # kubeadm init  [init 시 추가한 옵션, pod-network-cidr 등] `
        
    ```
    
### 2. Coredns CrashLoopBackOff 에러
- 발생 에러
```console 
NAMESPACE     NAME                             READY   STATUS                 RESTARTS   AGE
kube-system   coredns-5644d7b6d9-c5mn2         0/1     CrashLoopBackOff       1          1m8s
kube-system   coredns-5644d7b6d9-s2rwt         0/1     CrashLoopBackOff       1          1m8s
```

- 원인 파악
    - coredns pod 로그 확인
    ```console
    # kubectl get pod -n kube-system
    # kbuectl logs [coredns pod 이름 중 하나] -n kube-system
    ```
    - 
    ```console
    plugin/loop: Loop (127.0.0.1:55953 -> :1053) detected for zone ".", see 
    https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 4547991504243258144.3688648895315093531."
    ```
    - 일반적으로는 CoreDNS의 Query 요청을 127.0.0.1, :: 1 또는 127.0.0.53과 같은 루프백 주소를 통해 자신에게 직접 전달하는 경우에 발생
    - 덜 일반적으로는 CoreDNS는 업스트림 서버로 전달하고 다시 CoreDNS로 요청을 전달합니다.

- 해결 방법
    - DNS ip 확인 `nmcli device show | grep IP4.DNS`
    - `vi /etc/resolvconf/resolv.conf.d/head`에 내용 추가 `nameserver <DNS IP>`
    - 수정된 resolve.conf 적용 
    
      `# sudo resolvconf -u`
    - 수정된 /etc/resolve.conf 확인 
    
       `# cat resolv.conf`
    - coredns roleback
        ```console
            # kubectl rollout restart -n kube-system deployment/coredns
        ```
    

### 4. unknown service runtime.v1alpha2.RuntimeService 에러
- `kubeamd init` 실행 시 발생
 ``` text
 listing containers: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService
 ```
 - 원인 
     - kubernetes에서 실행될 컨테이너 런타임 인터페이스(CRI)가 정확하게 지정되지 않아서 발생하는 에러
 
 - 해결 방법 
     - CRI를 재시작(docker or containerd) 및 설정 삭제
     - docker 및 containerd 서비스 종료
         - `# service docker stop`
         - `# service containerd stop`
     - /etc/containerd/config.toml 삭제
         - `rm /etc/containerd/config.toml`
     - docker 재시작
     
   

## 참고

- [kubernetes 클러스터 구축](https://dongle94.github.io/kubernetes/kubernetes-cluster-build/)
- [kubernetes에서 NVIDIA GPU container 사용하기](https://kangwoo.github.io/devops/kubernetes/nvidia-docker2/)
- [kubeadm으로 단일 노드 Kubernetes 클러스터 만들기](https://essem-dev.medium.com/kubeadm%EC%9C%BC%EB%A1%9C-%EB%8B%A8%EC%9D%BC-%EB%85%B8%EB%93%9C-kubernetes-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0-%EB%A7%8C%EB%93%A4%EA%B8%B0-b3428ac6dbda)
- [kube init 에러](https://www.inflearn.com/questions/289117)
- [Troubleshooting: 갑자기 flannel pod가 crash나는 경우](https://happycloud-lee.tistory.com/171)
- [flannel pods in CrashLoopBackoff Error in kubernetes](https://stackoverflow.com/questions/49510788/flannel-pods-in-crashloopbackoff-error-in-kubernetes)
- [Kubernetes - coredns CrashLoopBackOff & Restarting 현상 조치방안](https://waspro.tistory.com/564)
- [coredns crashloopbackoff in kubernetes](https://stackoverflow.com/questions/54466359/coredns-crashloopbackoff-in-kubernetes)
- [coredns troublshooting](https://coredns.io/plugins/loop/#troubleshooting)
- [How do I force Kubernetes CoreDNS to reload its Config Map after a change?
](https://stackoverflow.com/questions/53498438/how-do-i-force-kubernetes-coredns-to-reload-its-config-map-after-a-change)
- [unknown service runtime.v1alpha2.ImageService](https://github.com/kubernetes-sigs/cri-tools/issues/710)

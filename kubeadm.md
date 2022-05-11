# kubeadm을 이용한 kubernetes 단일 노드 생성

## 1. 설치 환경
- vmware 16 player
- OS : Ubuntu 16.04.7 LTS
- CPU : 2core
- RAM : 4GB

## 2. kubernetes 마스터 노드 생성

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

```console
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```


- Docker 설치
```console
# sudo apt-get update
# sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```


- kubelet의 cgroup driver 설정

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
    - ```#docker version``` 등의 명령어로 동작 확인
    
- 참고 : docker 명령어 사용자 추가
    - ` # sudo usermod -aG docker $USER `

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
# sudo apt-get install -y kubelet kubeadm kubectl
```

- 패키지 업데이트 보류
```console
# sudo apt-mark hold kubeadm kubelet kubectl
```

### 4. kubernetes 클러스터 생성(마스터 노드)

- swap off 하기
    - off 명령어 : swapoff -a
    - swap 상태 확인 : swapon -s
        - 출력 내용이 없으면 off 상태
- kubeamd init
    - 명령어
        - ` # sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=kube.example.org `
        - pod-network-cidr : 이후 추가할 에드온인 Flannel을 위해 추가. 반드시 ip는 `10.244.0.0/16`으로 설정
        - apiserver-cert-extra-sans : 이후 원격에서 kubectl을 사용하기 위해서 필요

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
    kubeadm join 172.31.26.245:6443 --token cml8w5.aem9su86pztp735y \
     --discovery-token-ca-cert-hash sha256:15bd2983695a078b8c664e2aee482692da171881789682871f8bcd898c83e2d4

    ```
- Flannel Network 애드온 추가
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
    ```
    
- kubectl 설정
```console
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config ## kubectl을 통해 kubernetes 클러스터 접근하기 위해 필요
# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- master node에 일반 pod를 띄울 수 있도록 untaint 설정 변경
```console
# kubectl taint nodes --all node-role.kubernetes.io/master-
```
    - 참고 : master node에 다시 taint 설정하기
        - ` # kubectl taint nodes master node-role.kubernetes.io=master:NoSchedule `


## 3. TroubleShooting
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
        
### 2. flannel CrashLoopBackOff 에러
 - 발생 에러
     - ![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FMXorf%2FbtqSIYa9DdJ%2FvMr5zz8dnx29KKXgzsCA91%2Fimg.png)
- 로그 확인 방법
    - pod의 내부 로그 확인
        - pod 이름 확인 :  ``` kubectl get pod -n kube-system```
        - pod의 로그 확인 ``` kubectl logs [확인한 flannel pod 이름] -n kube-system
- 해결 방법
    - `kubeadm init` 시에 `--pod-network-cidr=10.244.0.0/16`을 옵션으로 사용했는지 확인
    - 안했다면 아래 명령어 실행
    ```console
    # kubeadm reset
    # kubeadm init --pod-network-cidr=10.244.0.0/16
    ```
    
### 3. Coredns CrashLoopBackOff 에러
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
     
     
 ### 5. taint "node-role.kubernetes.io/master:" not found
 - `kubectl taint nodes --all node-role.kubernetes.io/master-` 명령어 실행 시 발생
 
 - 원인 : master 노드가 존재 하지 않아 taint 명령어가 실행되지 않음
 
 - 해결 방법 
     - `kubectl get node`를 통해 node의 이름을 확인
       ```txt
       NAME     STATUS   ROLES           AGE   VERSION
       aiuser   Ready    control-plane   17h   v1.24.0
       ```
     
     - `kubectl describe node <nodename> | grep Taints` 를 통해 taint 상태를 확인
        ``` txt
        Taints:             node-role.kubernetes.io/control-plane:NoSchedule
        ```
     - 다음과 같이 taint에 있는 :Noschedule 대신 -를 명령어에 포함하여 taint 명령어 실행. `kubectl taint nodes --all node-role.kubernetes.io/control-plane-` 


## 참고

- [쿠버네티스 싱글 노드 설치](https://www.kudryavka.me/?p=885)
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
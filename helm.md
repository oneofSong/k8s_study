# helm 설치

## script 설치

- 스크립트 실행

```console
# curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
# chom 700 get_helm.sh
# ./get_helm.sh
```

- 설치 확인

``` console
# helm version
```

## helm repo 추가

- stabel repo 추가

```consle 
# helm repo add stable https //charts.helm.sh/stable
# helm repo add bitnami https//charts.bitnami.com/bitnami
```

- 추가된 repogitory 확인

```console
# helm repo update
# helm search repo stable
```



## helm을 이용한 테스트 패키지 설치

- nginx 서버를 helm을 이용하여 설치하여 정상작동을 하는지 확인

### nginx chart 확인하기

- helm chart 다운로드
```console
# helm pull bitnami/nginx
```

- tar 압축 풀기
```console
# tar -xvzf nginx-9.8.0.tgz
```

- chart의 values.yaml 수정
```console
# vi values.yaml
```
    - service.type의 기본값인 LoadBalancer를 NodePort로 변경

```yml
service
    type: NodePort
```

- 패키지 설치
```console 
# helm install test-nginx nginx
```

- 설치 확인
    - service 정보 확인시에 PORT에 있는 포트를 사용하여 nginx 서버에 접속
```console
# kubectl get all
# kubectl get service test-nginx            ## 서비스의 외부 포트 확인
NAME         TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
test-nginx   NodePort   10.100.243.135   <none>        80:30338    1m23s
```

- 브라우저에서 nginx가 정상작동하는지 확인
     - localhost:30338
     
     
- 패키지 삭제 
```console
# helm delete test-nginx
# kubectl get all     ## 삭제되었는지 확인
```
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
```

- 추가된 repogitory 확인

```console
# helm repo update
# helm search repo stable
```
#  < Harbor >



# 1. 개요

Harbor는 CNCF를 졸업한 프로젝트로, 대표적인 사설 레지스트리 (Private Registry) 오픈소스이다.



## Private Registry

보통 hub.docker.com에서 제공하거나 오픈소스 프로젝트에서 제공하는 컨테이너 이미지는 인터넷이 되는 모든 곳에서 풀을 받아 사용이 가능하다. 이러한 컨테이너 이미지를 제공해주는 레지스트리를 Public Registry라고 한다.

 

하지만, 회사와 같이 프로젝트가 오픈되지 않아야 하는 환경에서는 회사 환경 내부에서만 접근이 가능해야 하고, 큰 회사의 경우에는 특정 부서에서만 특정 이미지를 푸쉬하거나 풀 할 수 있도록 권한을 제어해야 한다. 이렇게 특정 환경에서만 접근이 가능해야 하는 레지스트리를 Private Registry라고 한다.



Harbor는 대표적인 Private registry 오픈소스이며, 컨테이너 이미지 저장 외에도 여러가지의 기능을 제공하고 있다.

 

- 컨테이너 이미지에 대한 보안 및 취약점 분석

- RBAC (Role based access control)

- 정책 기반 복제 (Replication)

- 여러 방식의 인증기능 제공 (LDAP / AD / OIDC)

- 이미지 삭제 및 가비지 컬렉션을 통해 주기적으로 리소스 확보

- Web 기반의 GUI

- 감사(Audit)기능 제공

- RESTful API 제공

- docker-compose, helm 등 여러가지 배포 방식 제공



```sh
$ helm repo add harbor https://helm.goharbor.io
$ helm repo update

$ helm fetch harbor/harbor

$ tar -xzvf harbor-1.14.1.tgz


$ helm -n harbor install harbor . \
    --set expose.type=ingress \
    --set expose.tls.enabled=true \
    --set expose.ingress.className=nginx \
    --set persistence.enabled=false \
    --set harborAdminPassword=admin \
    --set expose.ingress.hosts.core=harbor-cjs.duckdns.org \
    --set externalURL=https://harbor-cjs.duckdns.org \
    --set nginx.tls.enabled=true 
    
    
# 삭제시...
$ helm -n harbor delete harbor
```



## Harbor 설정

### Project 설정

![image-20240403195858095](/Users/jongsoo/cjs/git/edu/edu_cicd/harbor/image-20240403195858095.png)



#### Project 명 입력, public 설정

![image-20240403200505962](/Users/jongsoo/cjs/git/edu/edu_cicd/harbor/image-20240403200505962.png)





## Image Push&Pull&Remove Test

```sh
$ docker login https://harbor-cjs.duckdns.org

#로그인 실패시
$ sudo vi /etc/docker/daemon.json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "features": {
    "buildkit": true
  },
  "insecure-registries": [                             # <-- 추가
    "harbor.cjs.com"
  ]
}


$ docker tag spring-test:1.0.0 harbor-cjs.duckdns.org/edu/spring-test:1.0.0
$ docker push harbor-cjs.duckdns.org/edu/spring-test:1.0.0
$ docker rmi harbor-cjs.duckdns.org/edu/spring-test:1.0.0
$ docker pull harbor-cjs.duckdns.org/edu/spring-test:1.0.0


$ docker tag spring-test:1.0.0 public.ecr.aws/b3v0x0o0/cjs-edu:1.0.0
$ docker push public.ecr.aws/b3v0x0o0/cjs-edu:1.0.0

$ docker images



# 확인
$ docker images

```

ImagePullSecrets 생성

```sh
$ kubectl create secret docker-registry registry-secret --docker-server=harbor-cjs.duckdns.org --docker-username=admin --docker-password=admin --docker-email=admin@admin -n jenkins-admin
```



test.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test
  name: test
  namespace: jenkins-admin
spec:
  containers:
  - image: public.ecr.aws/b3v0x0o0/cjs-edu:0.0.2
    name: test
    
    
    
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test2
  name: test2
  namespace: jenkins-admin
spec:
  containers:
  - image: cjs0533/spring-test:1.0.0
    name: test2
```





```
Failed to pull image "harbor-cjs.duckdns.org/edu/spring-test": failed to pull and unpack image "harbor-cjs.duckdns.org/edu/spring-test:latest": failed to resolve reference "harbor-cjs.duckdns.org/edu/spring-test:latest": failed to do request: Head "https://harbor-cjs.duckdns.org/v2/edu/spring-test/manifests/latest": tls: failed to verify certificate: x509: certificate signed by unknown authority

```





```sh
cat <<'EOF' > kindconfig.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
# 3 control plane node and 1 workers
nodes:
- role: control-plane
# - role: worker
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor-cjs.duckdns.org"]
    endpoint = ["https://kind:5000"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor-cjs.duckdns.org".tls]
    insecure_skip_verify = true
EOF


$ brew install kind
$ kind create cluster -v1 --config kindconfig.yaml
```





```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor-cjs.duckdns.org"]
    endpoint = ["http://10.52.212.167:9000"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor-cjs.duckdns.org".tls]
    insecure_skip_verify = true 
```


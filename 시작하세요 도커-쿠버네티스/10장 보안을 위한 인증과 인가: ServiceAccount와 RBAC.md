**도입부**

- 쿠버네티스는 여러 개발자와 애플리케이션이 동시에 사용하므로 보안 고려가 필요합니다. 가장 자주 쓰는 기능은 RBAC(Role Based Access Control) 기반의 서비스 어카운트(Service Account)입니다.
- 서비스 어카운트는 사용자 또는 애플리케이션 하나에 대응하고, RBAC로 특정 명령을 실행할 수 있는 권한을 부여받습니다. 부여받은 권한에 해당하는 기능만 사용할 수 있습니다.
- 리눅스의 root와 일반 유저(+ sudoers) 구조와 비슷합니다. 지금까지 쓴 kubectl은 root처럼 최상위 권한이었습니다. 운영 환경이나 다중 사용자 환경에서는 필요한 최소 권한만 주는 것이 바람직합니다.

## 10.1 쿠버네티스의 권한 인증 과정

- 쿠버네티스는 kube-apiserver, kube-controller, kube-scheduler, etcd 등으로 구성되며 가장 자주 접하는 것은 kube-apiserver입니다. 컴포넌트들은 kube-system 네임스페이스에서 실행됩니다.
- `kubectl apply -f` 같은 명령의 내부 처리 순서는 다음과 같습니다.
    1. HTTP 핸들러가 요청을 받음
    2. Authentication(인증): 쿠버네티스 사용자가 맞는지 확인
    3. Authorization(인가): 해당 기능을 실행할 권한이 있는지 확인
    4. Mutating / Validating Admission Controller를 거침
    5. etcd에 반영되어 기능 수행
- 인증·인가 방법에는 서비스 어카운트 외에 서드파티 인증(OIDC, OAuth), 인증서 등이 있습니다.
- 지금까지 계정 생성 없이 kubectl을 쓸 수 있었던 이유는 설치 도구가 `~/.kube/config`에 관리자 권한을 자동 설정했기 때문입니다.
    - `users` 항목의 `client-certificate-data`, `client-key-data`는 base64 인코딩된 인증서 키 쌍이며, 최고 권한(cluster-admin)을 가집니다.
    - users에는 인증 정보, clusters에는 클러스터 접근 정보가 저장된다는 정도만 알면 됩니다.
- 인증서 키 쌍 방식은 절차가 복잡하고 관리가 어려워 자주 쓰지는 않습니다. 이 장에서는 주로 서비스 어카운트를 다룹니다.

## 10.2 서비스 어카운트와 롤(Role), 클러스터 롤(Cluster Role)

**서비스 어카운트 기본**

- 권한 관리를 위한 쿠버네티스 오브젝트로, 한 명의 사용자나 애플리케이션에 해당합니다. 네임스페이스에 속하며 `sa`로 줄여 쓸 수 있습니다.
- 각 네임스페이스에는 기본적으로 `default` 서비스 어카운트가 존재합니다.

```bash
kubectl get sa
kubectl create sa alicek106
```

- `-as` 옵션으로 특정 서비스 어카운트를 임시로 사용할 수 있습니다(공식 문서 용어로 impersonate).

```bash
kubectl get services --as system:serviceaccount:default:alicek106
# Error ... Forbidden: cannot list resource "services"
```

- `system:serviceaccount`는 서비스 어카운트로 인증한다는 뜻이고, `default:alicek106`은 default 네임스페이스의 alicek106을 의미합니다. 아직 권한이 없어 에러가 납니다.

**롤과 클러스터 롤 개념**

- 권한 부여 방법은 롤(Role)과 클러스터 롤(ClusterRole) 두 가지입니다. 둘 다 부여할 권한이 무엇인지 나타내는 오브젝트입니다.
- 롤은 네임스페이스에 속하며 디플로이먼트, 서비스처럼 네임스페이스에 속하는 오브젝트의 권한을 정의합니다.
- 클러스터 롤은 클러스터 단위 권한을 정의합니다. 퍼시스턴트 볼륨처럼 네임스페이스에 속하지 않는 오브젝트, 클러스터 전반에 걸친 기능, 여러 네임스페이스에서 반복 사용되는 권한 등에 씁니다.
- `kubectl get role`은 현재 네임스페이스의 롤만, `kubectl get clusterrole`은 클러스터 전체의 클러스터 롤을 보여줍니다.
- 클러스터 롤은 컴포넌트용 권한도 포함해 미리 많이 생성되어 있습니다(admin, cluster-admin, edit, nginx-ingress-clusterrole 등). 기본 kubeconfig 인증서에는 cluster-admin이 부여되어 있습니다.

**롤 작성 (예제 10.1 service-reader-role.yaml)**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: service-reader
rules:
- apiGroups: [""]           # 대상 오브젝트의 API 그룹
  resources: ["services"]   # 대상 오브젝트 이름
  verbs: ["get", "list"]    # 허용할 동작
```

- apiGroups: 오브젝트가 속한 API 그룹입니다. `""`는 파드, 서비스 등이 포함된 코어 API 그룹이며, 디플로이먼트나 레플리카셋은 `apps` 그룹입니다. `kubectl api-resources`로 확인할 수 있습니다.
- resources: 권한을 정의할 오브젝트 이름이며 api-resources에 출력되는 이름을 사용합니다.
- verbs: 수행 가능한 동작입니다. 위 예시는 개별 서비스 조회와 목록 조회를 허용합니다.
- YAML의 `["a", "b"]` 표현은 `a` 형태의 리스트와 같습니다.
- 정리하면 코어 API 그룹의 서비스 리소스에 대해 get, list를 할 수 있다는 의미입니다.

**롤 바인딩 (예제 10.2 rolebinding-service-reader.yaml)**

- 롤 생성만으로는 권한이 부여되지 않습니다. 롤 바인딩(RoleBinding)으로 대상과 롤을 연결해야 합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: service-reader-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount     # 권한을 부여할 대상
  name: alicek106
  namespace: default
roleRef:
  kind: Role               # Role에 정의된 권한을 부여
  name: service-reader
  apiGroup: rbac.authorization.k8s.io
```

- 적용 후 `get services`는 성공하지만 `get deployment`는 여전히 Forbidden입니다.
- 롤, 롤 바인딩, 서비스 어카운트는 1:1 관계가 아닙니다. 하나의 롤을 여러 롤 바인딩이 참조할 수 있고, 하나의 서비스 어카운트가 여러 롤 바인딩으로 권한을 받을 수 있습니다. 롤은 템플릿, 롤 바인딩은 중간 다리 역할입니다.
- verbs에는 get, list, watch, create, update, patch, delete 등과 와일드카드 를 쓸 수 있습니다. `kubectl exec`처럼 특정 기능에는 서브 리소스(예: `pods/exec`에 create)를 명시해야 할 수 있습니다.

### 롤 vs. 클러스터 롤

- 롤과 롤 바인딩은 네임스페이스에 한정되므로 노드, 퍼시스턴트 볼륨 같은 클러스터 수준 오브젝트나 `-all-namespaces` 조회는 Forbidden(at the cluster scope)이 납니다.
- 이때 클러스터 롤을 사용합니다. kind가 ClusterRole이라는 점 외에는 롤과 거의 같습니다(예제 10.3 nodes-reader-clusterrole.yaml, resources: nodes, verbs: get, list).
- 클러스터 롤은 클러스터 롤 바인딩(ClusterRoleBinding)으로 대상과 연결합니다(예제 10.4). roleRef의 kind가 ClusterRole입니다.
- 적용 후 `kubectl get nodes --as ...alicek106`이 정상 동작합니다.
- 네임스페이스에 속하는 오브젝트(서비스 등)를 클러스터 롤로 정의하면 모든 네임스페이스의 해당 리소스 권한이 부여됩니다.

### 여러 개의 클러스터 롤을 조합해서 사용하기

- 클러스터 롤 애그리게이션(aggregation): 자주 쓰는 클러스터 롤을 다른 클러스터 롤에 포함시켜 재사용하는 기능입니다(예제 10.5 clusterrole-aggregation.yaml).
    - parent-clusterrole에 `rbac.authorization.k8s.io/aggregate-to-child-clusterrole: "true"` 라벨과 nodes get/list 권한을 정의합니다.
    - child-clusterrole은 `rules: []`로 비워두고 `aggregationRule.clusterRoleSelectors`의 matchLabels로 위 라벨을 선택합니다.
    - 결과적으로 child-clusterrole이 parent의 권한을 그대로 물려받습니다.
- 여러 클러스터 롤 권한을 하나로 합치거나 다단계 상속 구조를 만들 수 있습니다.
- 기본 클러스터 롤도 이 구조입니다. view → edit → admin 순으로 권한이 전파되므로 view에 권한을 추가하면 admin에도 적용됩니다.

## 10.3 쿠버네티스 API 서버에 접근

### 10.3.1 서비스 어카운트의 시크릿을 이용해 쿠버네티스 API 서버에 접근

- 애플리케이션은 보통 kubectl이 아닌 REST API로 API 서버에 접근합니다. 엔드포인트는 별도 설정 없이 자동 개방되어 있습니다.
- kubeadm은 마스터 IP:6443, GKE나 kops는 443 포트를 씁니다. 원격 접근 시 `~/.kube/config`의 server 주소를 사용합니다. 기본적으로 HTTPS만 처리하며 자체 서명 인증서를 씁니다.
- 인증 정보 없이 `curl https://localhost:6443 -k`를 보내면 `system:anonymous` 사용자로 간주되어 403이 반환됩니다. 익명 유저를 비활성화했다면 401(Unauthorized)이 반환되며, `-anonymous-auth` 옵션으로 설정합니다.
- 서비스 어카운트임을 증명하려면 JWT 토큰이 필요합니다. 가장 간단한 발급 방법은 다음과 같습니다.

```bash
export TOKEN=$(kubectl create token alicek106)
curl https://localhost:6443/apis --header "Authorization: Bearer $TOKEN" -k
```

- 이 API는 TokenRequest API이며 클러스터 롤로 호출 권한(`serviceaccounts/token`에 create 등)을 정의할 수 있습니다.
- `kubectl create token` 토큰의 유효기간은 기본 1시간이며 `-duration`으로 조정합니다.
- 만료 없는 토큰이 필요하면 `kubernetes.io/service-account.name` 어노테이션과 `type: kubernetes.io/service-account-token`으로 시크릿을 만듭니다(예제 10.6 sa-secret-token.yaml). 다만 유출 시 위험하므로 가급적 쓰지 않는 것이 좋습니다.
- `kubectl proxy`로 인증 없이 로컬에서 접근할 수 있지만 테스트 용도로만 권장됩니다.
- kubectl 기능은 REST API로도 동일하게 쓸 수 있습니다. 예를 들어 `/api/v1/namespaces/default/services`는 `kubectl get services -n default`와 같습니다. 단, 롤이나 클러스터 롤로 권한을 부여하지 않으면 접근할 수 없습니다.
- `/logs`, `/metrics` 같은 비리소스 경로는 클러스터 롤의 `nonResourceURLs`로 권한을 부여합니다.

### 10.3.2 클러스터 내부에서 kubernetes 서비스를 통해 API 서버에 접근

- 클러스터 내부 애플리케이션도 API 서버 접근이 필요합니다(예: Nginx 인그레스 컨트롤러가 Watch API로 인그레스 변경을 감지).
- 이를 위해 default 네임스페이스에 `kubernetes`라는 서비스가 미리 생성되어 있으며, 파드는 `kubernetes.default.svc`라는 DNS 이름으로 API 서버에 접근할 수 있습니다.
- 이 서비스에 접근한다고 특별한 권한이 생기지는 않습니다. 토큰을 담아 보내야 인증과 인가가 진행됩니다.
- 쿠버네티스는 파드를 생성할 때 서비스 어카운트의 시크릿을 자동으로 파드 내부에 마운트합니다.
    - 아무 설정이 없으면 default 서비스 어카운트가 쓰입니다.
    - 마운트 경로는 `/var/run/secrets/kubernetes.io/serviceaccount`이고, `ca.crt`, `namespace`, `token` 파일이 들어 있습니다.
- 파드 스펙에 `serviceAccountName`을 지정하면 특정 서비스 어카운트의 시크릿을 마운트합니다(예제 10.7 sa-deploy-nginx.yaml).
- 파드 생성 권한은 신뢰할 수 있는 사용자에게만 주는 것이 좋습니다. serviceAccountName으로 다른 서비스 어카운트의 토큰을 쓸 수 있기 때문입니다. `automountServiceAccountToken: false`로 자동 마운트를 막을 수 있습니다.

### 10.3.3 쿠버네티스 SDK를 이용해 파드 내부에서 API 서버에 접근

- 파드 내부 애플리케이션은 보통 언어별 쿠버네티스 SDK를 사용합니다. 흐름은 다음과 같습니다.
    1. 서비스 어카운트를 생성하고 롤과 롤 바인딩으로 권한을 부여합니다(alicek106에 서비스 읽기 권한).
    2. 파드 YAML에 `serviceAccountName: alicek106`을 명시해 생성합니다(예제 10.8 sa-pod-python-sdk.yaml, 이미지 `alicek106/k8s-sdk-python:latest`).
    3. 파드 내부에 마운트된 시크릿을 확인합니다.
    4. 파이썬 코드를 작성해 실행합니다(예제 10.9 list-service-and-pod.py).
- 코드 핵심은 다음과 같습니다.
    - `config.load_incluster_config()`: 마운트된 토큰과 ca.crt를 읽어 인증·인가를 수행합니다.
    - `client.CoreV1Api().list_namespaced_service(...)`: 권한이 있으므로 성공합니다.
    - `client.CoreV1Api().list_namespaced_pod(...)`: 파드 권한이 없어 403 Forbidden이 납니다.
- 디플로이먼트는 apps 그룹이므로 `client.AppsV1Api.list_namespaced_deployment(...)`를 사용합니다.
- 쿠버네티스 API 서버는 일종의 OIDC 서버 역할도 하며 `/.well-known/openid-configuration`, `/openid/v1/jwks` 엔드포인트를 노출합니다. 다만 서비스 어카운트 토큰 검증에 필요한 최소 기능만 제공합니다.

## 10.4 서비스 어카운트에 이미지 레지스트리 접근을 위한 시크릿 설정

- 7장의 docker-registry 타입 시크릿은 파드 스펙의 `imagePullSecrets`에 명시해야 했습니다.
- 서비스 어카운트에 이 시크릿을 설정하면 디플로이먼트나 파드 YAML마다 지정하지 않아도 됩니다(예제 10.10 sa-reg-auth.yaml).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reg-auth-alicek106
  namespace: default
imagePullSecrets:
- name: registry-auth
```

- 파드의 serviceAccountName에 이 서비스 어카운트를 지정하면 imagePullSecrets가 자동으로 추가됩니다.
- default 서비스 어카운트에 imagePullSecrets를 추가하면 아무 설정이 없어도 사설 레지스트리 인증을 기본 수행하게 할 수 있습니다.

## 10.5 kubeconfig 파일에 서비스 어카운트 인증 정보 설정

- 기본 kubeconfig에는 관리자 권한 인증서가 들어 있습니다. 여러 개발자가 쓴다면 권한이 제한된 서비스 어카운트의 토큰을 kubeconfig에 등록해 kubectl 권한을 제한하는 것이 바람직합니다.
- kubeconfig는 보통 `~/.kube/config`에 있고 `KUBECONFIG` 환경 변수로 경로를 바꿀 수 있습니다. 세 파트로 구성됩니다.
    - clusters: API 서버 접속 정보 목록입니다. 원격 클러스터를 추가할 수 있습니다.
    - users: 사용자 인증 정보 목록입니다. 서비스 어카운트 토큰이나 루트 인증서로 발급한 하위 인증서를 넣을 수 있습니다. 이것만으로는 어느 클러스터에 쓸지 알 수 없습니다.
    - contexts: cluster와 user를 조합한 최종 사용 정보입니다. 여러 컨텍스트 중 하나를 선택해 사용하며 `current-context`로 현재 컨텍스트를 확인합니다.
- 이 원리로 로컬 개발 클러스터, AWS 운영 클러스터 등을 전환하며 쓸 수 있습니다. kubeconfig가 유출되면 클러스터 보안이 매우 취약해집니다.
- 서비스 어카운트를 kubeconfig에 등록하는 과정은 다음과 같습니다.

```bash
export TOKEN=$(kubectl create token alicek106)
kubectl config set-credentials alicek106-user --token=$TOKEN
kubectl config get-clusters
kubectl config set-context my-new-context --cluster=kubernetes --user=alicek106-user
kubectl config get-contexts
kubectl config use-context my-new-context
```

- my-new-context로 전환하면 alicek106으로 요청하는 것과 같아서 `get service`만 성공하고 나머지는 Forbidden입니다.
- 실습 후 `kubectl config use-context kubernetes-admin@kubernetes`로 되돌립니다.
- 실제 운영에서는 토큰을 직접 넣기보다 OIDC 등으로 사용자를 동적으로 인증하는 것이 일반적입니다.

## 10.6 유저(User)와 그룹(Group)의 개념

- 유저는 실제 사용자, 그룹은 유저의 집합입니다. 롤 바인딩이나 클러스터 롤 바인딩의 subjects kind에 ServiceAccount 대신 User나 Group을 쓸 수 있습니다.
- 서비스 어카운트도 개념상 유저의 한 종류입니다. 쿠버네티스에는 유저나 그룹이라는 오브젝트가 없으므로 `kubectl get user`, `kubectl get group`은 사용할 수 없습니다.
- `system:serviceaccount:default:alicek106`은 서비스 어카운트를 지칭하는 고유한 유저 이름입니다. 따라서 롤 바인딩에서 `kind: User`, `name: system:serviceaccount:default:alicek106`으로 써도 권한이 정상 부여됩니다.
- 미리 정의된 유저와 그룹은 `system:` 접두어를 씁니다.
    - `system:serviceaccounts`: 모든 네임스페이스의 모든 서비스 어카운트(예제 10.11 service-read-role-all-sa.yaml에서 kind: Group으로 클러스터 롤 바인딩)
    - `system:serviceaccounts:<네임스페이스>`: 특정 네임스페이스의 모든 서비스 어카운트
    - `system:authenticated`: 인증에 성공한 그룹
    - `system:unauthenticated`: 인증에 실패한 그룹
    - `system:anonymous`: 인증에 실패한 유저

### 다양한 인증 방법에서의 User와 Group

- 인증 방법은 서비스 어카운트 토큰만 있는 것이 아닙니다. 기본 kubeconfig는 쿠버네티스가 자체 지원하는 x509 인증서를 쓰고, 별도 인증 서버로 깃허브, 구글 계정, LDAP 등도 쓸 수 있습니다.
- 별도 인증 서버는 보통 덱스(Dex) 같은 별도 솔루션으로 구축합니다.
- 별도 인증 방법에서 유저와 그룹이 특히 유용합니다. 예를 들어 깃허브 Organization 인증이라면 팀 이름을 그룹으로, 사용자 이름을 유저로 매칭할 수 있습니다.
    - 팀 DevOps 전원에게 권한: `kind: Group`, `name: DevOps`
    - 개발자 Carol 개인에게 권한: `kind: User`, `name: Carol`
- 어떤 데이터를 유저, 그룹으로 매칭할지는 인증 서버나 API 서버 설정에 따라 다릅니다(예: 사용자 이름 대신 이메일).

## 10.7 x509 인증서를 이용한 사용자 인증

- 쿠버네티스는 자체 서명한 루트 인증서를 사용합니다. kubeadm은 `/etc/kubernetes/pki`, kops는 S3 버킷의 `<클러스터 이름>/pki/`에 저장됩니다.
    - `ca.crt`는 루트 인증서, `ca.key`는 그 비밀키입니다.
    - apiserver.crt 등은 루트에서 발급된 하위 인증서로, 핵심 컴포넌트 간 보안 연결에 쓰입니다.
- 루트 인증서로 발급한 하위 인증서로 사용자를 인증할 수 있으며, 기본 kubeconfig도 이 방식입니다.
- 하위 인증서를 직접 만드는 과정은 다음과 같습니다.

**비밀키와 CSR 생성**

```bash
openssl genrsa -out alicek106.key 2048
openssl req -new -key alicek106.key \
  -out alicek106.csr -subj "/O=alicek106-org/CN=alicek106-cert"
```

- 핵심은 `subj`입니다. 인증서의 CN(Common Name)이 유저, O(Organization)가 그룹으로 취급됩니다.
- 기본 kubeconfig 인증서는 `O=system:masters`, `CN=kubernetes-admin`이며, 클러스터 롤 바인딩 cluster-admin이 system:masters 그룹에 cluster-admin을 부여하기 때문에 관리자 권한을 쓸 수 있었습니다.

**CertificateSigningRequest로 서명 요청**

- openssl로 루트 비밀키에 직접 접근해 서명할 수도 있지만 유출 위험 때문에 바람직하지 않습니다. 대신 API로 간접 서명합니다(예제 10.12 alicek106-csr-k8s-latest.yaml, 1.22 이상 기준).
- 주요 항목은 `signerName: kubernetes.io/kube-apiserver-client`, `groups: system:authenticated`, `request: <CSR>`, `usages: digital signature / key encipherment / client auth`입니다.
- `.csr`을 base64 인코딩해 request에 넣습니다.

```bash
export CSR=$(cat alicek106.csr | base64 | tr -d '\n')
sed -i -e "s/<CSR>/$CSR/g" alicek106-csr-k8s-latest.yaml
kubectl apply -f alicek106-csr-k8s-latest.yaml
kubectl get csr   # CONDITION: Pending
```

**관리자 입장에서 승인 후 인증서 추출**

```bash
kubectl certificate approve alicek106-csr   # Approved,Issued
kubectl get csr alicek106-csr -o jsonpath='{.status.certificate}' | base64 -d > alicek106.crt
```

- macOS에서는 base64에 `D` 옵션을 씁니다.

**kubeconfig에 사용자와 컨텍스트 등록**

```bash
kubectl config set-credentials alicek106-x509-user \
  --client-certificate=alicek106.crt --client-key=alicek106.key
kubectl config set-context alicek106-x509-context \
  --cluster kubernetes --user alicek106-x509-user
kubectl config use-context alicek106-x509-context
```

- `-client-certificate`, `-client-key` 옵션으로 kubeconfig 등록 없이 임시 테스트도 가능합니다.
- 전환 후 `kubectl get svc`는 Forbidden이며 에러에 `User "alicek106-cert"`가 표시됩니다. 인증은 성공했으니 권한만 주면 된다는 뜻입니다. Unauthorized가 나오면 인증 자체가 실패한 것입니다.

**권한 부여 (예제 10.13 x509-cert-rolebinding-user.yaml)**

- subjects에 `kind: User`, `name: alicek106-cert`, roleRef에 service-reader 롤을 지정합니다.
- 현재 컨텍스트에는 권한이 없으므로 `-context kubernetes-admin@kubernetes`로 관리자 컨텍스트를 임시 사용해 적용합니다.
- 이후 `kubectl get svc`가 정상 출력됩니다.

**x509 방식의 한계**

- 인증서 유출 시 하위 인증서를 파기(revoke)하는 기능이 없습니다.
- 파일로 인증 정보를 관리하는 것은 보안상 바람직하지 않습니다.
- 따라서 개념만 알고, 실제 운영에서는 덱스 등으로 깃허브, LDAP 같은 서드파티 인증을 연동하는 것이 더 효율적입니다.
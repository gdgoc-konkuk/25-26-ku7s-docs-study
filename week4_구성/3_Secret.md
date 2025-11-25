# Secret

“쿠버네티스에서 **민감한 정보(기밀 정보)**를 저장하는 리소스”

- ConfigMap과 거의 똑같은 구조지만 **보안을 전제로 만들어진 객체**
- 저장하는 정보: 비밀번호, DB 계정 정보, API 토큰 등

→ ConfigMap: 일반 설정용 / Secret: 민감한 설정용

# 1. Secret이 필요한 이유

### 1) 민감한 정보를 이미지에 넣지 않기 위해

- 예전 방식: 환경 변수나 설정 파일을 Docker 이미지에 포함 → **보안 위험**
    - `kubectl get apply -o yaml` 하면 비밀번호가 그대로 보임
    - Git에 YAML이나 Dockerfile을 올리면 레포에 비밀번호 영구 저장
    - 이미지 푸시한 레지스토리에도 그대로 남음
        
        → 실수 한 번이면 **비밀번호를 인터넷 전체에 뿌리는 것**과 비슷
        

Secret 방식: 이미지에는 코드만, 민감 정보는 Secret에 분리 저장 → **안전**

### 2) base64 인코딩 + etcd 암호화 지원

Secret은 API 서버 뒤에서 **etcd라는 “key-value”** **DB**에 저장,  `data:` 필드 값은 base64 인코딩되어 저장됨.

```makefile
password: cGFzc3dvcmQ=
```

base64는 **암호화가 아니라 인코딩**이지만, 운영 환경에서는 etcd 암호화를 반드시 키라고 가이드에서 정의

→ Secret은 etcd 암호화, 엄격한 RBAC 정책, tmpfs 마운트 같은 보안 기능과 같이 쓰도록 디자인됨.

### 3) Secret은 권한 제어(RBAC)로 보호됨

Secret은 API 리소스 → `get`, `list`, `watch` 같은 권한을 별도로 제어 가능

- 일반 개발자는 자신의 네임스페이스 Secret만 읽게 한다든가
- 특정 서비스 계정만 Secret을 읽수 있게 RBAC 묶기

### * RBAC(Roel-Based Access Control): 역할 기반 접근 제어

“누가 어떤 리소스(Pod, ConfigMap 등)에 어떤 행동(get, list 등)을 할 수 있는지를 결정하는 권한 시스템

- Role/ClusterRole: “무엇을 할 수 있는지” 규정하는 문서
- RoleBinding/ClusterRoleBinding: “누가 이 Role을 쓰는가?” 연결하는 문서
- User: 클러스터에 접속하는 실제 사람
- ServiceAccount: Pod는 이를 통해서 “내가 이 Secret을 읽어도 되나요?” 이런 권한을 체크 받

# 2. Secret의 기본 구조

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

### 2-1. `type`

```yaml
type: Opaque
```

- 이 Secret이 **어떤 용도/포맷**인지 나타내는 문자열
- `Opaque`는 **기본값**으로, 그냥 “일반적인 키-값 비밀 저장”이라는 뜻
- 특별한 포맷을 가진 Secret은 `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson` 같은 타입을 사용한다.

(`kubernetes.io/tls`: 쿠버네티스에서 TLS 인증서와 개인 키를 저장하는 데 사용되는 시크릿(Secret) 타입)

### 2-2. `data`

```yaml
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

- `data` 안에 있는 값들은 **반드시 base64 인코딩되어 있어야 한다.**
- 예:
    
    ```bash
    echo -n 'admin' | base64    # YWRtaW4=
    echo -n 'password123' | base64
    ```
    
    Pod 안에서 volume이나 환경 변수로 읽을 때 → **자동으로 base64 디코딩된 상태**로 제공된다. 
    
    * base64는 “암호화”가 아니라 “인코딩”일 뿐임
    

### 2-3. `stringData` (사람 친화적인 입력용)

```yaml
stringData:
  username: admin
  password: password123
```

- `stringData`는 **쓰기 편하라고 만들어 둔 필드**
- 여기 들어간 값들은 apply 시점에 자동으로 base64로 변환되어 `data`에 저장됨.
- YAML 쓸 때 `data`에 직접 base64를 계산해서 넣는 대신,
    
    **사람이 읽기 좋은 stringData를 주로 쓰고, 실제 저장은 data로 되도록** 쓰는 패턴이 많다.
    

### 2-4. 기타

- 전체 Secret 크기 역시 **1MiB 제한** (ConfigMap과 동일)
- key 이름 규칙: 알파벳/숫자//`_`/`.` 가능 (ConfigMap과 유사)

# 3. Secret 타입 종류

### 3-1. `Qpaque`: 가장 기본

```yaml
type: Opaque
```

- “그냥 key-value 형태의 비밀 값 모음”
- username/password, 토큰 같은 일반적인 비밀 정보 저장할 때 사용
- 포맷에 대한 제약 거의 없음

### 3-2. **`kubernetes.io/tls` – TLS 인증서+키 전용**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64 cert>
  tls.key: <base64 key>
```

- 주로 Ingress TLS, HTTPS 설정에 사용
- 필수 key: `tls.crt`(서버 인증서 파일), `tls.key`(해당 인증서에 대한 개인 키 파일) → base64로 인코딩
- API 서버가 이 키들이 있는지는 확인해주지만, 내용이 진짜 유효한 인증서인지는 검사 안한다.
- `kubectl create secret tls` 라는 전용 명령으로 쉽게 만들 수 있다:
    
    ```bash
    kubectl create secret tls my-tls-secret \
      --cert=path/to/cert/file \
      --key=path/to/key/file
    ```
    

### 3-3. **`kubernetes.io/dockerconfigjson` – 프라이빗 레지스트리 로그인 정보**

```yaml
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64 of ~/.docker/config.json>
```

- 프라이빗 Docker 레지스트리 로그인 정보 저장용
- Pod에서 `imagePullSecrets`로 참조해서 프라이빗 레지스토리에서 이미지를 pull할 수 있게 해줌.

### 3-4. `kubernetes.io/ssh-auth` – SSH private key

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-ssh-auth
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64 of private key>
```

- SSH 인증용 비밀키를 저장할 때 사용
- 필수 key: `ssh-privatekey` (SSH 인증에 필요한 개인 키를 저장하는데 사용되는 “key-value” 쌍)

* SSH: 네트워크 상의 다른 컴퓨터를 원격으로 제어하고 파일 전송 등을 가능하게 하는 원격 접속 도구

# 4. Secret 생성 방법

### 1) key=value 리터럴로 생성

```yaml
kubectl create secret generic my-secret \  # generic -> opaque 타입 설정
  --from-literal=username=admin \          # --from-literal: key=value를 바로 넣는 방식  
  --from-literal=password=1234
```

→ 자동으로 base64 인코딩 처리된다.

### 2) 파일로부터 생성

```lua
kubectl create secret generic my-cert --from-file=tls.crt --from-file=tls.key
```

→ TLS 인증서 파일을 Secret에 저장할 때 자주 사용한다.

### 3) YAML 직접 작성 (가장 권장)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: 1234
```

### 4) 이미 있는 Secret 수정

1. 새 비밀번호를 base64 인코딩:
    
    ```yaml
    echo -n 'birdsarentreal' | base64      # => YmlyZHNhcmVudHJlYWw=
    ```
    
2. YAML의 `data.password` 값 변경:
    
    ```yaml
    data:
      username: YWRtaW4=
      password: YmlyZHNhcmVudHJlYWw=
    ```
    
3. 다시 apply:
    
    ```yaml
    kubectl apply -f secret.yaml
    ```
    

# 5. Secret을 Pod에서 사용하는 방법

### 1) 환경 변수로 사용

```yaml
env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: username
```

### 2) envFrom으로 전체 가져오기

```yaml
envFrom:
  - secretRef:
      name: db-secret
```

### 3) Volume으로 파일 마운트 (가장 안전한 방식)

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
containers:
  - name: app
    volumeMounts:
      - name: secret-volume
        mountPath: /etc/secret
        readOnly: true
```

### 4) docker-registory Secret 자동 사용

컨테이너 이미지 Pull 시 인증이 필요하면, `imagePullSecrets:` 사용하는 방식도 존재함.

# 6. Secret 변경이 Pod에 반영되는 방식

- **환경변수 방식:** Pod를 재시작해야 반영된다
    - 환경 변수는 컨테이너 시작 시점에만 읽힌다.
- **Volume 마운트 방식:** 자동으로 변경을 감지하여 파일이 업데이트

# 7. Secret 관련 중요 보안 포인트

- **base64는 암호화가 아니다 →** etcd 암호화는 반드시 활성화 필요
- **RBAC으로 Secret 읽기 권한 최소화 →** read 권한 제한
- Secret을 볼륨으로 마운트할 때,, **Secret은 tmpfs에 마운트해야 안전하다 (기본 동작)**
    
    tmpfs은 RAM 기반 임시 파일 시스템으로:
    
    - Secret 데이터가 RAM에만 존재하므로, 물리적인 영구 저장소에 기록 ❌
    - Pod 종료 시 자동 삭제
    - 노드 재부팅 시 데이터 소멸
- Secret과 ConfigMap은 같은 네임스페이스에서만 참조 가능

# 8. ConfigMap vs Secret

| 항목 | ConfigMap | Secret |
| --- | --- | --- |
| 용도 | 일반 설정값 | 민감한 설정값 |
| 저장 방식 | plain text | base64 인코딩(암호화 가능) |
| 보안 수준 | 낮음 | 높음 |
| etcd 저장 | 평문 | 암호화 가능 |
| Pod 사용 방식 | env, envFrom, volume | env, envFrom, volume |
| 주의점 | 값 공개되어도 괜찮아야 함 | 민감한 값만 넣어야 함 |
# Service

# 1️⃣ Service 개요

## 💡 서비스가 필요한 이유

- **파드는 일시적이다:**
    
    쿠버네티스 스케줄러가 언제든지 파드를 재시작하거나 다른 노드로 옮길 수 있는데, 이때 **파드의 IP 주소가 변경**된다. 
    
    | 상황 | 결과 |
    | --- | --- |
    | 파드 재시작 | 새 파드가 만들어지고 IP가 변경됨 |
    | 노드 다운 | 다른 노드에서 파드가 새로 스케줄됨 |
    | 스케일 조정 (ReplicaSet) | 파드 수가 동적으로 바뀜 |
    
    → 클러스터 안에서 다른 애플리케이션(ex. 프론트엔드)은 항상 **“백엔드 파드”**를 찾아 통신하는데, 파드의 IP가 계속 바뀐다면, 연결이 끊어지거나 새 주소를 일일이 알아야 하는 문제가 발생한다.
    
- **서비스(Service)**는 이 문제를 해결하기 위해 도입된 **“네트워크 추상화 계층”**이라고 볼 수 있다.
- 즉,
    - 여러 파드를 하나의 **논리적 그룹**으로 묶고,
    - 클라이언트가 그 그룹을 **하나의 IP 또는 이름으로 접근**할 수 있게 함.
    - 서비스가 생성되면, 쿠버네티스는 이를 위해 **가상 IP (ClusterIP)** 를 자동으로 할당 → 불변

결론: 서비스는 여러 파드에 대한 **하나의 고정된 접근 지점**을 제공하는 기능을 가지

# 2️⃣ Service란 무엇인가?

- **서비스란?**
    
    어떤 파드 집합을 하나의 논리적 대상으로 묶을지, 그 파드에 어떻게 접근할지를 정의하는 개체
    
- 파드 집합은 보통 **레이블 셀렉터(Label Selector)** 를 기반으로 결정된다.

* 파드의 논리적 집합: 같은 조건(selector)에 부합하는 파드들

### ⚙️ Service의 주요 구조

| 필드명 | 설명 |
| --- | --- |
| `apiVersion` | 항상 `v1` |
| `kind` | `Service` |
| `metadata.name` | 서비스 이름 |
| `spec.selector` | 연결할 파드 레이블 |
| `spec.ports[].port` | 서비스가 노출할 포트 (클라이언트 입장) |
| `spec.ports[].targetPort` | 실제 파드의 컨테이너 포트 |
| `spec.type` | 서비스 타입 (기본값: ClusterIP) |
| `spec.clusterIP` | 내부 IP 주소 (자동 생성) |

```yaml
apiVersion: v1   # 초기 안정 버전 API, 가장 표준적이고 안정적인 API 버전을 사용하겠다는 의미
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080   # 파드의 포트
```

예시: `app: backend` 레이블을 가진 파드들이 여러 개 있을 떄, 이들을 하나로 묶어 `backend-service` 라는 서비스로 내보낼 수 있다. 

- `selector` → `app=backend`  라벨을 가진 모든 파드를 연결한다.
- 서비스의 `port: 80` 으로 요청이 오면, 파드의 8080 포트로 전달된다.
- 클러스터 내부에서는 `backend-service` 라는 이름으로 접근이 가능하다.

### ✅ **Label 개념 알고 가기**

**Label:** 쿠버네티스 리소스에 붙일 수 있는 **키-값 쌍의 metadata**

**Label Selector:** 쿠버네티스 리소스(서비스 등)가  **어떤 파드를 대상으로 삼을지 지정하는 필터** 역할  

- Equality-based Selector: `=` 또는 `!=` 로 단순 비교
- Set-based Selector: `in`, `notin`, `exists` 등 집합 조건

### ⚒️ Service의 동작 방식

1. 사용자가 서비스 IP로 요청을 보낸다.
2. kube-proxy가 클러스터 노드 

# 3️⃣ 셀렉터(Selector) & 엔드포인트(Endpoints)

- 쿠버네티스 컨트롤 플레인은 서비스의 **셀렉터를 기준**으로 매칭되는 파드를 찾아 **엔트포인트** 라는 리소스로 관리한다.
    
    → 이 엔드포인트들이 실제 네트워크 연결 대상이 된다.
    
- 파드가 새로 생성/삭제될 때마다 자동으로 EndpointSlice가 갱신된다.
- 즉,
    - 서비스 = 논리적 접근점
    - 엔드포인트 = 실제 연결할 파드 IP 집합

### ✏️ Endpoints & EndpointSlice 차이점

- Endpoints는 **초기 버전,** EndpointSlice는 **업데이트 후 현재 표준**으로 사용되는 용어 → 같은 개념
- Endpoints 구성 (초기 버전)
    
    서비스 생성 → 컨트롤 플레인(kube-proxy)이 파드를 찾고 아래처럼 하나의 Endpoints 리소스를 만든다.
    
    ```yaml
    apiVersion: v1
    kind: Endpoints
    metadata:
      name: my-service
    subsets:
      - addresses:
          - ip: 10.244.1.4
          - ip: 10.244.1.5
        ports:
          - port: 9376
    ```
    
    - 모든 파드 IP가 `addresses` 배열 하나에 전부 삽입 → 파드가 늘어날수록 이 객체의 크기도 같이 커짐
    - kube-proxy도 계속 전체를 다시 읽어야 하므로 **비효율적**
- EndpointSlice 구성 (현재 표준)
    
    ```yaml
    apiVersion: discovery.k8s.io/v1
    kind: EndpointSlice
    metadata:
      name: my-service-abcde
      labels:
        kubernetes.io/service-name: my-service
    addressType: IPv4
    ports:
      - name: http
        port: 9376
    endpoints:
      - addresses: ["10.244.1.4"]
      - addresses: ["10.244.1.5"]
    ```
    
    - 한 서비스에 여러 `EndpointSlice` 객체를 넣어 관리한다. (최대 100개 endpoint 당 1개의 slice)
    - kube-proxy는 변결될 슬라이스만 보면 되므로 부하 ↓
        
        → **API 서버 성능, 네트워크 효율, 안정성**이 크게 향상된다. 
        

# 4️⃣ Service의 동작 방식

1. 클라이언트 파드가 **서비스 이름**으로 요청을 보내면 CoreDNS가 해당 이름을 **ClusterIP**로 바꿔준다.
2. DNS를 통해 얻은 IP로 클라이언트가 실제 요청을 보낸다.
    
    이 IP는 쿠버네티스가 서비스마다 제공한 **고정된 가상 IP**이기 때문에 변하지 않는다.
    
3. 노드의 kube-proxy는 iptables/IPVS 규칙을 이용해 들어온 요청을 “어떤 파드로 보낼지”를 결정한다.  
    
    → 주로 **라운드 로빈 / 랜덤 방식**으로 파드 중 하나를 선택한다.
    
4. 선택된 파드는 요청을 받아 실제로 작업을 수행한다.  → **컨테이너가 실제로 일을 하는 구간**
5. 만약 파드가 죽거나 새로 생기면, `EndpointSlice`를 통해 자동으로 반영된다.
    
    `endpointSlice` 목록이 바뀌면 kube-proxy가 즉시 새로운 규칙을 업데이트함에 따라,
    
    파드가 교체되어도 서비스 IP는 그대로 유지되고 사용자는 아무런 중단 없이 계속 접근이 가능하다.
    
    → 서비스는 항상 “살아있는 파드만 연결” 되도록 스스로 갱신한다.
    

# 5️⃣ Service 타입별 상세 설명

서비스 타입은 “이 서비스가 어느 범위까지, 어떤 방식으로 노출될지를 결정하는 설정값” 이다. 

| 타입 | 접근 범위 | 설명 |
| --- | --- | --- |
| **ClusterIP** | 내부 전용 | 클러스터 내부 파드끼리 통신 |
| **NodePort** | 외부 접근 | 클러스터 바깥에서 노드 IP+포트로 접근 |
| **LoadBalancer** | 외부 접근 (클라우드 LB 이용) | 클라우드 제공 로드밸런서를 이용해 외부 트래픽 분산 |
| **ExternalName** | 외부 리소스 연결 | 외부 도메인(DB, API 등)을 쿠버네티스 내부 이름으로 연결 |
| **Headless** | 내부 파드 개별 접근 | 클러스터IP 없이, 파드 IP를 직접 노출(StatefulSet 등에서 사용) |

### **1. ClusterIP (기본값)**

- **기본 서비스 타입**
- 클러스터 내부에서만 접근이 가능하다. (외부에서는 접근 ❌)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 80        # 서비스가 노출하는 포트
      targetPort: 8080  # 파드 컨테이너 포트
```

- 서비스 이름 또는 ClusterIP로 접근한다.
    
    ex. `curl http://backend-service` → 클러스터 내부 파드들이 접근 가능
    
- ✅ 장점: 간단하고 빠름 / ❌ 단점:  외부 사용자 접근 불가

### 2. Nodeport

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 31000     # 기본적으로 범위는 30000 - 32767
```

- 각 노드의 IP에 지정된 포트를 열어 외부 트래픽을 받는다.
- 모든 노드가 동일한 포트를 연다.
    
    → 외부: `https://<NodeIP>:31000` 으로 접근 가능 / 내부: ClusterIP가 자동 생성되어 파드에 전달
    
- ✅ 장점: 간단한 외부 노출 가능 / ❌ 단점:  포트 충돌 및 보안 문제 발생 가능성

### 3. LoadBalancer

- 클라우드 환경에서 외부 트래픽을 **자동으로 분산시켜주는 서비스**로
    - 외부에서 접근 가능한 **공인 IP**를 만들고, 그 IP로 들어온 요청을 노드들에 균등하게 전달한다.
    - 노드 안에서 서비스가 파드로 트래픽을 다시 분배한다.
        
        → 사용자는 **공인 IP 하나**로 접근하지만, 실제로는 여러 파드가 나눠서 처리하는 구조이다.
        
- NodePort 기반 위에 **클라우드 로드밸런서**가 자동으로 생성되고, 서비스와 연결된다.
    
    GCP/GKE, AWS/EKS, Azure/AKS 등에서 주로 사용한다.
    

* 클라우드 로드밸런서: 인터넷에서 들어오는 수많은 요청을 여러 서버(노드나 파드)로 나눠주는 역할

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 9376
```

- NodePort + ClusterIP가 내부적으로 함께 동작
- ✅ 장점: 실서비스용 외부 접근에 최적화 / ❌ 단점: 클라우드 환경 의존적, 비용 발생

### 4. ExternalName

- 파드나 클러스터 내부 자원과 연결되지 않고, 단순히 서비스 이름을 외부 DNS 이름으로 매핑한다.

```yaml
spec:
  type: ExternalName
  externalName: db.example.com
```

→ 클라언트가 서비스 요청하면 CoreDNS가 `db.example.com` 으로 CNAME 반환한다.

* CNAME: 쉽게 말해 **별명(Alias)** 을 만드는 레코드, 한 도메인 이름을 다른 도메인 이름으로 연결하는 DNS 설정 방식

- ✅ 장점: 외부 DB, API가 있어도 쿠버네티스 내부 서비스처럼 이름으로 접근 가능
- ❌ 단점: DNS 이름만 바꿀 뿐 실제 트래픽을 프록시(중간서버)로 전달 X

→ 쿠버네티스 네트워크 기능이 적용되지 않음 

### 5. Headless Service

- 로드밸런싱 없이 **파드 IP 전체 목록**을 반환한다.
- 주로 StatefulSet(데이터베이스 등)에서 사용한다.

```yaml
spec:
  clusterIP: None
  selector:
    app: my-db
```

- ✅ 장점: 각 파드로 직접 접근 가능 / ❌ 단점: 클라이언트가 직접 파드 리스트를 관리해야 함

### 6. Selector 없는 서비스

- **특징:** `spec.selector` 생략한다.
    
    대신 **Endpoints/EndpointSlice를 수동**으로 구성 (외부 IP/정적 백엔드 연결용).
    
- **사용처:** 가상 IP를 유지하면서 **클러스터 외부**로 포워딩 지점 만들 때.

### * StatefulSet

**각 파드가 고유한 정체성, 저장소, 순서**를 가져야 하는 경우, StatefulSet이 이를 담당, 관리한다.

| Pod 이름 | 역할 | IP | 데이터 |
| --- | --- | --- | --- |
| db-0 | Master | 10.1.1.10 | volume-0 |
| db-1 | Replica | 10.1.1.11 | volume-1 |
| db-2 | Replica | 10.1.1.12 | volume-2  |

**특징**

- 파드 이름 고정 (`pod-name-0`, `pod-name-1`, …)
- 순차적 생성/삭제 (0부터 차례대로)
- 각 파드에 **고유한 저장소** 연결
- 재시작해도 동일한 이름 + 볼륨 유지

# 6️⃣ DNS와 서비스 디스커버리

### 🧠 서비스 디스커버리란?

“서로 다른 서비스들이 **서로를 찾아서 통신할 수 있게 해주는 시스템”**

- 프론트엔드 서비스 → “백엔드 API 서비스” 의 IP를 알아야 하는 경우
- 백엔드 서비스 → “DB 서비스”의 IP를 알아야 하는 경우

→ 파드의 IP가 바뀌어도, **DNS 이름을 기반하여 자동으로 찾아주는 기능**이다. 

### 1. 이름 체계 (FQDN)

```sql
서비스이름.네임스페이스.svc.cluster.local
```

- svc: 서비스 리소스임을 의미 / cluster.local: 클러스터 내부 도메인
- 기본 FQDN:  `my-svc.my-namespace.svc.cluster.local`  → 전체 도메인 이
- 같은 네임스페이스에선 `my-svc` 만으로도 접근 가능
- 다른 네임스페이스에서 접근: `my-svc.my-namespace`
    
    즉, 파드 → 서비스 간 통신은 IP 대신 “서비스 이름” 만 사용하면 된다.
    

### 2. 서비스 타입에 따른 DNS 동작 차이

| 서비스 타입 | DNS가 반환하는 결과 | 설명 |
| --- | --- | --- |
| **ClusterIP** | 하나의 IP (ClusterIP) | 서비스 가상 IP로 연결됨 |
| **Headless** | 여러 개의 IP (파드 각각의 IP) | 파드 개별로 직접 접근 가능 |
| **ExternalName** | CNAME(외부 도메인 이름) | 내부 이름 → 외부 도메인으로 매핑 |

# 7️⃣ 엔드포인트와 트래픽 흐름

- **EndpointSlice**는 서비스에 연결된 파드들의 주소록(IP/포드/상태)이다.
- 파드의 상태가 변결될 때 Slice가 자동으로 갱신한다.

### 트래픽 흐름 예시:

1. 클라이언트 파드가 `my-service` 로 요청
2. CoreDNS가 서비스의 ClusterIP를 반환
3. kube-proxy가 해당 IP를 파드 IP 중 하나로 전달
4. 파드가 응답 → 클라이언트에게 반환

# 8️⃣ Ingress 및 NetworkPolicy와의 관계

| 리소스 | 역할 |
| --- | --- |
| **Service** | 내부 네트워크 접근점 (L4 수준) |
| **Ingress** | HTTP/HTTPS 기반 외부 라우팅 (L7 수준) |
| **NetworkPolicy** | 파드 간 트래픽 제어 (보안) |
- “Service”는 **파드 집합의 네트워크 진입점**
- “Ingress”는 **외부 사용자의 진입점**
- “NetworkPolicy”는 **통신 허용 규칙**

### 1. Ingress

- **클러스터 외부 사용자(인터넷)**가 내부서비스(서버, API 등)에 접근할 수 있도록 열어주는 “정문”
- DNS나 경로 단위로 라우팅을 제어하고 TLS(HTTPS 인증서)도 설정할 수 있다.

### 2. NetworkPolicy

- NetworkPolicy는 **파드 간 통신 규칙**을 정의해.
- “어떤 파드가 어떤 파드와 어떤 포트로 통신할 수 있는가?”를 지정하지.
- 설정하지 않으면 **모두 통신 가능(열려 있음)**
    
    → 설정하면 **기본 차단 후, 허용된 것만 통신 가능(화이트리스트)**
    

> 🧍‍♂️ NetworkPolicy = 통신 허용/차단을 관리하는 방화벽
> 

```sql
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:        # 이 정책이 어떤 파드에 적용될지를 경정
    matchLabels:
      app: backend
  ingress:
    - from:          # 어디서 오는 트래픽을 허용할지를 정의 
        - podSelector:
            matchLabels:
              app: frontend
```
# 인그레스 컨트롤러(Ingress Controller)란?

**Ingress Controller** 는 Kubernetes 클러스터의 외부 트래픽을 관리하는 컴포넌트. Ingress 리소스에 정의된 규칙에 따라 HTTP/HTTPS 트래픽을 적절한 백엔드 서비스로 라우팅.

### 역할
- **외부 접근 관리**: 클러스터 외부에서 들어오는 요청을 처리
- **라우팅**: URL 경로나 호스트명에 따라 요청을 다른 서비스로 분배
- **TLS 종료**: SSL/TLS 암호화를 처리하고 내부 통신을 단순화
- **로드 밸런싱**: 여러 Pod 인스턴스 간에 트래픽을 분산
### 구조
Ingress 리소스는 선언적인 규칙만 정의하고, **실제 동작은 Ingress Controller가 수행** . 마치 Deployment(선언)와 controller manager(실제 Pod 생성)의 관계와 비슷.



# Nginx Ingress Controller

**Nginx Ingress Controller**는 Nginx를 기반으로 한 가장 인기 있는 Ingress Controller 구현체.

### 특징
- **높은 성능**: Nginx의 안정성과 고속 처리로 대규모 트래픽 처리 가능
- **세밀한 제어**: Nginx 설정을 통해 다양한 고급 기능 지원
- **활발한 커뮤니티**: 많은 플러그인과 확장 기능 제공
- **Annotation 기반 설정**: Ingress 리소스의 annotation으로 쉽게 동작 커스터마이징 가능

### 주요 기능
- URL 재작성(rewrite), 리다이렉트
- Rate limiting(속도 제한)
- WebSocket 지원
- CORS 설정
- 세션 유지 및 스티키 세션
- 커스텀 에러 페이지

### 설치 및 사용
```yaml
# 기본 Ingress 리소스 예시
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 8080
```
# Istio 상세 설명

## Istio란?

**Istio**는 마이크로서비스 아키텍처를 관리하기 위한 **Service Mesh** 플랫폼. 클러스터 내 서비스 간 통신(East-West 트래픽)과 외부 트래픽(North-South)을 모두 관리하며, 보안, 관찰성, 트래픽 제어를 제공.


## Service Mesh 개념

Service Mesh는 서비스 간 통신을 관리하는 전용 인프라 계층입. 각 서비스 옆에 **사이드카 프록시(Envoy)**를 배치하여 모든 네트워크 통신을 중재.

```
기존 방식:
App A → App B (직접 통신)

Service Mesh 방식:
App A → Envoy Sidecar → Envoy Sidecar → App B (중개된 통신)
```

---

## Istio 아키텍처

### 1. 데이터 플레인 (Data Plane)
- **Envoy Proxy**: 각 Pod 옆에 자동 주입되는 사이드카 프록시
- 모든 서비스 간 트래픽을 중재합니다
- 네트워크 레벨의 세밀한 제어 가능

### 2. 컨트롤 플레인 (Control Plane)
여러 컴포넌트로 구성됩니다:

**Istiod**
- 중앙 집중식 컨트롤 플레인
- 설정을 데이터 플레인에 전달
- 서비스 발견 담당
- 인증 정책 관리

**Ingress/Egress Gateway**
- 클러스터 외부 트래픽의 진입/진출점


### 특징 : RequestAuthentication & AuthorizationPolicy
JWT 인증과 권한 관리:

```yaml
# JWT 인증
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
spec:
  jwtRules:
  - issuer: "https://auth-server.example.com"
    jwksUri: "https://auth-server.example.com/.well-known/jwks.json"
---
# 권한 관리
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: policy
spec:
  selector:
    matchLabels:
      app: my-app
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/frontend"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/*"]
```


## 핵심 기능

### 1. 트래픽 관리
- **카나리 배포**: 신 버전에 점진적으로 트래픽 분산
- **A/B 테스팅**: 특정 사용자에게만 신 버전 제공
- **타임아웃 및 재시도**: 신뢰성 향상
- **Circuit Breaker**: 장애 확산 방지

### 2. 보안
- **mTLS (Mutual TLS)**: 서비스 간 자동 암호화 통신
- **인증**: JWT 기반 요청 인증
- **권한 관리**: 어떤 서비스가 어떤 요청을 할 수 있는지 제어

```yaml
# mTLS 정책
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT  # 모든 통신을 mTLS로 강제
```

### 3. 관찰성 (Observability)
- **분산 추적 (Distributed Tracing)**: Jaeger, Zipkin 통합
- **메트릭 수집**: Prometheus 통합
- **로그**: 모든 트래픽 로깅
- **시각화**: Kiali 대시보드로 서비스 간 관계 표시

### 4. Resilience (탄력성)
- Retry 정책
- Timeout 설정
- Circuit Breaker
- Outlier Detection (이상 동작 감지)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

---

## 설치 및 사용

### 설치
```bash
# istioctl 다운로드
curl -L https://istio.io/downloadIstio | sh -

# Kubernetes에 설치
istioctl install --set profile=demo -y

# 네임스페이스에 자동 사이드카 주입 활성화
kubectl label namespace default istio-injection=enabled
```

#### 배포 후 자동으로 Envoy 사이드카가 주입.


### 장점

- **고급 트래픽 제어**: 복잡한 라우팅 시나리오 구현 가능
- **마이크로서비스 최적화**: 서비스 간 통신에 특화
- **자동 mTLS**: 보안이 기본으로 포함
- **풍부한 관찰성**: 트래픽 시각화 및 디버깅 용이
- **엔터프라이즈급**: 대규모 시스템에 적합

### 단점

- **복잡성**: 학습곡선이 가파름
- **성능 오버헤드**: 사이드카 프록시로 인한 레이턴시 증가 (~10-30ms)
- **리소스 사용**: 각 Pod마다 Envoy 프록시 실행
- **운영 복잡도**: 트러블슈팅이 어려울 수 있음
- **버전 호환성**: Kubernetes 버전과의 호환성 주의 필요


### 사용 사례

- **금융 시스템**: 높은 보안 및 통신 제어 필요
- **대규모 마이크로서비스**: 수십 개 이상의 서비스 운영
- **복잡한 배포**: 카나리, A/B 테스팅이 필요한 경우
- **규제 환경**: mTLS 기반 감시와 감사 필요한 경우
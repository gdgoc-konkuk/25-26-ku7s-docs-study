https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/
> ㄴ 급하게 준비한 이유..
# Kubernetes Gateway API 기초 설명

Gateway API는 Kubernetes에서 네트워크 서비스를 제공하기 위한 **Ingress의 후속 API**입니다. 동적 인프라 프로비저닝과 고급 트래픽 라우팅 기능을 제공하는 API 모음이라고 보시면 됩니다.

## 왜 Gateway API가 만들어졌나요?

기존 Ingress API의 한계를 극복하기 위해서입니다. Ingress에서는 헤더 기반 매칭, 트래픽 가중치 분배 같은 고급 기능을 사용하려면 커스텀 annotation을 써야 했는데, Gateway API는 이런 기능들을 표준화된 방식으로 제공합니다.

## 핵심 설계 원칙

1. **역할 지향적(Role-oriented)**: 조직의 역할에 따라 API가 나뉩니다
   - 인프라 제공자: 클러스터 인프라 관리
   - 클러스터 운영자: 정책, 네트워크 접근 관리
   - 애플리케이션 개발자: 애플리케이션 설정 관리

2. **이식 가능(Portable)**: 다양한 구현체(컨트롤러)에서 동일하게 사용 가능

3. **표현력(Expressive)**: 복잡한 라우팅 규칙을 직관적으로 표현

4. **확장 가능(Extensible)**: 필요에 따라 커스텀 리소스 추가 가능

## 4가지 핵심 리소스

### 1. **GatewayClass**
게이트웨이의 "타입"을 정의합니다. 어떤 컨트롤러가 이 게이트웨이를 관리할지 지정합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
```

### 2. **Gateway**
실제 트래픽 처리 인프라의 인스턴스입니다. 로드밸런서나 프록시 서버로 생각하시면 됩니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

### 3. **HTTPRoute**
HTTP 트래픽을 어떻게 라우팅할지 정의합니다. 경로, 헤더 등을 기반으로 백엔드 서비스로 트래픽을 보냅니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "www.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /login
    backendRefs:
    - name: example-svc
      port: 8080
```

### 4. **GRPCRoute**
gRPC 트래픽 전용 라우팅입니다. gRPC 서비스와 메서드를 지정해서 라우팅할 수 있습니다.

## 리소스 간의 관계

```
GatewayClass → Gateway → HTTPRoute/GRPCRoute → Service
```

- **GatewayClass**가 어떤 컨트롤러를 사용할지 정의
- **Gateway**가 GatewayClass를 참조하여 실제 게이트웨이 인스턴스 생성
- **HTTPRoute/GRPCRoute**가 Gateway를 참조하여 라우팅 규칙 정의
- 최종적으로 백엔드 **Service**로 트래픽 전달

## Ingress와의 차이점

- **역할 분리**: Gateway API는 인프라 관리와 라우팅 설정을 명확히 분리
- **표준화**: annotation 없이 고급 기능 사용 가능
- **확장성**: 더 유연하고 확장 가능한 구조
- **마이그레이션**: Ingress에서 Gateway API로 일회성 변환 필요

## 실제 요청 흐름 예시

1. 클라이언트가 `http://www.example.com/login` 요청
2. DNS가 Gateway의 IP 주소 반환
3. Gateway가 요청을 받아 Host 헤더로 HTTPRoute 매칭
4. HTTPRoute의 매칭 규칙 확인 (경로가 `/login`인지 등)
5. 필요시 헤더 수정 등 필터 적용
6. 백엔드 Service로 요청 전달

Kubernetes Gateway API의 최신 구현체 정보를 찾아보겠습니다.# Kubernetes Gateway API 주요 구현체


## 🌟 가장 인기 있는 구현체들

### 1. **Istio**
- **타입**: Service Mesh + Ingress Gateway
- **특징**: 
  - 가장 성숙하고 널리 사용되는 서비스 메시
  - Gateway API v1 완전 지원 (GA)
  - HTTPRoute, GRPCRoute 모두 지원
  - 마이크로서비스 간 통신 제어 강력
- **적합한 경우**: 서비스 메시 + 트래픽 관리를 함께 필요로 할 때

### 2. **Envoy Gateway**
- **타입**: 전용 Gateway Controller
- **특징**:
  - Gateway API를 위해 특별히 설계된 **최초이자 유일한 프록시**
  - 고성능 Rust 데이터플레인
  - Envoy 기반의 경량화된 구현
  - v1.2 conformance 인증
- **적합한 경우**: 순수하게 Ingress 용도로만 사용할 때

### 3. **Cilium**
- **타입**: eBPF 기반 네트워킹 솔루션
- **특징**:
  - eBPF를 활용한 초고성능
  - Service Mesh 기능 포함
  - 사이드카 없는 아키텍처 가능
  - 보안 + 관찰성(observability) 통합
  - v1.2 conformance 인증
- **적합한 경우**: 성능이 중요하고 eBPF 활용을 원할 때

### 4. **Kong Gateway**
- **타입**: API Gateway + Ingress
- **특징**:
  - 엔터프라이즈급 API 관리 기능
  - 플러그인 생태계 풍부
  - Rate limiting, Authentication 등 내장
  - OpenAPI 통합 강력
- **적합한 경우**: API 관리 기능이 필요할 때

### 5. **NGINX Gateway Fabric**
- **타입**: NGINX 기반 Gateway
- **특징**:
  - NGINX의 안정성과 성능
  - Kubernetes 네이티브 설계
  - v1.2 conformance 인증
  - NGINX 사용자에게 친숙
- **적합한 경우**: NGINX 경험이 있고 안정성이 중요할 때

## ☁️ 클라우드 제공자 구현체

### 6. **AWS Gateway API Controller**
- Amazon VPC Lattice와 통합
- EKS 환경에 최적화
- AWS 네이티브 로드밸런서 활용

### 7. **Azure Application Gateway for Containers**
- Azure 관리형 L7 로드밸런서
- AKS에 최적화
- Azure 생태계와 긴밀한 통합

### 8. **Google Cloud (GKE)**
- GKE Gateway Controller
- Google Cloud Load Balancer 통합

## 📊 구현체 선택 가이드

```
성능 최우선 → Cilium (eBPF)
서비스 메시 필요 → Istio, Kuma
API 관리 기능 → Kong, Gloo, APISIX
간단한 설정 → Traefik, NGINX Gateway Fabric
순수 Gateway만 → Envoy Gateway
클라우드 네이티브 → AWS/Azure/GCP 제공 구현체
엔터프라이즈 → Kong, Gloo, Istio (상용 지원)
```

## 🔍 Conformance 인증 상태

2024년 11월 기준, **v1.2 conformance 인증**을 받은 구현체:
- Cilium
- Envoy Gateway
- Gloo Gateway
- Istio
- NGINX Gateway Fabric

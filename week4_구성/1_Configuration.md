# Configuration

**“애플리케이션 코드랑, 그 애플리케이션이 어떻게/어디서 동작해야 하는지를 분리해서 관리하는 것”**

- 코드 = 컨테이너 이미지
- 구성 = 환경 변수, 설정파일, 리소스 제한, 클러스터 접근 정보 등

이를 전부 **YAML + ConfigMap + Secret + 기타 설정 리소스**로 선언해둔다고 보면 된다. 

→ “파드 구성 관련해서 이런 것들을 봐라” 하는 페이지

### Why?

- 이미지를 하나로 **개발/스테이징/운영** 환경을 다 돌리기 위해서
- 환경마다 다른 건 “환경변수, 접속 URL, 리소스 크기” 정보만 바꾸면 되게 만들기 위해서
- 설정을 Git에 넣어서 **버전 관리/롤백** 가능하게 만들기 위해서

# 1. 구성 모범 사례

### 1-1. 선언형 + 버전 관리

- YAML 파일로 리소스를 정의히고, 그 파일을 Git 같은 버전 관리 시스템에 넣는다.
    - 누가 언제 어떤 설정을 바꿨는지 추적 가능
    - 잘못되면 이전 커밋으로 롤백

→ 기본적으로 `kubrctl apply -f`로 클러스터에 반영하는 흐름이다.

### 1-2. 코드 / 구성 분리

- 컨테이너 이미지 안에 **환경별 설정을 넣는 거** ❌
- **ConfigMap / Secret / 환경변수 / Volume** 등으로 외부에서 주입
    
    → 이미지는 그대로 두고 설정만 바꿔서 배포 가능 
    

### 1-3. YAML 사용 추천

- 사람이 읽고 쓰기에는 JSON보다 YAML이 훨씬 편하다고 문서에 명시되어 있다.

### 1-4. 최신 안정된 API 버전 사용

- 리소스 정의 시, `V1`같이 **최신 stable 버전**으로 쓰라고 명시되어 있다.

### 1-5. 환경별 차이는 “변수·Config 객체”만 변경

- dev / stage / prod 환경이 있어도 **YAML 구조는 최대한 같이 유지**한다.
- 아래와 같은 내용에만 차이를 둔다.
    - Configmap
    - Secret
    - replica 수, 리소스 요청/제한 값

# 2. 구성에 관련된 대표 요소들

### 2-1. ConfigMap: 비밀이 아닌 설정값

- **기밀이 아닌 설정**을 “key-value”로 저장하는 API 오브젝트
- Pod에서
    - 환경 변수로 읽거나 → “[week2] 컨테이너 환경변수 - 2. 환경변수 참조”
    - CMD/ARGS 인자로 전달하거나
    - volume으로 파일처럼 마운트해서 읽을 수 있다.

### 2-2. Secret: 비밀 설정값

- 비밀번호, 토큰, 인증서 같은 **민감 정보** 저장용 API 오브젝트
- 구조는 ConfigMap이랑 비슷하지만,
    - base64 인코딩 (바이너리 데이터를 텍스트로 변환하는 방식)
    - 일부 스토리지/전송 구간에서 암호화 옵션 적용 가능
    - 접근 제어를 더 타이트하게 가져가는 걸 전제로 한다.

### 2-3. Liveness / Readiness / Startup Probe: 헬스 체크

- **Liveness Probe**
    - “이 컨테이너가 죽은 상태인가?” 확인
    - 실패하면 kubelet이 컨테이너 재시작
- **Readiness Probe**
    - “이 Pod이 트래픽을 받은 준비가 됐나?” 확인
    - 실패하면 Service 레벨에서 트래픽을 안 보냄
- **Startup Probe**
    - 기동이 느린 애플리케이션을 위해, 시작이 끝날 때까지 liveness 체크를 늦추는 역할

→ 구성 관점: **헬스 체크를 YAML에 선언해서, 서비스&재시작 정책을 자동으로 맞춘다**

### 2-4. Resource Management: 리소스 요청/제한 설정

→  “[**파드 및 컨테이너 리소스 관리](https://kubernetes.io/ko/docs/concepts/configuration/manage-resources-containers/)”** 섹션 

- Pod/Container에 **** **CPU·메모리 요청(request)와 상한(limit)을 설정**
    - 스케줄러가 어느 노드에 올릴지 결정하고
    - 노드에서 리소스 초과 사용을 제어한다.
- 이 설정에 따라 Pod가 **QoS 클래스**를 가지게 된다.

* QoS 클래스: Pod의 CPU/메모리 리소스 보장 정도를 구분하기 위한 등급
                         (노드에 리소스가 부족해졌을 때 어떤 Pod을 먼저 죽일지(추방할지) 결정하는 우선순위 체계)

** Guaranteed(최상위 보호): 모든 컨테이너의 request == limit

** Burstable(중간 보호): request/limit 일부만 있거나 다름

** BestEffort(보호 없음): request/limit 모두 없음

### 2-5. Kubeconfig: 클러스터 접근 구성

- `~/.kube/config` 같은 파일
    - cluster 정보 (API 서버 주소, 인증서)
    - user 정보 (토큰/인증서)
    - context (어느 클러스터 + 어느 네임스페이스 + 어느 유저 조합) 을 저장해두고
- `kubectl config use-context`로 쉽게 전환 가능

→ 구성 관점: **사람/툴이 클러스터에 접근하는 방법도 하나의 구성**

# 3. 간단한 구성 예시(YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: app
      image: my-app:1.0   # 코드(이미지)
      env:
        - name: APP_MODE  # 구성 ①: 환경 변수
          value: "production"
      resources:          # 구성 ②: 리소스 관리
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      livenessProbe:      # 구성 ③: 헬스체크
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5

```

실제로는 여기에 ConfigMap, Secret 연동이 추가되고, Deployment로 감싸고… 이런 식으로 점차 커진다.
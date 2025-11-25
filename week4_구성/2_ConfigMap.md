# ConfigMap

**“쿠버네티스에서 기밀이 아닌 설정값을 저장하는 객체 (API 오브젝트)”**

- DB URL, 로그 레벨, 애플리케이션 모드 같은 설정 값 저장
- Pod가 그 값을 **환경변수** 등의 방식으로 사용 가능

→ 핵심 개념: 컨테이너 이미지는 코드만 두고, 설정은 ConfigMap으로 분리해서 관리한다. 

# 1. ConfigMap의 필요성

### 1) 코드와 구성을 분리하기 위함

Docker 이미지에 설정 값을 넣으면:

- 환경마다 다른 설정 때문에 이미지가 여러 개 필요
- 설정 값을 수정할 때마다 이미지 재빌드 필요
- 실수로 민감한 값이 이미지에 포함될 위험성 존재
    
    → ConfigMap에 분리하여 해결한다.
    

### 2) Pod에서 외부 환경 설정 값을 쉽게 제어

- 애플리케이션은 **환경 변수로 설정을 읽도록** 만들고
- 다양한 환경에 따라 ConfigMap만 바꿔서 배포할 수 있다.

### 3) Service, Deployment 등 YAML에서 구성 가능

ConfigMap은 쿠버네티스 오브젝트라 YAML로 관리한다.

# 2. ConfigMap의 구조

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  key: value
binaryData:
  file.bin: <base64 인코딩된 값>
immutable: true|false
```

## 🔍 필드 설명

- data
    - 문자열 기반의 “key-value”로 구성
    - 대부분의 ConfigMap이 이걸 사용
    - UTF-8 텍스트여야 한다.
- binaryData
    - 파일이나 바이너리를 base64 로 인코딩해 저장할 때 사용
    - ex. 이미지 파일, 인증서 파일, 바이너리 설정 파일
- immutable
    - `true`로 설정하면 “절대 수정 불가”
    - 실수로 값 바뀌는 사고 방지 + 성능 최적화

# 3. ConfigMap을 생성하는 여러 방식

### 1) 리터럴 값으로 생성

```lua
kubectl create configmap my-config \   
  --from-literal=LOG_LEVEL=debug \     # --from-literal: key=value를 바로 넣는 방식 
  --from-literal=MODE=production
```

**해석:** `my-config` 라는 이름의 ConfigMap을 생성하는데,

          LOG_LEVEL=debug, MODE=production 값을 직접(key=value) 넣어서 만든다. 

→ 여러 key를 한 명령에 넣을 수 있다. 즉 만들어지는 ConfigMap은 아래 YAML과 동일하다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  LOG_LEVEL: "debug"
  MODE: "production"
```

### 2) 파일로부터 생성

파일이 있을 때: `config.txt`

```lua
max_size=100
enable_feature=true
```

명령:

```lua
kubectl create configmap file-config --from-file=config.txt
```

→ key 이름은 파일 이름(`config.txt`)이 되고 value는 파일 전체 내용이 된다. 

### 3) 디렉터리 전체로 샐성

```lua
kubectl create configmap dir-config --from-file=./config-dir/
```

→ 디렉터리 내 모든 파일이 각각 “key-value” 로 들어감

### 4) YAML 직접 작성 (가장 권장)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config
data:
  player_initial_lives: "3"
  ui_properties: |
    color=blue
    font=Arial
```

→ 이 파일을 `kubectl apply -f` 로 적용한다. 

# 4. ConfigMap을 Pod에서 사용하는 방법

### 1) 환경 변수로 주입 (env)

Pod에서 특정 key 하나를 환경 변수로 매핑:

```yaml
env:
  - name: GAME_MODE
    valueFrom:
      configMapKeyRef:
        name: game-config
        key: player_initial_lives
```

→ 컨테이너에서 `echo $GAME_MODE` 하면 값이 출력된다.

### 2) 환경 변수 여러 개 가져오기 (envFrom)

ConfigMap 전체를 환경 변수로 펼쳐 쓰기:

```yaml
envFrom:
  - configMapRef:
      name: game-config
```

→ ConfigMap의 key들이 “환경 변수 이름이 되고, key의 value가 실제 값이 된다. 

### 3) Volume으로 파일 마운트하기

ConfigMap의 key들을 파일로 변환해서 Pod 내부에 디렉토리로 붙인다.

```yaml
volumes:
  - name: config-volume
    configMap:
      name: game-config
containers:
  - name: app
    volumeMounts:
      - name: config-volume
        mountPath: /etc/config
```

결과:

```arduino
/etc/config/player_initial_lives
/etc/config/ui_properties
```

→ 각 key가 파일명, value가 파일 내용이 된다. 

### 4) command/args에 주입

환경 변수를 command 인자로 넘기는 방식:

```arduino
command: ["sh", "-c", "run-app --mode=$GAME_MODE"]
```

→ 이때, `GAME_MODE` 는 ConfigMap으로부터 가져온 env 값이다. 

# 5. ConfigMap 변경이 Pod에 반영되는 방식

- **환경변수 방식:** Pod를 재시작해야 반영된다
    - 환경 변수는 컨테이너 시작 시점에만 읽힌다.
- **Volume 마운트 방식:** 자동으로 변경을 감지하여 파일이 업데이트

# 6. ConfigMap의 제한사항

1. 전체 크기 제한 **1 MiB:**  큰 파일은 따로 스토리지에 두고 경로만 ConfigMap으로 관리한다.
2. ConfigMap은 비밀을 저장하면 안됨: 민감 정보는 절대 삽입 ❌ (민감 정보는 secret 사용)
3. 모든 Pod는 같은 네임스페이스 내에서만 ConfigMap 참조 가능
4. key 이름 규칙 존재: 영문자/숫자/특수문자(`-`, `_`, `.`) 허용, 파일명으로 쓰이기 때문에 제약 ✅

# 7. immutable ConfigMap

```yaml
immutable: true
```

→ 이 옵션이 있으면 ConfigMap이 **변경불가**합니다.

- 장점:
    - 실수로 변경 방지
    - kubelet이 변경 감시를 하지 않아 성능 향상
    - 대규모 클러스터에서 성능 최적화에 도움
- 단점: 값을 바꾸려면 새 ConfigMap을 만들고 Pod에서 참조 이름을 변경해야 한다.

# 8. ConfigMap 사용팁

- 애플리케이션이 환경 변수를 읽도록 설계하면 관리가 쉬워짐
- **ConfigMap은 Git으로 버전 관리** → YAML로 저장해두면 변경 이력 추적 + 롤백이 쉬움
- **환경마다 다른 ConfigMap을 사용 →** dev-config / staging-config / prod-config  ← 이런 식으로 분리
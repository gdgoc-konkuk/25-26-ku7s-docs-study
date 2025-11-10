쿠버네티스에서 pod가 사용할 수 있는 프로세스 ID의 수와 운영체제와 데몬이 사용하는 각 노드에 할당 가능한 프로세스 ID의 수를 미리 정의 가능

> **시스템 데몬**: 운영체제에서 항상 백그라운드로 돌면서 시스템 기능을 담당하는 프로그램

- 왜?
  특정 파드가 너무 많은 프로세스를 생성하면 노드가 더 이상 프로세스를 만들지 못하게 되고 결국 노드가 멈춤
  → PID가 소진되지 않도록 제한하는 방법이 필요
- 특징
  - PID는 overcommit 가능
  - 파드별 PID 제한은 파드별로 보호를 해주지만, 노드 전체 PID 한도에 대한 보호는 없음
  - Pod말고 운영체제/데몬용으로 쓸 PID를 별도로 둠
- Example
  pid_max=262144, pod가 250개 미만으로 예상되면 파드 당 1000 PID를 할당

### 노드 PID 제한

`--system-reserved`, `--kube-reserved` 명령줄에서 `pid=<number>`를 통해서 PID를 예약 가능

### 파드 PID 제한

Pod에서 실행되는 프로세스 제한을 위해 노드 수준에서 진행 → 노드의 모든 파드에 PID 상한을 적용하는 방식

- 커맨드라인 매개변수 `--pod-max-pids`를 지정
- kubelet의 구성파일의 PodPidsLimit

### PID 기반 축출(eviction)

Pod가 오작동하고 비정상적인 양의 리소스를 사용할 때 파드를 종료하게끔 구성 → 축출

- 축출 신호
  일정 시간마다 kubelet이 상태를 확인하고 갱신
  - memory.available: 노드의 남은 메모리가 적을 때
  - nodefs.available: 디스크 공간 부족
  - pid.available: 사용 가능한 PID가 거의 다 찼을 때
- 정책
  - Soft Eviction Policy: 일정 시간 지속될 때 pod 축출
  - Hard Eviction Policy: 즉시 pod 축출

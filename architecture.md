
## Overview 구성(안)

![스케치](https://github.com/ywchae1209/DBMS_CDC/blob/master/architecture02.png "스케치")

## 설계방향

### 1. Multi-Thread 기반의 단일실행파일
* **운영관리/배포 등에서 version-conflict여지가 아예 없도록 (전체기능을 포함하는) 단일실행파일로 하자.**

> * 극단적으로 높은 수준의 fault-tolerant 필요 (--> 그렇게 만들면 되니 그렇게 하자.)
> * 최적 Runtime 자원관리 필요 (--> 그렇게 만들면 되니 그렇게 하자.)

### 2. 일원화된 관리/연동 Interface
* **gRPC(HTTP/2 기반) 서비스를 통해 일원화된 관리가능하도록 하자.**

> * 경량 gRPC(google Remote Procedure Call) 서버 내장.
> * gRPC 프로토콜은 gRPC IDL(Interface Definition Language)로 누구든 알 수 있도록 정의.
> * gRPC의 테스트도구를 사용하여 검증가능하도록. (**테스트 용이성**)
> * 불가능하거나 억지스럽지 않은 모든 관리/모니터링 기능을 gRPC API로 일원화.
> * **배포시 필요한 설정 최소화**( ~> 비밀(?)스런 설정파일관리는 아예 없도록)

### 3. 자기완결적 기능 구성
* **모든 동작시나리오를 완결적으로 지원하자.**

> * 완결적 동작으로 **CDC capture cluster구성이 가능한 분산 구조**를 지원하도록 하자.
> * 배포, 설치, 동작에 필요한 **제약사항 최소화**

> * Note
```
초고성능 Que기능을 위해서 MMF기능을 사용함.
(Memory-Mapped-File; 초고속 FileI/O를 위해 Memory와 File을 직접 Mapping하는 OS 기능)

MMF는 유일하게 OS 의존성 제약이 있는 부분으로 이식성 제한이 있는지 확인필요함.
( Modern OS는 관계없고, 오래된 AIX 같은 OS)
~> 만약, 제약 있으면 traditional File-API기반으로 Que를 추가로 만들어야 함.
```

### 4. Reactive Architecutre First
* **Reactive Architecutre First로 하자.**

> * Reactive Architecture(반응형 구조)  
>   ~ 가장 간단한 요약: 필요할 때, 필요한만큼만 자원을 사용하는 설계방식.  
>   [참고]
> * [[https://www.reactivemanifesto.org/]]
> * [[https://techbuzzonline.com/reactive-architecture-patterns-guide/]]

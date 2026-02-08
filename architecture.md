
## Overview 구성(안)

![스케치](https://github.com/ywchae1209/DBMS_CDC/blob/master/architecture02.png "스케치")

### 용어
* 역할에 따른 Node구분

| Node        | 물리적(또는 논리적)으로 OS가 설치된 Machine |
|-------------|---------------------------------------------|
| Source Node | 데이터의 원천 노드                          |
| Pump Node*  | Source Node에서 추출하는 노드               |
| Relay Node* | 다른 노드의 요청에 따라 Msg제공하는 노드    |
| Sink Node*  | 데이터가 최종 소비되는 노드                 |
| Target Node | 데이터가 최종 적용되는 노드                 |

~ DeltaFlow의 설치여부와 할당한 동작에 따라 Node의 역할이 달라진다.

* DeltaFlow sub-module

|            | DeltaFlow의 주요 구성요소                                                     |
|------------|-------------------------------------------------------------------------------|
| Pump       | 원천 또는 supply-node에서 데이터를 가져와서(pull) Que에 쌓는(append) Thread들 |
| Supplier   | Que의 메시지를 (외부에) 제공하는 Thread들                                     |
| Applier    | Que의 메지시를 소비하는 Thread들                                              |
| Que        | 원천메시지의 저장소( file-Que)                                                |
| Controller | DeltaFlow 서버 (gRPC server)                                                  |


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

### 4. Reactive Architecutre First
* **Reactive Architecutre First로 하자.**

> * Reactive Architecture(반응형 구조)  
>   ~ 가장 간단한 요약: 필요할 때, 필요한만큼만 자원을 사용하는 설계방식.  
>   [참고]
> * [[https://www.reactivemanifesto.org/]]
> * [[https://techbuzzonline.com/reactive-architecture-patterns-guide/]]

### note

* 유일하게 OS 의존성 제약이 있는 부분
```
초고성능 Que기능을 위해서 MMF기능을 사용함.
(Memory-Mapped-File; 초고속 FileI/O를 위해 Memory와 File을 직접 Mapping하는 OS 기능)

MMF는 유일하게 OS 의존성 제약이 있는 부분으로 이식성 제한이 있는지 확인필요함.
( Window, Linux, MacOS등과 POSIX 호환 Unix-Like OS 대부분 지원)

~> 만약, 제약 있으면 traditional File-API기반으로 Que를 추가로 만들어야 함.
(  보통의 방식이긴 하나, 성능상 많은 불이익 발생.)
```

* MMF 지원 여부에 대한 최소한의 질문
```
Shared Memory 기반의 mmap()이 완벽히 작동하는가? (POSIX)
64-bit Atomic 연산을 OS 수준에서 보장하는가?
최소한 Java 8 이상의 JVM이 해당 OS용으로 빌드되어 있는가?

>>> IBM의 AIX
* 7.x이후는 지원가능성 높음 (2010년 Release)
* 6.x 버전은 지원가능할 수도 있으나, 사실상 안된다고 보는 게 맞음.
```

* 참고

jar(java 실행파일)도 jar-file버전에 따라 실행가능한 JVM 최소버전이 다름

| 빌드 JDK | 생성되는 class file version | 실행 가능한 최소 JVM | 출시 연도 |
|:--------:|:---------------------------:|:--------------------:|:---------:|
| *JDK 8    | 52                          | JDK 8                | 2014      |
| JDK 9    | 53                          | JDK 9                | 2017      |
| JDK 10   | 54                          | JDK 10               | 2018      |
| JDK 11   | 55                          | JDK 11               | 2018      |
| JDK 12   | 56                          | JDK 12               | 2019      |
| JDK 13   | 57                          | JDK 13               | 2019      |
| JDK 14   | 58                          | JDK 14               | 2020      |
| JDK 15   | 59                          | JDK 15               | 2020      |
| JDK 16   | 60                          | JDK 16               | 2021      |
| JDK 17   | 61                          | JDK 17               | 2021      |
| JDK 18   | 62                          | JDK 18               | 2022      |
| JDK 19   | 63                          | JDK 19               | 2022      |
| JDK 20   | 64                          | JDK 20               | 2023      |
| *JDK 21   | 65                          | JDK 21              | 2023      |
| JDK 22   | 66                          | JDK 22               | 2024      |
| JDK 23   | 67                          | JDK 23               | 2024      |


* JDK 8은 Modern Java의 시작버전이며, 많은 Java 프로그램에서 최소한 JDK version임.
* JDK 버전에 따라 Memory관리등의 최적화 수준 차이가 크며,
* 시스템 자원 성능을 최대한 쥐어짜내야 하니, Virtual Thread가 포함된 **JDK 21**이 권장 버전임.

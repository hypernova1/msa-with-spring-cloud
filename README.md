# 스프링 클라우드를 활용한 MSA 기초

#### 학습 자료

[Spring Cloud를 활용한 MSA 기초](https://www.youtube.com/watch?v=D6drzNZWs-Y&list=PL9mhQYIlKEhdtYdxxZ6hZeb0va2Gm17A5)

### 목표

* 모놀리식이 나온 배경과 아키텍쳐를 이해한다
* Cloud Native한 MSA를 이해한다
* Netflix OSS, Spring Cloud를 통해 MSA를 구축한다.

### 목차
* [기존 모놀리식 방식의 개발](#기존-모놀리식-방식의-개발)
* [MSA(Microservice Architecture)](#MSAMicroservice-Architecture)
* [Cloud Native](#Cloud-Native)
* [Spring Cloud](#Spring-Cloud)
* [Server Side LoadBalancer](#Server-Side-LoadBalancer)
* [Hystrix](#HystrixCircuit-Breaker)
* [Ribbon](#RibbonClient-Load-Balancer)
* [Eureka](#EurekaService-Discovery)
* [Feign](#FeignDeclarative-Http-Client)

## 기존 모놀리식 방식의 개발

신규 프로젝트가 시작되면 개발자가 개발한 코드는 하나의 톰캣에서 실행되고 프로그램의 상태는 하나의 DB에 저장된다. 그리고 개발자들은 협업을 위하여 형상관리 툴(svn, git 등)을 사용한다. 하지만 현재는 상용 배포시 톰캣 하나, 데이터베이스 하나를 이용하기 때문에 무중단 배포가 불가능하다.  

그래서 이제는 톰캣 두 개를 두고 앞에 로드밸런서(L4 switch, L7)를 두어 구성한다(High Avaliability). 하나의 톰캣을 배포한 뒤 완료되면 나머지 톰캣을 배포한다(무중단 배포). 그런데 서비스가 잘되서 동시 접속자 수가 1000명 이상이 되었다고 하자. 그러면 두 개의 톰캣으로는 감당이 되지 않아 여러 개의 톰캣을 추가해야 할 것이다(LB 설정 변경). 그리고 배포방식도 변경될 것이다(Jenkins, Ansible, Chef 등). 또한 조직 개편을 하여 상품 팀과 주문팀으로 분리가 되면 다음과 같은 문제 발생할 수 있다.

1. branch merge 시 충돌 발생
2. QA를 어디까지 해야하나? - 정기배포일
3. 팀마다 각자 다른 일정, 배포 이슈
4. etc..

위와 같은 문제를 해결하기 위하여 도메인을 2개로 분리하여 주문팀과 상품팀의 서버를 나눈다. 서로 공통되는 로직은 share.jar로 따로 빼낸다. 하지만 실제로 대부분의 작업은 share.jar에서 이루어지기 때문에 문제는 해결되지 않는다.


회사가 점점 커져감에 따라 개발자도 늘어나고 조직도 커져가는데, 사용되는 데이터베이스는 하나이고 공통 코드(share.jar)는 한 군데 모여있다. 또한 복사 붙여넣기로 오는 중복과 복잡도가 증가한다.

>  콘웨이의 법칙
> * 광범위하게 정의하면 모든 조직은 조직의 의사소통 구조와 똑같은 구조를 갖는 시스템을 설계한다.
> * 콘웨이 법칙은 조직도에 초점을 두지 않고, 실질적인 소통 관계에 관심을 둔다. 너무 많은 통신 관계를 갖는 것은 프로젝트에 대한 진정한 위험이 된다.

### 모놀리식 아키텍쳐의 장단점

* 장점
  * 개발, 배포가 단순하다.
  * Scale-out이 단순하다(서버를 복사하면 됨)
    * 하지만 DB 성능으로 인한 한계가 있다.
* 단점
  * 무겁다
  * 어플리케이션 시작이 오래 걸린다.
  * 기술 스택을 바꾸기 어렵다.
  * 높은 결합도
  * 코드베이스의 책임 한계와 소유권이 불투명
    *  프로젝트 전체를 파악하는 사람이 없고 그에 따라 프로젝트의 경계가 불분명해진다.


### 앞으로의 방향을 결정해야 한다.

1. 새로운 언어, 깔끔한 코드로 전면개편
2. MSA 플랫폼을 구축하여 기존 레거시를 고시시킴


## MSA(Microservice Architecture)

MSA란 시스템을 여러개의 독립된 서비스로 나눈 후, 이 서비스를 조합함으로서 기능을 제공하는 아키텍쳐 디자인 패턴이다. 현재까지 공식적인 정의는 없지만 다음과 같은 공감대가 존재한다.

* 각 서비스 간 네트워크를 통해(보통 http) 통신
* 각 서비스는 독립된 배포 단위를 가진다.
* 각 서비스는 쉽게 교체 가능하다.
* 각 서비스는 기능 중심으로 구성된다.
  * ex) 프론트 엔드, 추천, 정산, 상품 등
* 각 서비스에 적합한 프로그래밍 언어, 데이터베이스, 환경으로 만들어진다.
* 서비스의 크기가 작고, 상황에 따라 경계를 정하고, 자율적으로 개발되고, 독립적으로 배포되고, 분산되고, 자동화된 프로세스로 구축되고 배포된다.
* 마이크로서비스는 한 팀에 의해 개발할 수 있는 크기가 상한선이다. 3~9명의 사람들이 개발을 할 수 없을 정도로 커지면 안 된다.

#### 모놀리식 vs 마이크로서비스

![monolithic vs microservice](https://blogs.bmc.com/wp-content/uploads/2018/10/microservices-vs-monolithic-1024x544.png)


#### 기존 모놀리식 방식의 문제를 MSA로 변경

공통 로직(share.jar)를 제거하고 각각의 마이크로서비스 별로 를 가지고 api를 통해서만 서로를 호출한다.

## Cloud Native

클라우드 네이티브의 핵심은 애플리케이션을 어떻게 만들고 배포하는지에 있으며 위치는 중요하지 않다.

> 클라우드 서비스를 활용한다는 것은 컨테이너와 같이 민첩하고 확장 가능한 구성 요소를 사용해서 재사용 가능한 개별적인 기능을 제공하는 것을 의미한다. 이러한 기능은 멀티 클라우드와 같은 여러 기술 경계 간에 매끄럽게 통합되므로 제공 팀이 반복 가능한 자동화와 오케스트레이션을 사용해서 빠르게 작업과정을 반복할 수 있다. - 앤디 맨

### 클라우드 네이티브의 다섯가지 요소
* 신축성(Resiliency): 서비스가 중단이 되어도 (인스턴스가 종료되어도) 전체적인 장애로 퍼지지 않고 빠르게 복구
* 민첩성(Agility): 빠르게 배포하고, 독립적으로 빠르게 운영
* 확장 가능성(Scalable): 수평적으로 확장 가능하게
* 자동화(Automation): 이러한 과정들을 자동화하여 운영에 드는 수고를 줄임
* 무상태(State-less): 각 서비스의 상태는 무상태


### DevOps

#### 전통적 모델
* 개발과 운영 조직의 분리
* 다른 쪽으로 일을 알아서 처리하라고 맡김
#### DevOps
* You run it, you build it. 만들면 운영까지 - 베르너 보겔스, 아마존 CTO
* 개별 팀은 프로젝트 그룹이 아닌, 제품 그룹에 소속
* 운영과 제품 관리 모두가 포함되는 조직적 구조. 제품 팀은 소프트웨어를 만들고 운영하는 데 필요한 모든 것을 보유

### [Twelve-Factors](https://12factor.net/ko/)

* Heroku 클라우드 플랫폼 창시자들이 정립한 애플리케이션 개발 원칙 중 유익한 것을 모아 정리한 것
* 탄력적(elastic)이고 이식성(portability) 있는 배포를 위한 Best Practice

#### 핵심 사상
* 선언적(<->명령적) 형식으로 설정을 자동화해서 프로젝트에 새로 참여하는 동료가 적응하는 데 필요한 시간과 비용을 최소화한다.
* 운영체제에 구애받지 않는 투명한 계약을 통해 다양한 실행 환경에서 작동할 수 있도록 이식성을 극대화한다.
* 현대적인 클라우드 플랫폼 기반 개발을 통해서 서버와 시스템 관리에 대한 부담을 줄인다.
* 개발과 운영의 간극을 최소화해서 지속적인 배포(Continuous Deployment)를 가능하게 하고 애자일성을 최대화한다.
* 도구, 아키텍쳐, 개발 관행을 크게 바꾸지 않아도 서비스 규모의 수직적 확장이 가능하다.

### 12가지 제약 조건

#### 1. 코드베이스
  * 버전 관리되는 하나의 코드베이스가 여러 번 배포된다.
  * 코드베이스와 앱 사이에는 항상 1대1 관계가 성립된다.
    * 하나의 프로젝트에서 여러 서비스를 배포하지 않는다.

#### 2. 종속성
* 애플리케이션의 의존관계는 명시적으로 선언되어야 한다.
* 모든 의존 라이브러리는 아파치 메이븐, 그레들 등의 의존관계 관리 도구를 써서 라이브러리 저장소에서 내려받을 수 있어야 한다.

#### 3. 설정
* 설정 정보는 코드에 있으면 안 되고 실행 환경(환경 변수)에 저장해야 한다.
  * 설정 정보는 애플리케이션 코드와 엄격하기 분리
  * ex) 데이터베이스 정보, cdn, 외부 서비스 인증, 호스트 이름 등

#### 4. 백엔드(지원) 서비스
* 지원 서비스(Backing Service)는 필요에 따라 추가되는 자원으로 취급한다.
  * 지원 서버비스는 애플리케이션의 자원으로 간주한다.
  * ex) 데이터베이스, API 기반 RESTFul 웹 서비스, SMTP 서버, FTP 서버 등
  * ex) 테스트 환경에서 사용하던 임베디드 SQL을 스테이징 환경에서 MySQL로 교체할 수 있어야 한다.

#### 5. 빌드, 릴리즈, 실행
* 철저하게 분리된 빌드와 실행 단계
* 코드베이스는 3단계를 거쳐 (개발용이 아닌) 배포로 변환된다.
  * 빌드 단계: 소스 코드를 가져와 컴파일 후 하나의 패키지를 만든다.
  * 릴리즈 단계: 빌드에 환경설정 정보를 조합한다. 릴리즈 버전은 실행 환경에서 운영될 수 있는 준비가 완료되어 있다.
    * ex) jenkins에서 아티팩트를 만드는 단계
  * 실행 단계: 보통 런타임이라 불리며, 릴리즈 버전 중 하나를 선택해 실행 환경 위에서 애플리케이션을 실행한다.

#### 6. 프로세스
* 애플리케이션을 하나 혹은 여러개의 무상태(stateless) 프로세스로 실행한다.

#### 7. 포트 바인딩
  * 서비스는 포트에 연결해서 외부에 공개한다.
  * 실행 환경에 웹 서버를 따로 추가해줄 필요 없이 스스로 웹 서버를 포함하고 있어서 완전히 자기 완비적(self-contained)이다.

#### 8. 동시성(Concurrency)
* 프로세스 모델을 통해 수평적으로 확장한다.
* 애플리케이션은 필요할 때 마다 프로세스나 스레드를 수평적으로 확장해서 병렬로 실행할 수 있어야 한다.
* 장시간 소요되는 데이터 프로세싱 작업은 스레드풀에 할당해서 스레드 실행기를 통해 수행되어야 한다
* 예를 들어, HTTP 요청은 서블릿 스레드가 처리하고, 시간이 오래 걸리는 작업은 워커 스레드가 처리해야 한다.

#### 9. 처분성(Disposability)
* 빠른 시작과 그레이스풀 셧다운(Graceful Shutdown)을 통한 안정성 극대화
* 애플리케이션은 프로세스 실행 중에 언제든지 중지될 수 있고, 중지될 때 처리되어야 하는 작업을 모두 수행한 다음에 종료될 수 있다.
* 가능한 짧은 시간 내에 시작되어야 한다.

#### 10. 개발과 운영의 일치
* development, staging, production 환경을 최대한 비슷하게 유지
* 개발 환경과 운영 환경을 가능한 동일하게 유지하는 짝맞춤(parity)를 통해 분기(divergence)를 예방할 수 있어야 한다.
* 유념해야 할 세 가지 차이
  * 시간 차이: 개발자는 변경 사항을 운영 환경에 빨리 배포해야 한다.
  * 개인 차이: 코드 변경을 맡은 개발자는 운영 환경으로의 배포 작업까지 할 수 있어야 하고, 모니터링도 할 수 있어야 한다.
  * 도구 차이: 각 실행 환경에서 사용된 기술이나 프레임워크는 동일하게 구성되어야 한다.

#### 11. 로그
* 로그는 이벤트 스트림으로 취급한다
* 로그는 stdout에 남긴다.
* 애플리케이션은 로그 파일 저장에 관여하지 않아야 한다.
* 로그 집계와 저장은 애플리케이션이 아니라 실행 환경에 의해 처리되어야 한다.

#### 12. Admin 프로세스
* admin/maintenance 작업을 일회성 프로세스로 실행
* 실행되는 프로세스와 동일한 환경에서 실행
* admin 코드는 애플리케이션 코드와 함께 배포되어야 한다.

### HTTP, REST API
보통 MSA 구축할 때 REST를 많이 사용한다.

* HTTP
  * 클라이언트의 상태를 갖지 않음(stateless)
  * 각 요청은 자기 완비적(self-contained)
* REST API
  * 원격 자원과 엔티티를 다루는데 초점
  * 동사 대신 명사를, 행위 대신 엔티티에 집중
  * REST는 기술 표준이 아닌 아키텍쳐 제약사항
  * 상태가 없고 요청이 자기 완비적이기 때문에 서비스도 수평적으로 쉽게 확장할 수 있다.

### Netflix OSS

* 50개 이상의 사내 프로젝트를 오픈 소스로 공개
* 플랫폼(AWS) 안의 여러 컴포넌트와 자동화 도구를 사용하면서 파악한 패턴과 해결 방법을 블로그, 오픈 소스로 공개

## Spring Cloud

#### 모놀리식에서의 의존성 호출
* 모놀리식에서의 의존성 호출은 100% 신뢰할 수 있다.

#### Failure as a First Class Citizen
* 분산 시스템, 특히 클라우드 환경에서는 실패는 일반적인 표준이다.
* 모놀리식엔 없던 장애 유형
* 한 서비스의 가동률(uptime) 최대 99.99%
  * 99.99^30 = 99.7% uptime
  * 10억 요청 중 0.3% 실패 = 300만 요청이 실패
  * 모든 서비스 들이 이상적인 uptime을 갖고 있어도 매 달마다 2시간 이상의 downtime이 발생

## Hystrix(Circuit Breaker)

* 톰캣의 기본 maxThread는 200개이다.
* RestTemplate timeout의 기본 설정은 무한대이다.
  * 한 번의 요청이 끝나기 전에는 스레드가 계속 요청을 붙잡고 있는다.
  * 만약 톰캣 하나가 네트워크 장애가 발생하면 하면 무한정 대기를 하게 되는데, 200개가 꽉 차면 서버는 다운되게 된다.

### Hystrix 적용하기

~~~java
@HysterixCommand
public String anyMethodWithExternalDependency() {
  URI uri = URI.create("http://172.32.1.22:8090/recommended");
  String result = this.restTemplate.getForObject(uri, String.class);
  return result;
}
~~~
위의 메소드를 호출하면 HystrixCommand가 Intercept하여 대신 실행한다. 실행된 결과의 성공/실패(Exception) 여부를 기록하고 통계를 낸다. 실행 결과 통계에 따라 Circuit Open 여부를 판단하고 필요한 조치를 취한다.

#### Circuit Open이란
  * Circuit이 오픈된 Method는 주어진 시간동안 호출이 제한되며, 즉시 에러를 반환한다.
  * 특정 메소드에서 지연이(주로 외부 연동에서의 지연) 시스템 전체의 Resource(Thread, Memory)를 모두 소모하여 시스템 전체의 장애를 유발한다.
  * 특정 외부 시스템에서 계속 에러를 발생 시킨다면, 지속적인 호출이 에러 상황을 더욱 악화시킨다.
  * 장애를 유발하는 (외부) 시스템에 대한 연동을 조기에 차단시킴으로서 시스템을 보호한다.
  * 기본 설정
    * 10초동안 20개 이상의 호출이 발생했을 때, 50% 이상의 호출에서 에러가 발생하면 Circuit Open

Circuit이 오픈된 경우 Fallback이 호출된다. Fallback 메소드는 Circuit이 오픈된 경우, 혹은 Exception이 발생한 경우에 대신 호출될 메소드이며, 장애 발생시 Exception대신 응답할 Default 구현을 넣는다.

~~~java
@HistrixCommand(commandKey = "ExtDep1", fallbackMethod = "recommendFallback")
public String anyMethodWithExternalDependency1() {
  URI uri = URI.create("http://172.32.1.22:8090/recommended");
  String result = this.restTemplate.getForObject(uri, String.class);
  return result;
}

public String recommendFallback() {
  return "No recommend available";
}
~~~

#### 오랫동안 응답이 없는 메소드 처리방법 - Timeout
~~~java
@HystrixCommand(commandKey = "ExtDep1", fallbackMethod= "recommendFallback",
  commandProperties = {
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMiliseconds", value = "500")

})
public String anyMethodWithExternalDependency1() {
  URI uri = URI.create("http://172.32.1.22:8090/recommended");
  String result = this.restTemplate.getForObject(uri, String.class);
  return result;
}

public String recommendFallback() {
  return "No recommend available";
}
~~~

* 설정하지 않으면 default는 1000ms
* 설정 시간동안 메소드가 끝나지 않으면 return/exception
* Hystrix 메소드를 실제 실행중인 Thread의 interrupt를 호출하고, 자신은 즉시 HystrixException을 발생시킨다.
  * 이 경우에도 Fallback이 있다면 Fallback 실행


### 실습 - Hystrix 사용하기

#### 배경
* Display 서비스는 외부 Server인 Product API와 연동되어 있음
* Product API에 장애가 나더라도 Display의 다른 서비스는 이상없이 작동해야 한다.
* Product API에 응답 오류가 발생한 경우, Default값을 넣어준다.
#### 결정 내용
* Display - Product 연동 구간에 Circuit Breaker를 적용

#### 사용 순서
1. [display] gradle.build에 의존성 추가
  ~~~
  compile('org.springframework.cloud:spring-cloud-starter-netflix-hystrix')
  ~~~
2. `DisplayApplication`에 `@EnableCircuitBreaker` 추가
~~~java
@EnableCircuitBreaker
@SpringBootApplication
public class DisplayApplication {
~~~
3. [display] `DisplayRemoteServiceImpl`에 `@HystrixCommand` 추가
~~~java
@Override
@HystrixCommand
public String getProductInfo(String productId) {
  return restTemplate.getForObject(URL + productId, String.class);
}
~~~

4. [product] `ProductController`에서 항상 Exception을 던지게 수정(장애 상황 흉내)
~~~java
@GetMapping("/{productId}")
    public String getProduct(@PathVariable String productId) {
        throw new RuntimeException("I/O Exception");
        //return "[product id = " + productId + " at " + System.currentTimeMillis() + "]";
    }
~~~

5. [display] `ProductRemoteServiceImpl`에 FallbackMethod 작성
~~~java
@Override
@HystrixCommand(fallbackMethod = "getProductInfoFallback")
public String getProductInfo(String productId) {
  return restTemplate.getForObject(URL + productId, String.class);
}

public String getProductInfoFallback(String productId) {
  return "[This Product is sold out]";
}
~~~

6. 확인
~~~
http://localhost:8082/product/22222
http://localhost:8081/display/11111
~~~

* Histrix가 발생한 Exception을 잡아서 Fallback을 실행. Fallback 정의 여부와 상관없이 Circuit 오픈 여부 판단을 위한 에러 통계는 계산하고 있지만 아직 Circuit은 오픈된 상태가 아님

7. Fallback 원인 출력하기
* Fallback메소드 파라미터에 `Throwable` 추가 및 출력
~~~java
public String getProductInfoFallback(String productId, Throwable t) {
  System.out.println("t=" + t);
  return "[This Product is sold out]";
}
~~~
* 결과
~~~
t=org.springframework.web.client.HttpServerErrorException: 500 null
~~~

### Hystrix로 Timeout 처리하기
`@HystrixCommand`로 표시된 메소드는 지정된 시간 안에 반환되지 않으면 자동으로 Exception이 발생 (기본 설정: 1000ms)

1. 2초가 걸리는 작업 테스트
~~~java
@GetMapping("/{productId}")
public String getProduct(@PathVariable String productId) {
  try {
    Thread.sleep(2000);
  } catch (InterruptedException e) {
    e.printStackTrace();
  }
//throw new RuntimeException("I/O Exception");
  System.out.println("Called product id: " + productId);
  return "[product id = " + productId + " at " + System.currentTimeMillis() + "]";
}
~~~

2. 확인
* 호출하는 쪽(display)에서는 1초가 지났기 때문에 exception을 띄우지만, 호출을 받는 쪽에선 정상적으로 `System.out.println`이 실행 된다.

3. [display] application.yml을 수정하여 Hystrix Timeout 시간 조정하기
~~~yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
~~~

4. 확인
* 정상적으로 호출이 된다.
~~~
[display id = 11111 at 1593527795635 [product id = 1111 at 1593527795625] ]
~~~

5. 정리
* Hystix를 통해 실행되는 모든 메소드는 정해진 응답시간 내에 반환되어야 한다.
* 그렇지 못한 경우, Exception이 발생하며, Fallback이 정의된 경우 Fallback이 수행된다.
* Timeout은 조절 가능하다.
* 모든 외부 연동은 최대 응답 시간을 가정할 수 있어야 한다.
* 여러 연동을 사용하는 경우 최대 응답 시간을 직접 Control하는 것은 불가능하다.
  * 다양한 timeout, 다양한 지연 등)

### Hystrix Circuit Open 테스트

1. [display] application.yml에 Hystrix 프로퍼티 추가
~~~yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
      circuitBreaker:
        requestVolumeThreshold: 1 # default: 20
        errorThresholdPercentage: 50 # default: 50
~~~
* 1번의 요청이 실패하면 Circuit Open이 발생하도록 설정

* `ProductController` 수정
~~~java
    @GetMapping("/{productId}")
    public String getProduct(@PathVariable String productId) {
//    try {
//      Thread.sleep(2000);
//    } catch (InterruptedException e) {
//      e.printStackTrace();
//    }
      throw new RuntimeException("I/O Exception");
//    System.out.println("Called product id: " + productId);
//    return "[product id = " + productId + " at " + System.currentTimeMillis() + "]";
    }
~~~

* 결과
~~~
t=java.lang.RuntimeException: Hystrix circuit short-circuited and is OPEN
~~~


## Server Side LoadBalancer
* 일반적인 L4 스위치 기반의 로드밸런싱
* 클라이언트는 L4의 주소만 알고 있음
* L4 스위치는 서버의 목록을 알고 있음(Server Side Load Balancing)
* H/W Server Side Load Balancer의 단점
  * 하드웨어가 필요
  * 서버 목록의 추가를 위해서는 설정이 필요(자동화가 어려움)
  * Load Balancing Schema가 한정적 (Round Robin, Sticky)
* Twelve Factors의 개발/운영 일치를 만족하기 어려움

## Ribbon(Client Load Balancer)
* 클라이언트(API Caller)에 탑재되는 소프트웨어 모듈
* 주어진 서버 목록에 대해서 로드밸런싱을 수행함
* Ribbon의 장점
  * 하드웨어가 필요없이 소프트웨어만으로 가능
  * 서버 목록의 동적 변경이 자유로움
  * Load Balancing Schema를 마음대로 구성 가능

### 실습 - RestTemplate에 Ribbon 적용

1. [product] `ProductController`를 정상적인 코드로 복원
~~~java
@GetMapping("/{productId}")
public String getProduct(@PathVariable String productId) {
//        throw new RuntimeException("I/O Exception");
    return "[product id = " + productId + " at " + System.currentTimeMillis() + "]";
}
~~~

2. [display] `build.gradle`에 의존성 추가
~~~
compile('org.springframework.cloud:spring-cloud-starter-netflix-ribbon')
~~~

3. [display] `DisplayApplication`의 `RestTemplate`빈에 `@LoadBalanced` 추가
~~~java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
~~~
* `@LoadBalanced`를 붙이면 내부적으로 `RestTemplate`에 `Interceptor`를 추가해준다. 그 후 `Interceptor`안에서 product를 `application.yml`에 있는 `listOfServers`의 값으로 변경해준다.

4. [display] `ProductRemoteServiceImpl`에서 주소를 제거하고 `product`로 변경
~~~java
//    public static String URL = "http://localhost:8082/products/";
public static String URL = "http://product/products/";
~~~

5. [display] `application.yml`에 ribbon 설정 넣기
~~~yaml
product:
  ribbon:
    listOfServers: localhost:8082
~~~

### Ribbon의 Retry 기능
1. [display] `build.gradle`에 의존성 추가
~~~
compile('org.springframework.retry:spring-retry:1.2.2.RELEASE')
~~~

2. [display] `application.yml`에서 서버 주소 추가 및 Retry 관련 속성 조정
~~~yaml
product:
  ribbon:
    listOfServers: localhost:8082, localhost:7777
    MaxAutoRetries: 0
    MaxAutoRetriesNextServer: 1
~~~

3. 확인
* localhost:7777은 없는 주소이므로 Exception이 발생하지만 Ribbon Retry로 항상 성공함
  * Round Robin Cliend Load Balancing & Retry

4. 주의
* Histrix로 Ribbon을 감싸서 호출한 상태이기 때문에 Retry를 시도하다가 HistrixTimeout이 발생하면, 즉시 에러가 반환된다.
* Retry를 끄거나 재시도 횟수를 0으로 만들어도 해당 서버로의 호출이 항상 동일한 비율로 실패하지는 않는다. 
  * 실패한 서버로의 호출은 특정 시간동안 Skip 되고 그 간격은 조정된다(BackOff)
* classpath에 Retry가 존재해야 한다.

5. 정리
* Ribbon은 여러 컴포넌트에 내장되어 있으며, 이를 통해 Client Side Load Balancing이 수행 가능하다.
* Ribbon은 매우 다양한 설정이 가능하다.
  * 서버 선택, 실패시 Skip 시간, Ping 체크
* Ribbon에는 Retry 기능이 내장되어있다.
* Eureka와 함께 사용될 때 강력하다.

### Ribbon 예제에서 서버 목록을 yml에 직접 넣었는데 자동화 하는 방법은?

## Eureka(Service Discovery)

* Service Registry
  * 서비스 탐색, 등록
  * 클라우드의 전화번호부
  * 단점: 침투적 방식 코드 변경

* DiscoveryClient
  * Spring Cloud는 서비스 레지스트리 사용 부분을 추상화함(interface)
  * Eureka, Consul, Zookeeper, etcd 등의 구현체가 존재

* Ribbon은 Eureka와 결합하여 사용될 수 있으며, 서버 목록을 자동으로 관리해준다.

### Eureka in Spring Cloud
* 서버 시작 시 Eureka Server(Registry)에 자동으로 자신의 상태를 등록(UP)
  * eureka.client.register-with-eureka: true(default)
* 주기적으로 Heart Beat로 Eureka Server에 자신이 살아있음을 알림
  * eureka.instance.lease-renewal-interval-in-seconds: 30(default)
* 서버 종료 시 Eureka Server에 자신의 상태 변경(DOWN) 혹은 자신의 목록 삭제
* Eureka 상에 등록된 이름은 'spring.application.name'

### Eureka Server(Registry) 만들기

1. [product, display] `build.gradle`에 의존성 추가
~~~
compile('org.springframework.cloud:spring-cloud-starter-netflix-eureka-client')
~~~

2. [product, display] 메인 클래스에 `@EnableEurekaClient`추가
~~~java
@EnableEurekaClient
@SpringBootApplication
public class ProductApplication {
//...
@EnableEurekaClient
@EnableCircuitBreaker
@SpringBootApplication
public class DisplayApplication {
//...
~~~

3. [product, display] `application.yml`에 설정 추가
~~~yaml
eureka:
  instance:
    prefer-ip-address: true
  client:
    service-url: 
      defaultZone: http://127.0.0.1:8761/eureka # default address
~~~

4. 확인
* http://localhost:8761/에 접속하여 두 인스턴스가 등록되어 있는지 확인


### RestTemplate에 Eureka 적용하기

#### 목적
Diplay -> Product 호출시에 Eureka를 적용하여 ip 주소를 코드나 설정 모두에서 제거

[display] `application.yml` 변경
~~~yaml
product:
  ribbon:
    # listOfServers: localhost:8082, localhost:7777 # 주석 처리
    MaxAutoRetries: 0
    MaxAutoRetriesNextServer: 1
~~~
* 서버 주소를 Eureka Server에서 가져오게 함

### Eureka 서버 주소 직접 명시
* `@EnableEurekaServer` / `@EnableEurekaClient`를 통하여 서버 구축, 클라이언트 Enable 가능
* `@EnableEurakeClient`를 붙인 Applicaion은 Eureka 서버로 부터 남의 주소를 가져오는 역할과, 자신의 주소를 등록하는 역할 둘 다 수행 가능
* Eureka Client가 Eureka Server에 자신을 등록할 때 `spring.application.name`이 이름으로 사용된다.

### 이중화 테스트

1. (Intellij 기준) Product 프로젝트를 톰캣 설정에서 복사 후 VM Options에 `-Dserver.port=8083`를 입력하여 Product를 추가로 하나 더 띄움(Eureka 서비스 레지스트리에 인스턴스(8083)가 추가로 등록됨)

## Feign(Declarative Http Client)
* Interface 선언을 통해 자동으로 Http Client를 생성
* RestTemplate는 concreate 클래스라 테스트하기 어렵다.
* 관심사의 분리
  * 서비스 관심 - 다른 리소스, 외부 서비스 호출과 리턴 값
  * 관심이 없는 것 들 - 어떤 URL, 어떻게 파싱할 것인가
* Spring Cloud에서 Open-Feign 기반으로 Wrapping 한 것이 Spring Cloud Feign
* url을 명시하게 되면 Robbin, Eureka, Hystrix를 사용하지 않는다.
  * 순수 Feign Client로서만 동작
  * url을 명시하지 않으면 Eureka에서 product 서버 목록을 조회해서 Ribbon을 통해 로드밸런싱을 하면서 HTTP 호출을 수행

### Feign 클라이언트 사용하기
#### 목적
* Display에서 Product 호출시 Feign을 사용하여 RestTemplate 대체

1. [display] `build.gradle`에 의존성 추가
~~~
compile('org.springframework.cloud:spring-cloud-starter-openfeign')
~~~

2. [display] `DisplayApplication`에 `@EnableFeignClients` 추가
~~~java
@EnableEurekaClient
@EnableCircuitBreaker
@EnableFeignClients
@SpringBootApplication
public class DisplayApplication {
~~~

3. [display] Feign용 Interface 추가
~~~java
@FeignClient(name = "product", url = "http://localhost:8082")
public interface FeignProductRemoteService {
    @RequestMapping(value = "/products/{productId}")
    String getProductInfo(@PathVariable String productId);
}
~~~
* `@FeignClient`는 내부적으로 Robbin으로 구현되어있기 때문에 url을 지우게 되면 Eureka를 사용하게 된다.

4. [display] `DisplayController` 변경
~~~java
 @GetMapping(path = "/{displayId}")
public String getDisplayDetail(@PathVariable String displayId) {
    String productInfo = getProductInfo();
//        String productInfo = productRemoteService.getProductInfo("1111");
    return String.format("[display id = %s at %s %s ]", displayId, System.currentTimeMillis(), productInfo);
}

private String getProductInfo() {
    return feignProductRemoteService.getProductInfo("12345");
}
~~~

### Feign + Hystrix
* [display] `application.yml` 설정 추가
~~~yaml
feign:
  hystrix:
    enabled: true
~~~

### Hystrix Fallback 사용하기
* Feign으로 정의한 Interface를 직접 구현하고 Spring Bean으로 선언
~~~java
@Component
public class ProductResourceFallback implements ProductResource {
  @Override
  public String getItemDetail(String itemId) {
    return "default value";
  }
}

@FeignClient(fallback=ProductResourceFallback.class, name="dp", url="http://localhost:8080/")
public interface ProductResource {
  @RequestMapping(value="/query/{itemId}")
  String getItemDetail(@PathVariable String itemId);
} 
~~~
* 단점
  * 에러가난 원인을 알 수 없다.
    * 인터페이스를 상속한 클래스이기 때문에 에러를 넣을 부분이 없다. (시그니처가 같아야 하기 때문에)
* 대안
  * FallbackFactory 사용

### Fallback Factory 사용하기

~~~java
@Component
public class FeignProductRemoteServiceFallbackFactory implements FallbackFactory<FeignProductRemoteService> {
    @Override
    public FeignProductRemoteService create(Throwable cause) {
        System.out.println("t = " + cause);
        return productId -> "[ this product id sold out ]";
    }
}
~~~

### Fiegn용 Hystrix 프로퍼티 정의하기

[display] `application.yml` 설정 추가
~~~yaml
hystrix:
  command:
    ProductInfo: # commandKey 이름 주의 'default'는 글로벌 설정임
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
      circuitBreaker:
        requestVolumeThreshold: 1 # default: 20
        errorThresholdPercentage: 50 # default: 50
    FeignProductRemoteService#getProductInfo(String): # 함수명과 인자를 적어줌
      execute:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
      circuitBreaker:
        requestVolumeThreshold: 1
        errorThresholdPercentage: 50
~~~

### 정리
* Hystrix를 통한 메소드별 Circuit Breaker
  * Hystrix를 사용하려면 추가 설정 필요
* Eureka 타겟 서버 주소 획득
* Robbin을 통한 Client-Side Load Balancing

### 장애 유형별 동작 예
#### 특정 API 서버의 인스턴스가 한개 Down 된 경우
* Eureka: HeartBeat 송신이 중단 됨으로 인해 일정 시간 후 목록에서 사라짐
* Ribbon: IOException이 발생한 경우 다른 인스턴스로 Retry
* Hystrix: Circuit은 오픈되지 않음(error = 33%), Fallback, Timeout은 동작

#### 특정 API가 비정상적으로 동작하는 경우(지연, 에러)

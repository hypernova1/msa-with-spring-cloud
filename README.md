# 스프링 클라우드를 활용한 MSA 기초

### 학습 자료

[스프링 클라우드를 활용한 MSA 기초](https://www.youtube.com/watch?v=D6drzNZWs-Y&list=PL9mhQYIlKEhdtYdxxZ6hZeb0va2Gm17A5)

## 목표

* 모놀리식이 나온 배경과 아키텍쳐를 이해한다
* Cloud Native한 MSA를 이해한다
* Netflix OSS, Spring Cloud를 통해 MSA를 구축한다.

### 기존 모놀리식 방식의 개발

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


#### 앞으로의 방향을 결정해야 한다.

1. 새로운 언어, 깔끔한 코드로 전면개편
2. MSA 플랫폼을 구축하여 기존 레거시를 고시시킴


## MSA

MSA란 시스템을 여러개의 독립된 서비스로 나눈 후, 이 서비스를 조합함으로서 기능을 제공하는 아키텍쳐 디자인 패턴이다. 현재까지 공식적인 정의는 없지만 다음과 같은 공감대가 존재한다.

* 각 서비스 간 네트워크를 통해(보통 http) 통신
* 각 서비스는 독립된 배포 단위를 가진다.
* 각 서비스는 쉽게 교체 가능하다.
* 각 서비스는 기능 중심으로 구성된다.
  * ex) 프론트 엔드, 추천, 정산, 상품 등
* 각 서비스에 적합한 프로그래밍 언어, 데이터베이스, 환경으로 만들어진다.
* 서비스의 크기가 작고, 상황에 따라 경계를 정하고, 자율적으로 개발되고, 독립적으로 배포되고, 분산되고, 자동화된 프로세스로 구축되고 배포된다.
* 마이크로서비스는 한 팀에 의해 개발할 수 있는 크기가 상한선이다. 3~9명의 사람들이 개발을 할 수 없을 정도로 커지면 안 된다.

### 모놀리식 vs 마이크로서비스

![monolithic vs microservice](https://blogs.bmc.com/wp-content/uploads/2018/10/microservices-vs-monolithic-1024x544.png)


### 기존 모놀리식 방식의 문제를 MSA로 변경

공통 로직(share.jar)를 제거하고 각각의 마이크로서비스 별로 를 가지고 api를 통해서만 서로를 호출한다.

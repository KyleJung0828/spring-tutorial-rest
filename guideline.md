이 문서는 [이 웹페이지][1] 에서 설명된 튜토리얼입니다.

# Getting Started

Spring Initializr를 이용해 프로젝트를 시작합니다.

저는 IntelliJ에서 제공하는 Spring Initializr 플러그인을 통해 작업을 했는데, 하는 방법은 다음과 같습니다

- File 탭 > New > Project 클릭하여 New Project 다이얼로그 열기
- New Project 다이얼로그에서 왼쪽 리스트뷰에서 Spring Initializr 선택
- Project SDK는 1.8, "Choose Initializr Service URL"은 Default로 설정한 후 Next 버튼 클릭
- 다음 설정 창에서는 프로그램의 기본 정보를 작성하는 곳인데, 특별한 설정을 줄 필요가 없어서 default로 되어 있는 정보를 그대로 썼음. 다만 "Type"은 Maven Project 선택 (이 튜토리얼은 Maven을 사용) 후 Next 버튼 클릭
- 다음 설정 창에서는 의존성 패키지를 선택하는 곳인데, "Web (Spring Web)", "JPA (Spring Data JPA)", "H2 (H2 Database)", "Lombok"의 4가지를 체크한 후 Next 버튼 클릭
- 마지막 설정 창에서는 이름을 정하는데, 따로 정하고 싶지 않다면 default로 되어 있는 이름을 그대로 사용하고, Finish 버튼을 눌러 설정 완료.

설정이 완료되면 IntelliJ에서 의존성 패키지를 백그라운드에서 설치하기 시작합니다. 조금 시간이 걸릴 수 있습니다.

# The Story so Far...

가장 만들기 쉬운 간단한 것부터 시작해봅시다. 사실 가장 간단하게 만드려면 REST 컨셉마저 배제할 수 있습니다 (나중에 차이점을 알기 위해 REST를 더해보겠습니다).

우리가 사용할 예시는 회사의 직원 급여 서비스입니다. 간단히 말해, H2 인메모리 데이터베이스에 직원 객체를 넣고, JPA를 통해 접근하는 것입니다. 이는 스프링 MVC 레이어에 래핑되어 원격으로 접근이 가능할 것입니다.

`nonrest/src/main/java/payroll/Employee.java`
```java
package payroll;

import lombok.Data;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

@Data
@Entity
class Employee {

  private @Id @GeneratedValue Long id;
  private String name;
  private String role;

  Employee() {}

  Employee(String name, String role) {
    this.name = name;
    this.role = role;
  }
}
```

작은 크기의 코드지만, 이 자바 클래스는 많은 것을 담고 있습니다:
- `@Data`는 필드를 기반으로 모든 getter들, setter들, `equals`, `hash`, `toString` 메서드를 만들어주는 Lombok 애너테이션입니다.
- `@Entity`는 JPA 기반 데이터의 스토리지에 객체가 담길 수 있도록 준비해주는 JPA 애너테이션입니다.
- `id`, `name`, `role`은 도메인 객체의 속성인데, `id`의 경우 `@Id`와 `@GeneratedValue`라는 추가적인 JPA 애너테이션이 달려 있어 주요 키 (primary key)라는 점과 JPA 제공자에 의해 자동으로 생성된다는 특징이 있습니다.
- 새로운 인스턴스를 생성해야할 때, 커스텀 생성자가 불리는데 아직 id가 없습니다.

이러한 도메인 객체 정의를 하였으니, 이제 Spring Data JPA를 이용해 지겨운 데이터베이스 작업을 해보겠습니다. Spring Data 저장소는 백엔드 데이터 스토어에 대한 읽기, 업데이트, 삭제, 기록 생성 등을 지원하는 메서드가 담긴 인터페이스입니다. 몇몇 저장소는 적합한 곳에서 데이터 페이징과 정렬을 지원합니다. Spring Data는 메소드의 이름 컨벤션을 기반으로 여러 구현들을 합성합니다.

> JPA 외에도 여러 저장소가 존재합니다. Spring Data MongoDB, Spring Data GemFire, Spring Data Cassandra 등이 있습니다. 이 튜토리얼에서는 JPA를 사용하겠습니다.

`nonrest/src/main/java/payroll/EmployeeRepository.java`
```java
package payroll;

import org.springframework.data.jpa.repository.JpaRepository;

interface EmployeeRepository extends JpaRepository<Employee, Long> {

}
```

이 인터페이스는 `JpaRepository`를 `extends`하고 있고, 도메인 타입이 `Employee`와 id 타입이 `Long`이라는 것을 나타내고 있습니다. 이 인터페이스는 비어 있는 것처럼 보이지만, 다음과 같은 기능을 제공합니다.
    - 새로운 인스턴스 생성
    - 이미 존재하는 인스턴스의 업데이트
    - 삭제
    - 검색 (간단하거나 복잡한 특성을 사용해서 단일, 전체 검색)

믿거나 말거나, 애플리케이션을 띄우기에 충분한 조건이 갖춰줬습니다! Spring Boot 애플리케이션의 가장 최소화된 버전은 `public static void main` 진입점과 `@SpringBootApplication` 애너테이션으로 이뤄져 있습니다. 이게 Spring Boot로 하여금 동작할 수 있게 하는 것입니다.

`nonrest/src/main/java/payroll/PayrollApplication.java`

```java
package payroll;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PayrollApplication {

  public static void main(String... args) {
    SpringApplication.run(PayrollApplication.class, args);
  }
}
```

`@SpringBootApplication`은 메타 애너테이션으로, component scanning, autoconfiguration, property support를 지원합니다.
간단히 말해, 이는 서블릿 컨테이너에 시동을 걸고 서빙을 시작해줄 것입니다.

그래도 아무 데이터가 없는 애플리케이션은 전혀 흥미롭지 않겠죠, 그러니까 preload 해봅시다. 다음 클래스는 Spring에 의해 자동으로 로딩될 것입니다:

`nonrest/src/main/java/payroll/LoadDatabase.java`

```java
package payroll;

import lombok.extern.slf4j.Slf4j;

import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@Slf4j
class LoadDatabase {

  @Bean
  CommandLineRunner initDatabase(EmployeeRepository repository) {
    return args -> {
      log.info("Preloading " + repository.save(new Employee("Bilbo Baggins", "burglar")));
      log.info("Preloading " + repository.save(new Employee("Frodo Baggins", "thief")));
    };
  }
}
```

이렇게 로딩이 될 때 무슨 일이 일어날까요?
- 애플리케이션 컨텍스트가 로딩되면 Spring Boot가 모든 `CommandLineRunner` bean을 run 시킬 것입니다. (주: Spring Boot에서는 `@Bean` 애너테이션을 붙여 `CommandLineRunner` 클래스를 bean이라는 싱글턴 객체로 등록하여, 다른 곳에서 사용하게 됩니다.)
- 이 runner가 당신이 방금 만든 `EmployeeRepository`의 복사본을 요청할 것입니다.
- 이를 사용하여 두 개의 엔티티를 만들고 저장할 것입니다.
- `@Slf4j`는 Lombok 애너테이션으로, Slf4j 기반의 `LoggerFactory`를 `log`로 자동 생성하여 새롭게 생성된 `Employee`에 대한 로그를 기록하게 해줍니다.

`PayrollApplication`에 대고 우클릭하여 `Run`하게 되면 다음과 같은 결과를 얻게 될 것입니다:

```console
...
2018-08-09 11:36:26.169  INFO 74611 --- [main] payroll.LoadDatabase : Preloading Employee(id=1, name=Bilbo Baggins, role=burglar)
2018-08-09 11:36:26.174  INFO 74611 --- [main] payroll.LoadDatabase : Preloading Employee(id=2, name=Frodo Baggins, role=thief)
...
```

이건 모든 로그는 아니고, preloading된 데이터의 주요 부분을 보여줍니다 (당연히 콘솔을 전부 체크해야 합니다. 아름다워요.).

# HTTP is the Platform

저장소를 웹 레이어로 감싸기 위해서 이제 Spring MVC을 건드려야 합니다. Spring Boot 덕분에 환경을 마련하는 코드는 아주 짧게 완성이 될 것입니다. 대신 우리는 동작에 초점을 맞춰보겠습니다:

`nonrest/src/main/java/payroll/EmployeeController.java`
```java
package payroll;

import java.util.List;

import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
class EmployeeController {

  private final EmployeeRepository repository;

  EmployeeController(EmployeeRepository repository) {
    this.repository = repository;
  }

  // Aggregate root

  @GetMapping("/employees")
  List<Employee> all() {
    return repository.findAll();
  }

  @PostMapping("/employees")
  Employee newEmployee(@RequestBody Employee newEmployee) {
    return repository.save(newEmployee);
  }

  // Single item

  @GetMapping("/employees/{id}")
  Employee one(@PathVariable Long id) {

    return repository.findById(id)
      .orElseThrow(() -> new EmployeeNotFoundException(id));
  }

  @PutMapping("/employees/{id}")
  Employee replaceEmployee(@RequestBody Employee newEmployee, @PathVariable Long id) {

    return repository.findById(id)
      .map(employee -> {
        employee.setName(newEmployee.getName());
        employee.setRole(newEmployee.getRole());
        return repository.save(employee);
      })
      .orElseGet(() -> {
        newEmployee.setId(id);
        return repository.save(newEmployee);
      });
  }

  @DeleteMapping("/employees/{id}")
  void deleteEmployee(@PathVariable Long id) {
    repository.deleteById(id);
  }
}
```

- 클래스 선언에 있는 `@RestController`는 각 메서드의 리턴 데이터가 템플릿에 렌더링되지 않는 대신 response body에 직접 쓰여지게 된다는 것을 의미합니다. (주: `@RestController`는 RESTful 컨트롤러로, Spring MVC 컨트롤러인 `@Controller`와 `@ResponseBody`를 합쳐놓은 애너테이션입니다. `@Controller`의 경우 ViewResolver를 통해 text/html 타입의 응답을 해서 View Page에 출력됩니다. 반면 `@RestController`의 경우 내부에 `@ResponseBody`가 포함되어 있는데, 이는 MessageConverter를 통해서 application/json, text/plain 등 알맞은 형태로 응답을 해서 HTTP response body에 직접 쓰여지게 됩니다. 다시 말해, `@Controller`는 View Page를 리턴하지만, `@RestController`는 객체를 반환하기만 하면 데이터는 HTTP Response Body에 직접 작성되는 것입니다).
- `EmployeeRepository`는 생성자에 의해 컨트롤러에 주입됩니다.
- 각 작업에 대한 애너테이션이 존재합니다 (`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`이 각각 HTTP `GET`, `POST`, `PUT`, `DELETE` 호출에 대응됩니다).
- `EmployeeNotFoundException`은 직원을 검색했는데 결과가 나오지 않았을 때를 의미하는 예외입니다.

`nonrest/src/main/java/payroll/EmployeeNotFoundException.java`
```java
package payroll;

class EmployeeNotFoundException extends RuntimeException {

  EmployeeNotFoundException(Long id) {
    super("Could not find employee " + id);
  }
}
```

`EmployeeNotFoundException`이 던져지면, 다음과 같은 조그마한 Spring MVC 구성이 사용되어 HTTP 404를 렌더링하게 됩니다:

`nonrest/src/main/java/payroll/EmployeeNotFoundAdvice.java`
```java
package payroll;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.ResponseStatus;

@ControllerAdvice
class EmployeeNotFoundAdvice {

  @ResponseBody
  @ExceptionHandler(EmployeeNotFoundException.class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  String employeeNotFoundHandler(EmployeeNotFoundException ex) {
    return ex.getMessage();
  }
}
```

- `@ResponseBody`는 이 advice가 response body에 바로 렌더링된다는 것을 의미합니다 (주: 위의 `@RestController`의 주석에서 언급했었습니다.)
- `@ExceptionHandler`는 advice를 `EmployeeNotFoundException`이 던져졌을 때만 응답하도록 구성합니다.
- `@ResponseStatus`는 HTTP 404를 의미하는 `HttpStatus.NOT_FOUND` 메시지를 발급하도록 합니다.
- Advice의 body는 내용을 생성합니다. 이 경우는, 예외의 메시지를 주게 됩니다.

IDE에서 애플리케이션을 띄우기 위해서는 `PayrollApplication`의 `public static void main`에 우클릭을 하거나,

Spring Initializr는 maven 래퍼를 사용하므로 다음과 같이 입력합니다:
```console
$ ./mvnw clean spring-boot:run
```
다른 방법으로는, 이미 설치된 maven 버전을 사용해 다음과 같이 입력합니다:
```console
$ mvn clean spring-boot:run
```
앱이 시작되면 곧바로 내부를 조사해볼 수 있습니다.
```console
$ curl -v localhost:8080/employees
```
이는 다음과 같은 결과를 줄 것입니다:
```console
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> GET /employees HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200
< Content-Type: application/json;charset=UTF-8
< Transfer-Encoding: chunked
< Date: Thu, 09 Aug 2018 17:58:00 GMT
<
* Connection #0 to host localhost left intact
[{"id":1,"name":"Bilbo Baggins","role":"burglar"},{"id":2,"name":"Frodo Baggins","role":"thief"}]
```
여기서 간단한 서식으로 pre-load된 데이터들을 볼 수 있습니다.

만약 다음과 같이 존재하지 않는 유저에 대한 쿼리를 날리게 된다면...
```console
$ curl -v localhost:8080/employees/99
```
이런 결과를 보게 되겠죠:
```console
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> GET /employees/99 HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 404
< Content-Type: text/plain;charset=UTF-8
< Content-Length: 26
< Date: Thu, 09 Aug 2018 18:00:56 GMT
<
* Connection #0 to host localhost left intact
Could not find employee 99
```
이 메시지는 HTTP 404 에러를 지정 메시지인 `Could not find employee 99`와 함께 잘 보여줍니다.

실시간 코딩된 결과를 보여주는 것도 쉽습니다:
```console
$ curl -X POST localhost:8080/employees -H 'Content-type:application/json' -d '{"name": "Samwise Gamgee", "role": "gardener"}'
```
라고 하면 새로운 `Employee` 데이터를 만들어 다음과 같은 내용을 다시 전달해 줍니다.
```console
{"id":3,"name":"Samwise Gamgee","role":"gardener"}
```
그리고 이 유저를 이렇게 바꿀 수 있죠:
```console
$ curl -X PUT localhost:8080/employees/3 -H 'Content-type:application/json' -d '{"name": "Samwise Gamgee", "role": "ring bearer"}'
```
그러면 다음과 같이 업데이트 됩니다:
```console
{"id":3,"name":"Samwise Gamgee","role":"ring bearer"}
```

> 당신이 어떻게 제작할것인가에 따라 서비스에 큰 영향을 미칠 수 있습니다. 이 경우, 업데이트보다 교체 (replace)라고 설명하는 것이 적합합니다. 예를 들어, 이름이 제공되지 않았다면 무효화되었을 겁니다.

그리고 삭제도 할 수 있습니다:
```console
```
좋습니다, 그런데 이게 RESTful 서비스일까요? (힌트를 눈치 못 채셨다면, 답은 아니오입니다).

뭐가 빠졌을까요?

# What makes something RESTful?

지금까지, 직원 데이터를 다루는 핵심 작업을 제어하는 웹 기반 서비스를 만들었습니다. 근데 그건 "RESTful"하다고 말하기에는 충분하지 않습니다.

- `/employees/3`같은 Pretty URL이라고만 해서 REST가 아닙니다.
- 단순히 `GET`, `POST` 등을 사용한다고 해서 REST가 아닙니다.
- 모든 CRUD 동작이 정의되었다고 해서 REST가 아닙니다.

사실, 우리가 지금껏 만들었던 건 RPC (Remote Procedure Call)이라고 보는 것이 훨씬 낫습니다. 왜냐면 이 서비스가 어떻게 서로 작용하는지 알 수 있는 방법이 없습니다. 이런걸 요즘 시대에 제공했다가는, 모든 세부 사항을 적은 문서를 적거나 개발자 포털을 호스팅해야 했을 겁니다.

Roy Fielding이 다음과 같이 쓴 글을 본다면 REST와 RPC의 차이점에 대한 단서를 더 알아낼 수 있을 겁니다:

> 나는 HTTP 기반 인터페이스를 모두 REST API라고 부르는 사람들이 너무 많아서 좌절감이 든다. 오늘의 예시는 SocialSite REST API다. 이건 RPC다. RPC라고 소리치고 있다. 디스플레이에 커플링이 너무 많아서 미성년자 관람 불가 등금을 매겨야 할 정도다.

> 하이퍼텍스트가 제약 조건이라는 개념에서 REST 아키텍처 스타일을 명확히 하려면 어떻게 해야할까? 즉, 애플리케이션 상태 엔진이, (그리고 그로인해 API가,) 하이퍼텍스트에 의해 구동되지 않는 경우 RESTful이 될 수 없으며 REST API가 될 수 없다. 그런거다. 어딘가 수정해야 할 잘못된 메뉴얼이 있나?

우리의 구현에서 하이퍼미디어를 넣지 않아서 생기는 부작용은, 클라이언트가 API를 탐색하기 위해 "반드시" URI들을 하드코딩해야 한다는 점입니다. 이는 이커머스의 상승을 집어삼켰던 불안정한 본질과 같은 결과로 이어지게 됩니다. 이는 우리의 JSON 출력이 조금 수정되어야 한다는 걸 의미합니다.

Spring HATEOAS를 소개합니다. 이 프로젝트는 하이퍼미디어 기반 출력물의 작성을 돕기 위해 만들어졌습니다. 우리 서비스를 RESTful하도록 업그레이드하기 위해서, 빌드 파일에 다음 구문을 넣습니다 (주: 의존성 등 빌드 설정을 담당하는 `pom.xml`에 의존성을 추가하는 작업입니다):

`Adding Spring HATEOAS to pom.xml`
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

이 조그마한 라이브러리는 RESTful 서비스를 정의하고 클라이언트의 사용을 위해 적합한 포맷으로 렌더링해줄 수 있는 구조를 제공합니다.

모든 RESTful 서비스의 중요 재료는, 연관된 작업에 링크를 넣는 것입니다. 우리 Controller를 RESTful하게 만들기 위해, 다음과 같은 링크를 넣습니다:

`Getting a single item resource`
```java
@GetMapping("/employees/{id}")
Resource<Employee> one(@PathVariable Long id) {

  Employee employee = repository.findById(id)
    .orElseThrow(() -> new EmployeeNotFoundException(id));

  return new Resource<>(employee,
    linkTo(methodOn(EmployeeController.class).one(id)).withSelfRel(),
    linkTo(methodOn(EmployeeController.class).all()).withRel("employees"));
}
```

`Relevant import statements`
```java
import static org.springframework.hateoas.mvc.ControllerLinkBuilder.*;
import org.springframework.hateoas.Resource;
import org.springframework.hateoas.Resources;
```

이는 우리가 원래 갖고 있었던 점과 매우 비슷하지만, 몇몇 부분이 바뀌었습니다:

- 메서드의 리턴 타입이 `Employee`에서 `Resource<Employee>`로 바뀌었습니다. `Resource<T>`는 Spring HATEOAS의 제네릭 컨테이너로, 데이터 뿐만 아니라 링크들도 담게 됩니다.
- `linkTo(methodOn(EmployeeController.class).one(id)).withSelfRel()`은 Spring HATEOAS에게 `EmployeeController`의 `one()` 메서드에 링크를 만들도록 요청하고 self 링크라는 것을 표시합니다.
- `linkTo(methodOn(EmployeeController.class).all()).withRel("employees")`는 Spring HATEOAS에게 `all()`이라는 aggregate root에 링크를 만들도록 요청하고, "employees"라고 부릅니다.

"링크를 만든다"는게 무슨 뜻일까요? Spring HATEOAS의 핵심 타입 중 하나가 `Link`입니다. 여기엔 URI와 rel (관계)이 포함되어 있습니다. 링크는 웹에 힘을 실어주는 것입니다. World Wide Web 전에, 다른 문서 시스템은 정보나 링크들을 렌더링해줬겠지만, 웹을 한데 모아준 것은 바로 "데이터를 가진" 문서의 링크였습니다.

Roy Fielding은 웹을 성공적으로 만들었던 기술로 API를 만드는 것을 권장했고, 링크가 그 중 하나인 것입니다.

애플리케이션을 다시 시작하고 Bilbo의 직원 기록을 쿼리한다면, 이전과 약간 다른 응답을 받게 될 것입니다:

`RESTful representation of a single employee`
```console
{
  "id": 1,
  "name": "Bilbo Baggins",
  "role": "burglar",
  "_links": {
    "self": {
      "href": "http://localhost:8080/employees/1"
    },
    "employees": {
      "href": "http://localhost:8080/employees"
    }
  }
}
```
위는 비압축 출력물로, 이전에 봤던 데이터 요소 (`id`, `name`, `role`) 뿐만 아니라 두 개의 URI를 가진 `_links`를 보여줍니다. 이 전체 문서는 HAL을 이용해 서식화되었습니다.

HAL은 경량 미디어타입으로, 데이터 뿐만 아니라 하이퍼미디어 컨트롤의 인코딩이 가능하게 해줘 API의 다른 부분의 컨슈머들이 앞으로 탐색할 수 있다는 것을 알려줍니다. 이 경우, "self" 링크 (코드의 `this`와 비슷한 개념)와 aggregate root에 대한 링크가 있습니다.

Aggregate root 또한 RESTful로 만들기 위해서는 상위 레벨 링크를 추가해야 하는데, 그 안에도 RESTful 요소가 있어야 합니다.
`Getting an aggregate root resource`
```java
@GetMapping("/employees")
Resources<Resource<Employee>> all() {

  List<Resource<Employee>> employees = repository.findAll().stream()
    .map(employee -> new Resource<>(employee,
      linkTo(methodOn(EmployeeController.class).one(employee.getId())).withSelfRel(),
      linkTo(methodOn(EmployeeController.class).all()).withRel("employees")))
    .collect(Collectors.toList());

  return new Resources<>(employees,
    linkTo(methodOn(EmployeeController.class).all()).withSelfRel());
}
```
와우! `repository.findAll()`이었던 메서드가 아주 커졌군요! 이제 풀어봅시다.

`Resources<>`는 컬렉션들을 캡슐화하기 위한 Spring HATEOAS 컨테이너입니다. 이것도 마찬가지로 링크를 포함할 수 있도록 해줍니다. 첫 문장을 그냥 지나치면 안되겠죠. "컬렉션들을 캡슐화"한다는게 무슨 의미일까요? `Employee`의 컬렉션일까요?

아닙니다.

우리는 REST를 얘기하고 있기 때문에, `Employee` resource의 컬렉션을 캡슐화해야 합니다.

그게 바로 모든 `Employee`들을 가져오지만 `Resource<Employee>` 객체의 리스트로 변환시키는 이유입니다 (Java 8 스트림 API 고마워요!)

애플리케이션을 재시작하고 aggregate root를 가져오게 되면 다음과 같은 결과를 보게 됩니다:

`RESTful representation of a collection of employee resources`
```console
{
  "_embedded": {
    "employeeList": [
      {
        "id": 1,
        "name": "Bilbo Baggins",
        "role": "burglar",
        "_links": {
          "self": {
            "href": "http://localhost:8080/employees/1"
          },
          "employees": {
            "href": "http://localhost:8080/employees"
          }
        }
      },
      {
        "id": 2,
        "name": "Frodo Baggins",
        "role": "thief",
        "_links": {
          "self": {
            "href": "http://localhost:8080/employees/2"
          },
          "employees": {
            "href": "http://localhost:8080/employees"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:8080/employees"
    }
  }
}
```

이 aggregate root는 `Employee` resource의 컬렉션을 제공하고, 상위 레벨 "self" 링크가 있습니다. 컬렉션은 `_embedded` 밑에 나열되어 있습니다. 이는 HAL이 어떻게 컬렉션을 표현하는지 보여줍니다.

컬렉션의 각각의 멤버는 정보와 관련 링크를 담고 있습니다.

이 모든 링크를 더하는 것이 무슨 의미가 있을까요? 바로 REST 서비스가 시간이 지나면서 진화할 수 있게 만들어줍니다. 기존에 존재하는 링크들이 유지되고, 나중에는 새로운 링크들이 추가됩니다. 새로운 클라이언트들은 새로운 링크를 이용할 수 있고, 기존의 클라이언트들은 기존의 링크를 유지할 수 있습니다. 이는 서비스가 이전되거나 이리 저리 옮겨졌을 때 특히 유용합니다. 링크 구조가 유지되는 한 클라이언트가 그 링크를 찾아 상호작용을 할 수 있으니까요.

# Simplifying Link Creation

단일 `Employee` 링크 생성 부분에서 반복되는 부분을 눈치 채셨나요? `Employee`에 대한 단일 링크와 aggregate root에 대한 "employees" 링크 가 두 번 나타나고 있죠. 이걸 보고 뭔가 생각이 들었다면, 좋습니다! 해결책이 있어요.

간단히 말해, `Employee` 객체를 `Resource<Employee>` 객체로 변환하는 한수를 정의해야 합니다. 이 메서드를 직접 쉽게 코딩할 수 있겠지만, Spring HATEOAS의 `ResourceAssembler` 인터페이스의 도움을 받을 수 있습니다.

`evolution/src/main/java/payroll/EmployeeResourceAssembler.java`
```java
package payroll;

import static org.springframework.hateoas.mvc.ControllerLinkBuilder.*;

import org.springframework.hateoas.Resource;
import org.springframework.hateoas.ResourceAssembler;
import org.springframework.stereotype.Component;

@Component
class EmployeeResourceAssembler implements ResourceAssembler<Employee, Resource<Employee>> {

  @Override
  public Resource<Employee> toResource(Employee employee) {

    return new Resource<>(employee,
      linkTo(methodOn(EmployeeController.class).one(employee.getId())).withSelfRel(),
      linkTo(methodOn(EmployeeController.class).all()).withRel("employees"));
  }
}
```








[1]: https://spring.io/guides/tutorials/rest/

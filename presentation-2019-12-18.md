이 문서는 [이 웹페이지][1] 에서 설명된 튜토리얼입니다.

# Getting Started

Spring Initializr를 이용해 프로젝트를 시작합니다.

저는 IntelliJ에서 제공하는 Spring Initializr 플러그인을 통해 작업을 하였습니다.

- File 탭 > New > Project 클릭하여 New Project 다이얼로그 열기
- New Project 다이얼로그에서 왼쪽 리스트뷰에서 Spring Initializr 선택
- Project SDK는 1.8, "Choose Initializr Service URL"은 Default로 설정한 후 Next 버튼 클릭
- 다음 설정 창에서는 프로그램의 기본 정보를 작성하는 곳인데, 특별한 설정을 줄 필요가 없어서 default로 되어 있는 정보를 그대로 썼음. 다만 "Type"은 Maven Project 선택 (이 튜토리얼은 Maven을 사용) 후 Next 버튼 클릭
- 다음 설정 창에서는 의존성 패키지를 선택하는 곳인데, "Web (Spring Web)", "JPA (Spring Data JPA)", "H2 (H2 Database)", "Lombok"의 4가지를 체크한 후 Next 버튼 클릭
- 마지막 설정 창에서는 이름을 정하는데, 따로 정하고 싶지 않다면 default로 되어 있는 이름을 그대로 사용하고, Finish 버튼을 눌러 설정 완료.

설정이 완료되면 IntelliJ에서 의존성 패키지를 백그라운드에서 설치하기 시작합니다. 조금 시간이 걸릴 수 있습니다.

# 지금까지의 이야기...

가장 만들기 쉬운 간단한 비-REST 서비스를 만들어봅니다. (나중에 차이점을 알기 위해 REST를 넣을 예정입니다)

우리가 사용할 예시는 회사의 직원 급여 서비스입니다. 간단히 말해, H2 인메모리 데이터베이스에 직원 객체를 넣고, JPA를 통해 접근하는 것입니다. 이는 스프링 MVC 레이어에 래핑되어, 원격으로 접근이 가능할 것입니다.

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

- `@Data`는 필드를 기반으로 모든 getter들, setter들, `equals`, `hash`, `toString` 메서드를 만들어주는 Lombok 애너테이션입니다.
- `@Entity`는 JPA 기반 데이터의 스토리지에 객체가 담길 수 있도록 준비해주는 JPA 애너테이션입니다.
- `id`, `name`, `role`은 도메인 객체의 속성인데, `id`의 경우 `@Id`와 `@GeneratedValue`라는 추가적인 JPA 애너테이션이 달려 있어 주요 키 (primary key)라는 점과 JPA 제공자에 의해 자동으로 생성된다는 특징이 있습니다.
- 생성자에서 `name`과 `role` 매개변수를 받아 새로운 인스턴스를 생성하고, `id`는 `@Id`에 의해 자동 지정됩니다.

도메인 객체 정의가 끝났습니다. 이제 Spring Data JPA를 이용해 데이터베이스 작업을 해보겠습니다. Spring Data 저장소는 백엔드 데이터에 대한 읽기, 업데이트, 삭제, 기록 생성 등을 지원하는 메서드가 담긴 인터페이스입니다. 몇몇 저장소는 데이터 페이징과 정렬을 지원합니다. Spring Data는 메소드의 이름 컨벤션을 기반으로 여러 구현들을 합성합니다.

> JPA 외에도 여러 저장소가 존재합니다. Spring Data MongoDB, Spring Data GemFire, Spring Data Cassandra 등이 있습니다.

`nonrest/src/main/java/payroll/EmployeeRepository.java`
```java
package payroll;

import org.springframework.data.jpa.repository.JpaRepository;

interface EmployeeRepository extends JpaRepository<Employee, Long> {

}
```

이 인터페이스는 `JpaRepository`를 확장하고 있는데, 도메인 타입은 `Employee`이고 id 타입은 `Long`이라는 것을 나타내고 있습니다. 이 인터페이스는 비어 있는 것처럼 보이지만, 다음과 같은 기능을 제공합니다.
    - 새 인스턴스 생성
    - 기존 인스턴스 업데이트
    - 삭제
    - 검색

믿거나 말거나, 애플리케이션을 띄우기에 충분한 조건이 갖춰줬습니다. Spring Boot 애플리케이션의 가장 간소화 버전은 바로 `public static void main` 엔트리 포인트와 `@SpringBootApplication` 애너테이션으로 이뤄져 있습니다. 이게 Spring Boot가 동작할 수 있게 하는 것입니다.

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

`@SpringBootApplication`은 서블릿 컨테이너에 시동을 걸어서 서빙을 시작해줍니다.

그래도 아무 데이터가 없는 애플리케이션은 전혀 흥미롭지 않겠죠, 그러니까 데이터를 프리로딩 (preloading) 해봅니다. 다음과 같이 클래스를 작성하면, Spring에 의해 자동으로 로딩될 것입니다:

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
- 애플리케이션 컨텍스트가 로딩될 때, Spring Boot가 `@Configuration`이 붙은 클래스를 찾아가 `@Bean`이 붙은 `CommandLineRunner` bean을 생성해줍니다.
    - 주: Spring Boot에서는 `@Bean` 애너테이션을 붙여 bean이라는 싱글턴 객체로 등록합니다. 이렇게 만들어진 객체는 싱글턴이므로 다른 곳에서 사용할 수 있게 됩니다.
- 두 개의 `Employee` 엔티티를 만들고 `EmployeeRepository`에 저장할 것입니다.
- `@Slf4j`는 Lombok 애너테이션으로, Slf4j 기반의 `LoggerFactory`를 `log`로 자동 생성하여 새롭게 생성된 `Employee`에 대한 로그를 기록하도록 해줍니다.

`PayrollApplication`에 대고 우클릭하여 `Run`하게 되면 다음과 같은 결과를 얻게 될 것입니다:

```console
...
2018-08-09 11:36:26.169  INFO 74611 --- [main] payroll.LoadDatabase : Preloading Employee(id=1, name=Bilbo Baggins, role=burglar)
2018-08-09 11:36:26.174  INFO 74611 --- [main] payroll.LoadDatabase : Preloading Employee(id=2, name=Frodo Baggins, role=thief)
...
```

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

- 클래스 선언에 있는 `@RestController`는 각 메서드의 리턴 데이터가 템플릿에 렌더링되지 않는 대신 response body에 직접 쓰여지게 된다는 것을 의미합니다.
    - 주: `@RestController`는 RESTful 컨트롤러로, Spring MVC 컨트롤러인 `@Controller`와 `@ResponseBody`를 합쳐놓은 애너테이션입니다. `@Controller`의 경우 ViewResolver를 통해 text/html 타입의 응답을 해서 View Page에 출력됩니다. 반면 `@RestController`의 경우 내부에 `@ResponseBody`가 포함되어 있는데, 이는 MessageConverter를 통해서 application/json, text/plain 등 알맞은 형태로 응답을 해서 HTTP response body에 직접 쓰여지게 됩니다. 다시 말해, `@Controller`는 View Page를 리턴하지만, `@RestController`는 객체를 반환하기만 하면 데이터가 HTTP Response Body에 직접 작성되는 것입니다.
- `EmployeeRepository`는 생성자에 의해 컨트롤러에 주입됩니다.
- 각 작업에 대한 애너테이션이 존재합니다 (`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`이 각각 HTTP의 `GET`, `POST`, `PUT`, `DELETE` 호출에 대응됩니다).
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
```json
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
```json
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

이 간단한 인터페이스는 한 개의 메서드를 가지고 있습니다: `toResource()`. 이 메서드는 비-resource 객체 (`Employee`)를 resource 기반의 객체 (`Resource<Employee>`)로 변환해줍니다.

이전에 봤었던 컨트롤러 코드가 이 클래스 안으로 옮겨질 수 있습니다. 그리고 Spring Framework의 `@Component`를 적용하여 앱이 시작할 때 이 컴포넌트가 자동으로 생성될 것입니다.

> Spring HATEOAS에서 모든 Resource에 대한 추상 베이스 클래스는 `ResourceSupport`입니다. 하지만 명료함을 위해, 모든 POJO를 쉽게 감싸는 방법으로 `Resource<T>`의 사용을 권장합니다.

이 assembler를 활용하기 위해서 `EmployeeController`의 생성자 안에 assembler를 주입하는 방식으로 변경하면 됩니다.

`Injecting EmployeeResourceAssembler into the controller`
```java
@RestController
class EmployeeController {

  private final EmployeeRepository repository;

  private final EmployeeResourceAssembler assembler;

  EmployeeController(EmployeeRepository repository,
             EmployeeResourceAssembler assembler) {

    this.repository = repository;
    this.assembler = assembler;
  }

  ...

}
```

지금부터 단일 직원을 가져오는 메서드에서 사용할 수 있습니다.

`Getting single item resource using the assembler`
```java
@GetMapping("/employees/{id}")
Resource<Employee> one(@PathVariable Long id) {

  Employee employee = repository.findById(id)
    .orElseThrow(() -> new EmployeeNotFoundException(id));

  return assembler.toResource(employee);
}
```

이 코드는 거의 같은데, `Resource<Employee>` 인스턴스를 여기서 생성하지 않고 assembler에 대리해주는 점만 다릅니다. 별로 다른 건 없어보이죠?

Aggregate root 컨트롤러 메서드에 같은 방식을 적용하는 것은 더 인상적인 결과를 보여줍니다:

`Getting aggregate root resource using the assembler`
```java
@GetMapping("/employees")
Resources<Resource<Employee>> all() {

  List<Resource<Employee>> employees = repository.findAll().stream()
    .map(assembler::toResource)
    .collect(Collectors.toList());

  return new Resources<>(employees,
    linkTo(methodOn(EmployeeController.class).all()).withSelfRel());
}
```

이 코드도 마찬가지로 거의 같은데, `Resource<Employee>` 생성 로직을 `map(assembler::toResource)`로 바꾼 점이 다릅니다. Java 8 메서드 참조 덕분에 이렇게 쉽게 끼워넣어 컨트롤러를 간단히 만들 수 있습니다.

> Spring HATEOAS의 중요 목표는 "올바른 것"을 쉽게 하는 것입니다. 이 시나리오에서는 하드코딩 하나 없이 하이퍼미디어를 서비스에 추가하였습니다.

이 단계에서 당신은 실제로 하이퍼미디어 기반 컨텐츠를 생성하는 Spring MVC REST 컨트롤러를 만들었습니다. HAL을 모르는 클라이언트는 추가적인 부분을 무시하고 순수 데이터만 사용하면 됩니다. HAL을 아는 클라이언트는 더 나아진 API를 탐색할 수 있겠죠.

그렇지만 이건 Spring을 이용해서 진정한 RESTful 서비스를 만들기 위해 필요한 것 중 하나일 뿐입니다.

# Evolving REST APIs

라이브러리 한 개를 추가하고 몇 줄의 코드를 작성하여 당신의 애플리케이션에 하이퍼미디어를 추가하였습니다. 하지만 이게 당신의 서비스를 RESTful하게 만들기 위한 유일한 재료는 아닙니다. REST의 중요한 측면은 이게 기술 스택이나 어떠한 표준이 아니라는 점입니다.

REST는 구조적 제약을 모아둔 것으로, 애플리케이션에 적용 되었을 때 더욱 탄력적으로 (resilient) 만들게 해줍니다. 탄력의 중요한 점은, 당신의 서비스를 업그레이드 하였을 때, 클라이언트가 다운타임 (downtime)으로 인해 피해를 받지 않는다는 점입니다.

"지난 날"에는, 업그레이드라는 것은 클라이언트를 고장나게 하는 것으로 악명이 높았습니다. 즉, 서버가 업그레이드하려면 클라이언트도 업그레이드되어야 했었습니다. 요즘 시대에는, 업그레이드로 인한 수 시간 수 분 동안의 다운타임이 몇 십억원의 매출 손실로 이어질 수 있습니다.

몇몇 회사들은 다운타임을 최소화하기 위한 계획을 경영진에게 제시하도록 요구합니다. 예전에는, 부하가 가장 최소화되는 월요일 오전 2시에 업그레이드할 수 있었습니다. 하지만 현시대에 이르러는 외국 고객들이 있는 인터넷 기반 전자상거래에서 이런 전략이 효과적이지 못합니다.

SOAP 기반 서비스와 CORBA 기반 서비스는 믿을 수 없을 정도로 취약했습니다. 예전 클라이언트와 새로운 클라이언트를 모두 지원하는 서버를 롤아웃하기 힘들었습니다. REST 기반에서는 훨씬 쉽습니다. Spring 스택을 사용한다면 더욱 더요.

이 설계 문제를 생각해봅시다: `Employee` 기반 기록 시스템을 롤아웃하였습니다. 이 시스템이 히트를 쳤습니다. 수많은 기업에 이 시스템을 팔았습니다. 갑자기, 직원 이름을 `firstName`과 `lastName`으로 나눠야 하는 필요성이 생겼습니다.

어, 이 생각은 못했네.

자 `Employee` 클래스를 열어서 `name` 필드를 `firstName`과 `lastName`으로 교체하기 전에, 잠깐 멈춰서 생각해보세요. 이게 어떤 클라이언트를 고장나게 할까요? 그 클라이언트를 업그레이드하는 데 얼마나 시간이 걸릴까요? 심지어, 당신 서비스에 접근하는 모든 클라이언트를 다 제어하나요?

다운타임 = 금전적 손실. 경영진이 이걸 받아들일 준비가 되어 있을까요?

REST가 나오기 수 년 전부터 있었던 오래된 전략이 하나 있습니다.

> 데이터베이스에서 column을 절대 지우지 마라. - 알 수 없는 이.

데이터베이스 테이블에 항상 column이나 필드를 넣을 수 있습니다. 하지만 지우면 안 됩니다. RESTful 서비스의 원칙도 같습니다. 다음과 같이, JSON 표현에 새로운 필드를 넣기만 하고, 지우면 안됩니다:

``
```json
{
  "id": 1,
  "firstName": "Bilbo",
  "lastName": "Baggins",
  "role": "burglar",
  "name": "Bilbo Baggins",
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

이 형식에서 `firstName`, `lastName`, `name`을 어떻게 나타내는지 보이시나요? 정보의 중복을 보여주긴 하지만, 예전과 새로운 클라이언트 모두를 지원하는 목적을 가집니다. 다운타임을 줄일 만한 좋은 조치죠.

"이전 방식"과 "새로운 방식" 모두에서 이 정보를 보여줘야 하기도 하지만, 두 방식으로 들어오는 데이터를 처리해야 합니다.

어떻게요? 간단해요. 이렇게요:

`Employee record that handles both "old" and "new" clients`
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
  private String firstName;
  private String lastName;
  private String role;

  Employee() {}

  Employee(String firstName, String lastName, String role) {
    this.firstName = firstName;
    this.lastName = lastName;
    this.role = role;
  }

  public String getName() {
    return this.firstName + " " + this.lastName;
  }

  public void setName(String name) {
    String[] parts =name.split(" ");
    this.firstName = parts[0];
    this.lastName = parts[1];
  }
}
```

이전 버전의 `Employee`와 굉장히 비슷한 클래스입니다. 변화점을 하나씩 살펴봅시다:

- `name` 필드는 `firstName`과 `lastName`으로 교체되었습니다. Lombok이 이 필드에 대한 getter와 setter를 생성할 것입니다.
- 기존의 `name` 특성을 위한 "가상" getter로 `getName()`이 정의되었습니다. 이 메서드는 `firstName`과 `lastName` 필드를 이용해 값을 만들어냅니다.
- 기존의 `name` 특성을 위한 "가상" setter도 `setName()`에서 정의되었습니다. 들어오는 스트링을 파싱해서 알맞은 필드에 저장합니다.

물론 모든 API의 변화가 스트링을 분리하거나 합치는 것처럼 간단하진 않겠죠. 하지만 대부분의 상황에서 변환을 할 수 있는게 불가능하진 않다는 게 확실하지 않나요?

또 다른 미세 조정 방법으로는 각각의 REST 메서드가 적합한 응답을 리턴하도록 보장하는 것이 있습니다.
POST 메서드를 이렇게 업데이트하세요:

`POST that handles "old" and "new" client requests`
```java
@PostMapping("/employees")
ResponseEntity<?> newEmployee(@RequestBody Employee newEmployee) throws URISyntaxException {

  Resource<Employee> resource = assembler.toResource(repository.save(newEmployee));

  return ResponseEntity
    .created(new URI(resource.getId().expand().getHref()))
    .body(resource);
}
```

- 새로운 `Employee` 객체가 이전과 같은 방식으로 저장되었습니다. 하지만 `EmployeeResourceAssembler`로 감싸졌습니다.
- Spring MVC의 `ResponseEntity`는 HTTP 201 Create 상태 메시지를 만들기 위해 사용되었습니다. 이러한 종류의 응답은 보통 Location 응답 헤더를 포함하고, 우리는 새롭게 만들어진 링크를 사용합니다.
- 추가적으로, 저장된 객체를 resource 기반 버전으로 리턴합니다.

이 변화를 적용하면, 같은 방식으로 레거시인 `name` 필드를 사용해서 새로운 직원 데이터를 만들수 있습니다:

```console
$ curl -v -X POST localhost:8080/employees -H 'Content-Type:application/json' -d '{"name": "Samwise Gamgee", "role": "gardener"}'
```

출력 결과는 다음과 같습니다:

```console
> POST /employees HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Type:application/json
> Content-Length: 46
>
< Location: http://localhost:8080/employees/3
< Content-Type: application/hal+json;charset=UTF-8
< Transfer-Encoding: chunked
< Date: Fri, 10 Aug 2018 19:44:43 GMT
<
{
  "id": 3,
  "firstName": "Samwise",
  "lastName": "Gamgee",
  "role": "gardener",
  "name": "Samwise Gamgee",
  "_links": {
    "self": {
      "href": "http://localhost:8080/employees/3"
    },
    "employees": {
      "href": "http://localhost:8080/employees"
    }
  }
}
```

결과물 객체가 HAL로 렌더된 것 뿐만 아니라 (`name`과 `firstName`/`lastName` 모두), Location 헤더가 `http://localhost:8080/employees/3`에 생성된 것을 볼 수 있습니다. 하이퍼미디어를 사용하는 클라이언트가 이 새로운 자원을 탐색하고 사용을 시작할 수 있습니다.

PUT 컨트롤러 메서드에도 비슷한 변화가 필요합니다:

`Handling a PUT for different clients`
```java
@PutMapping("/employees/{id}")
ResponseEntity<?> replaceEmployee(@RequestBody Employee newEmployee, @PathVariable Long id) throws URISyntaxException {

  Employee updatedEmployee = repository.findById(id)
    .map(employee -> {
      employee.setName(newEmployee.getName());
      employee.setRole(newEmployee.getRole());
      return repository.save(employee);
    })
    .orElseGet(() -> {
      newEmployee.setId(id);
      return repository.save(newEmployee);
    });

  Resource<Employee> resource = assembler.toResource(updatedEmployee);

  return ResponseEntity
    .created(new URI(resource.getId().expand().getHref()))
    .body(resource);
}
```

`save()`를 통해 만들어진 `Employee` 객체는 `EmployeeResourceAssembler`를 이용해 `Resource<Employee>` 객체로 감싸지게 됩니다. 200 OK보다는 더 자세한 HTTP 응답 코드가 필요하기 때문에, Spring MVC의 `ResponseEntity` 래퍼를 사용하겠습니다. 여기엔 유용한 정적 메서드인 `created()`가 있어 이 안에 resource URI를 집어넣을 수 있습니다.

`resource` 를 가져오면 거기에 `getId()` 메서드를 호출해 self 링크를 가져올 수 있습니다. 이 메서드는 `Link`를 얻게 되는데 이걸 Java `URI`로 만들 수 있습니다. 잘 묶어서 보기 위해 `resource` 자체를 `body()` 메서드 안에 주입합니다.

> REST에서는 resource의 URI가 그 resource의 id입니다. 그러므로, Spring HATEOAS는 기저 데이터 타입의 `id` 필드를 직접 건네주지 않습니다 (어떤 클라이언트도 그러면 안되죠). 대신 URI를 넘겨줍니다. `ResourceSupport.getId()`와 `Employee.getId()`를 헷갈리지 마세요.

우리가 항상 새로운 resource를 "생성"하진 않는데, HTTP 201 Created가 과연 적합한 시맨틱을 가지고 있는지는 논란의 여지가 있습니다. 하지만 Location 응답 헤더와 함께 pre-load되기 때문에 run 시킵니다.

```console
$ curl -v -X PUT localhost:8080/employees/3 -H 'Content-Type:application/json' -d '{"name": "Samwise Gamgee", "role": "ring bearer"}'

* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> PUT /employees/3 HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Type:application/json
> Content-Length: 49
>
< HTTP/1.1 201
< Location: http://localhost:8080/employees/3
< Content-Type: application/hal+json;charset=UTF-8
< Transfer-Encoding: chunked
< Date: Fri, 10 Aug 2018 19:52:56 GMT
{
	"id": 3,
	"firstName": "Samwise",
	"lastName": "Gamgee",
	"role": "ring bearer",
	"name": "Samwise Gamgee",
	"_links": {
		"self": {
			"href": "http://localhost:8080/employees/3"
		},
		"employees": {
			"href": "http://localhost:8080/employees"
		}
	}
}
```

그 직원 resource가 이제 업데이트 되었고 location URI가 되돌려 보내졌습니다. 마지막으로, DELETE 동작을 적절히 업데이트 하세요:

`Handling DELETE requests`
```java
@DeleteMapping("/employees/{id}")
ResponseEntity<?> deleteEmployee(@PathVariable Long id) {

  repository.deleteById(id);

  return ResponseEntity.noContent().build();
}
```

이건 HTTP 204 No Content 응답을 리턴합니다.

```console
$ curl -v -X DELETE localhost:8080/employees/1

* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> DELETE /employees/1 HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 204
< Date: Fri, 10 Aug 2018 21:30:26 GMT
```

> `Employee` 클래스의 필드를 바꾸려면 데이터베이스 팀과 협업이 필요할 것입니다. 그래야 그들이 기존 데이터를 새로운 column에 제대로 마이그레이션할 수 있습니다.

당신은 이제 기존 클라이언트를 방해하지 않으면서, 새로운 클라이언트가 개선된 기능을 사용할 수 있도록 해주는 업그레이드를 위한 모든 준비가 완료 되었습니다!

그나저나, 너무 많은 정보를 보내는 것 같이서 걱정되진 않으신가요? 바이트 하나하나가 아까운 시스템에서는 API 진화를 살짝 늦춰야할 때도 있습니다. 하지만 실제로 측정해보기 전까진 미리 최적화를 해보진 마세요.

# Building links into your REST API

지금까지, 뼈대만 있는 링크를 가진 진화 가능한 API를 만들어 봤습니다. API를 더 키우고 클라이언트를 더 잘 서빙하기 위해서 "애플리케이션 상태 엔진으로써의 하이퍼미디어"라는 컨셉을 받아들여야 합니다.

이게 무슨 뜻일까요? 이 섹션에서 자세히 알아볼 겁니다.

비즈니스 로직은 프로세스와 연관된 규칙을 어쩔 수 없이 쌓게 됩니다. 이런 시스템의 위험성은, 바로 서버사이드 로직을 클라이언트로 가져와 강력한 커플링을 만들 수 있다는 점입니다. REST는 이런 연결성을 끊어내고 커플링을 최소화하는 것에 관련 있습니다.

클라이언트의 고장을 유발하지 않으면서 상태 변화를 해결하는 방법을 보여주기 위해, 주문을 이행하는 시스템을 추가한다고 가정해봅시다.

첫 번째 단계로, `Order` 기록을 정의합니다:

`links/src/main/java/payroll/Order.java`
```java
package payroll;

import lombok.Data;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity
@Data
@Table(name = "CUSTOMER_ORDER")
class Order {

  private @Id @GeneratedValue Long id;

  private String description;
  private Status status;

  Order() {}

  Order(String description, Status status) {

    this.description = description;
    this.status = status;
  }
}
```

- 이 클래스에는 테이블 이름을 `CUSTOMER_ORDER`로 바꾸기 위해 JPA `@Table` 애너테이션이 필요한데, 이는 `ORDER`이 테이블 이름으로 유효하지 않기 때문입니다.
- `description` 필드와 `status` 필드를 갖고 있습니다.

주문은 고객이 주문을 넣었을 때와 이행이 되거나 취소되었을 때 여러 상태 변이를 하게 됩니다. 이는 Java `enum`으로 표현될 수 있습니다:

`links/src/main/java/payroll/Status.java`
```java
package payroll;

enum Status {

  IN_PROGRESS,
  COMPLETED,
  CANCELLED;
}
```

이 `enum`은 `Order`이 가질 수 있는 여러 상태를 나타냅니다. 이 튜토리얼에서는 간단하게 표현하겠습니다.

데이터베이스에 있는 주문과 상호작용을 지원하려면 대응되는 Spring Data 저장소를 반드시 정의해야 합니다:

Spring Data JPA’s `JpaRepository` base interface
```java
interface OrderRepository extends JpaRepository<Order, Long> {
}
```

이걸 제 자리에 넣었다면, 이제 기본적인 `OrderController`를 정의할 수 있습니다:

`links/src/main/java/payroll/OrderController.java`
```java
@RestController
class OrderController {

  private final OrderRepository orderRepository;
  private final OrderResourceAssembler assembler;

  OrderController(OrderRepository orderRepository,
          OrderResourceAssembler assembler) {

    this.orderRepository = orderRepository;
    this.assembler = assembler;
  }

  @GetMapping("/orders")
  Resources<Resource<Order>> all() {

    List<Resource<Order>> orders = orderRepository.findAll().stream()
      .map(assembler::toResource)
      .collect(Collectors.toList());

    return new Resources<>(orders,
      linkTo(methodOn(OrderController.class).all()).withSelfRel());
  }

  @GetMapping("/orders/{id}")
  Resource<Order> one(@PathVariable Long id) {
    return assembler.toResource(
      orderRepository.findById(id)
        .orElseThrow(() -> new OrderNotFoundException(id)));
  }

  @PostMapping("/orders")
  ResponseEntity<Resource<Order>> newOrder(@RequestBody Order order) {

    order.setStatus(Status.IN_PROGRESS);
    Order newOrder = orderRepository.save(order);

    return ResponseEntity
      .created(linkTo(methodOn(OrderController.class).one(newOrder.getId())).toUri())
      .body(assembler.toResource(newOrder));
  }
}
```

- 이는 지금까지 당신이 만들었던 컨트롤러와 같은 REST 컨트롤러 설정을 담고 있습니다.
- `OrderRepository`와 `OrderResourceAssembler` (아직 만들진 않았지만)를 주입합니다.
- 처음 두 Spring MVC route들이 aggregate root와 단일 `Order` resource 요청을 처리합니다.
- 세 번째 Spring MVC route는 `IN_PROGRESS` 상태로 시작는 새 주문의 생성을 처리합니다.
- 모든 컨트롤러 메서드는 하이퍼미디어 (혹은 그런 타입의 래퍼)를 제대로 렌더링 해주기 위하여 Spring HATEOAS의 `ResourceSupport` 서브클래스 중 하나를 리턴합니다.

`OrderResourceAssembler`를 만들기 전에, 어떤 일이 일어나야 하는지 논의해봅시다. 당신은 지금 `Status.IN_PROGRESS`, `Status.COMPLETED`, `Status.CANCELLED` 간의 상태 흐름을 모델화하고 있습니다. 이런 데이터를 클라이언트에 서빙할 경우, 클라이언트가 이 payload를 기반으로 뭘 할지 직접 결정하게 하도록 하는 것을 자연스럽게 떠올리게 됩니다.

하지만 그건 잘못되었을 수 있습니다.

이 흐름에 새로운 상태가 추가되면 어떤 일이 벌어질까요? UI의 다양한 버튼 배치가 에러를 나타낼 것입니다.

만약 외국어 지원을 하거나, locale-specific한 텍스트를 보여주기 위해 각 상태의 이름을 바꿨다면 어떻게 될까요? 이건 모든 클라이언트의 장애를 일으킬 가능성이 높습니다.

HATEOAS (Hypermedia as the Engine of Application State)로 오세요. 클라이언트가 payload를 파싱하는 대신 유효한 동작을 나타내기 위해 링크를 주세요. 상태 기반 동작과 데이터 payload 간의 커플링을 제거하세요. 다시 말해, CANCEL과 COMPLETE이 유효한 동작이라면, 링크 리스트에 동적으로 추가하세요. 클라이언트는 링크가 존재할 때만 해당하는 버튼을 유저에게 보여주게 됩니다.

이는 클라이언트가 "언제" 동작이 유효한지 알 필요가 없도록 커플링을 제거해주어, 서버와 클라이언트 사이의 상태 전이 로직이 싱크가 맞지 않을 위험을 낮춰줍니다.

Spring HATEOAS의 `ResourceAssembler` component 컨셉을 받아들였으니, 그런 로직을 `OrderResourceAssembler`에 넣으면 됩니다:

`links/src/main/java/payroll/OrderResourceAssembler.java`
```java
package payroll;

import static org.springframework.hateoas.mvc.ControllerLinkBuilder.*;

import org.springframework.hateoas.Resource;
import org.springframework.hateoas.ResourceAssembler;
import org.springframework.stereotype.Component;

@Component
class OrderResourceAssembler implements ResourceAssembler<Order, Resource<Order>> {

  @Override
  public Resource<Order> toResource(Order order) {

    // Unconditional links to single-item resource and aggregate root

    Resource<Order> orderResource = new Resource<>(order,
      linkTo(methodOn(OrderController.class).one(order.getId())).withSelfRel(),
      linkTo(methodOn(OrderController.class).all()).withRel("orders")
    );

    // Conditional links based on state of the order

    if (order.getStatus() == Status.IN_PROGRESS) {
      orderResource.add(
        linkTo(methodOn(OrderController.class)
          .cancel(order.getId())).withRel("cancel"));
      orderResource.add(
        linkTo(methodOn(OrderController.class)
          .complete(order.getId())).withRel("complete"));
    }

    return orderResource;
  }
}
```

이 resource assembler는 단일 resource에 거는 self 링크와 aggregate root에 거는 링크를 항상 포함하게 됩니다. 이것은 또한 `OrderController.cancel(id)`와 `OrderController.complete(id)`
 (아직 정의되지 않음)에 대한 두 개의 조건부 링크를 포함합니다. 이 링크들은 주문의 상태가 `Status.IN_PROGRESS`일 때만 보이게 됩니다.

만약 클라이언트가 단순이 plain old JSON 데이터를 읽는 게 아니라, HAL을 적용하여 링크를 읽을 수 있다면, 주문 시스템에 대한 도메인 지식을 알 필요성이 줄어들게 됩니다. 이는 자연스럽게 클라이언트와 서버 사이의 커플링을 줄여줍니다. 그리고 이것은 실행 중인 클라이언트에서 장애가 없도록 주문 이행의 흐름을 조정할 수 있게 해줍니다.

`cancel` 작업을 위해 다음 코드를 `OrderController`에 넣어 주문 이행을 손봅시다:

`Creating a "cancel" operation in the OrderController`
```java
@DeleteMapping("/orders/{id}/cancel")
ResponseEntity<ResourceSupport> cancel(@PathVariable Long id) {

  Order order = orderRepository.findById(id).orElseThrow(() -> new OrderNotFoundException(id));

  if (order.getStatus() == Status.IN_PROGRESS) {
    order.setStatus(Status.CANCELLED);
    return ResponseEntity.ok(assembler.toResource(orderRepository.save(order)));
  }

  return ResponseEntity
    .status(HttpStatus.METHOD_NOT_ALLOWED)
    .body(new VndErrors.VndError("Method not allowed", "You can't cancel an order that is in the " + order.getStatus() + " status"));
}
```

이 코드는 `Order`의 취소를 승인하기 전에 그 상태를 검사합니다. 만약 유효한 상태가 아니라면, 하이퍼미디어를 지원하는 에러 컨테이더나인 Spring HATEOAS `VndError`를 리턴합니다. 만약 전이가 유효하다면, `Order`의 상태를 `CANCELLED`로 변경합니다.

주문 완료를 구현하기 위해 다음과 같은 코드를 `OrderController`에 추가하세요:

`Creating a "complete" operation in the OrderController`
```java
@PutMapping("/orders/{id}/complete")
ResponseEntity<ResourceSupport> complete(@PathVariable Long id) {

    Order order = orderRepository.findById(id).orElseThrow(() -> new OrderNotFoundException(id));

    if (order.getStatus() == Status.IN_PROGRESS) {
      order.setStatus(Status.COMPLETED);
      return ResponseEntity.ok(assembler.toResource(orderRepository.save(order)));
    }

    return ResponseEntity
      .status(HttpStatus.METHOD_NOT_ALLOWED)
      .body(new VndErrors.VndError("Method not allowed", "You can't complete an order that is in the " + order.getStatus() + " status"));
}
```

`Order` 상태가 적절한 상태가 아닌 이상 주문을 완료하지 못하도록 방지하는 비슷한 로직이 구현되었습니다.

`LoadDatabase`에 다음과 같이 짧은 초기화 코드를 넣는다면:

`Updating the database pre-loader`
```java
orderRepository.save(new Order("MacBook Pro", Status.COMPLETED));
orderRepository.save(new Order("iPhone", Status.IN_PROGRESS));

orderRepository.findAll().forEach(order -> {
  log.info("Preloaded " + order);
});
```

이제 테스트를 할 수 있어요!

새로 단장된 주문 서비스를 사용하기 위해서는 그냥 몇 개의 동작을 수행하세요:

```console
$ curl -v http://localhost:8080/orders

{
  "_embedded": {
    "orderList": [
      {
        "id": 3,
        "description": "MacBook Pro",
        "status": "COMPLETED",
        "_links": {
          "self": {
            "href": "http://localhost:8080/orders/3"
          },
          "orders": {
            "href": "http://localhost:8080/orders"
          }
        }
      },
      {
        "id": 4,
        "description": "iPhone",
        "status": "IN_PROGRESS",
        "_links": {
          "self": {
            "href": "http://localhost:8080/orders/4"
          },
          "orders": {
            "href": "http://localhost:8080/orders"
          },
          "cancel": {
            "href": "http://localhost:8080/orders/4/cancel"
          },
          "complete": {
            "href": "http://localhost:8080/orders/4/complete"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:8080/orders"
    }
  }
}
```

이 HAL 문서는 각 주문의 현재 상태에 따라 다른 링크를 곧바로 보여줍니다.

- 첫 번째 주문은 `COMPLETED` 상태이고 탐색 링크만 가지고 있습니다. 상태 전이 링크들은 안보여집니다.
- 두 번째 주문은 `IN_PROGRESS` 상태이고 `cancel` 링크와 `complete` 링크를 추가적으로 갖고 있습니다.

주문 취소를 해볼까요:

```console
$ curl -v -X DELETE http://localhost:8080/orders/4/cancel

> DELETE /orders/4/cancel HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200
< Content-Type: application/hal+json;charset=UTF-8
< Transfer-Encoding: chunked
< Date: Mon, 27 Aug 2018 15:02:10 GMT
<
{
  "id": 4,
  "description": "iPhone",
  "status": "CANCELLED",
  "_links": {
    "self": {
      "href": "http://localhost:8080/orders/4"
    },
    "orders": {
      "href": "http://localhost:8080/orders"
    }
  }
}
```

이 응답은 HTTP 200 상태 코드를 보여주는데, 성공했다는 의미입니다. 응답 HAL 문서는 이 주문이 새로운 상태인 `CANCELLED`가 되었다는걸 보여주네요. 그리고 상태 전이 링크는 사라졌습니다.

같은 동작을 다시 한다면...

```console
$ curl -v -X DELETE http://localhost:8080/orders/4/cancel

* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> DELETE /orders/4/cancel HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 405
< Content-Type: application/hal+json;charset=UTF-8
< Transfer-Encoding: chunked
< Date: Mon, 27 Aug 2018 15:03:24 GMT
<
{
  "logref": "Method not allowed",
  "message": "You can't cancel an order that is in the CANCELLED status"
}
```

... HTTP 405 Method Not Allowed 응답을 보게 됩니다. DELETE이 이제 유효하지 않은 동작이 되었어요. `VndError` 응답 객체는 이미 `CANCELLED` 상태인 주문을 취소할 수 없다고 분명히 나타내고 있습니다.

추가적으로, 같은 주문을 완료 처리 하는 것도 실패하게 됩니다:

```console
$ curl -v -X PUT localhost:8080/orders/4/complete

* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> PUT /orders/4/complete HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 405
< Content-Type: application/hal+json;charset=UTF-8
< Transfer-Encoding: chunked
< Date: Mon, 27 Aug 2018 15:05:40 GMT
<
{
  "logref": "Method not allowed",
  "message": "You can't complete an order that is in the CANCELLED status"
}
```

이제 모든 것이 갖춰졌고, 당신의 주문 이행 서비스는 어떤 동작이 가능한지 조건부로 보여줄 수 있게 되었습니다. 유효하지 않은 동작에 대한 방어도 되었네요.

하이퍼미디어와 링크 프로토콜을 사용함으로써, 클라이언트는 더욱 견고해졌고, 단순히 테이터에 변경이 있었다고 해서 장애가 나타날 확률이 적어졌습니다. Spring HATEOAS는 클라이언트에 서빙해야 하는 하이퍼미디어를 만드는 것을 쉽게 해주었습니다.

# 요약

이 튜토리얼을 통해 REST API를 반들기 위한 여러 전략을 체험해봤습니다. REST는 그냥 pretty URI나 XML 대신 JSON을 리턴하는 개념이 아니란 걸 알게 되었습니다.






[1]: https://spring.io/guides/tutorials/rest/

---
aliases: []
type:
  - Lecture
tags:
  - 3-2/웹서비스프로그래밍
draft: false
date: 2026-10-30
prev:
  - "[[11_리스너와 필터]]"
next:
  - "[[13_스프링 프레임워크]]"
---
> HTTP 메서드와 자원 중심 설계를 따르는 REST API를 설계하고, JSON을 반환하는 엔드포인트를 구현한다.

## 이번 장의 핵심
**REST API**
- HTTP로 자원에 접근하고, 처리 결과를 JSON 등의 데이터로 주고받는 인터페이스다.

이 장에서는 JAX-RS로 API를 만들고
Postman으로 요청을 보내며,
기존 뉴스 서비스의 DAO를 재사용한다.

학습 목표는 다음과 같다.
- REST에서 자원과 표현이 무엇인지 설명한다.
- HTTP 메서드와 URI를 보고 API의 기능을 파악한다.
- JAX-RS 어노테이션으로 경로, 요청 방식, 데이터 형식을 지정한다.
- Postman으로 조회·등록·삭제를 테스트한다.
- 기존 뉴스 서비스를 데이터 중심의 API로 확장한다.

전체 처리 흐름:
```
브라우저·앱·Postman
→ HTTP 요청: 메서드 + URL + 헤더 + 필요한 본문
→ JAX-RS: 경로와 메서드에 맞는 Java 메서드 실행
→ DAO: 데이터베이스 조회·변경
→ Java 객체·목록 반환
→ JSON 등의 HTTP 응답
```

## 1. REST API
> [[REST API]]

### 1.1. 등장 배경
1. 전용 클라이언트·서버 구조에서는 통신 규격을 직접 정의해야 했고 서비스 간 호환도 어려웠다.
2. HTTP와 기존 웹 서버 기술을 이용하면 공통된 통신 기반을 활용할 수 있다.
3. 화면 전체보다 데이터만 필요한 서비스가 늘면서 경량 데이터 형식인 JSON이 널리 활용되었다.
4. 데이터 API를 이용하면 웹과 모바일 앱에서 같은 서버 기능을 사용할 수 있다.

### 1.2. 화면을 받는 방식과 데이터를 받는 방식
| 구분        | 화면 중심의 웹 서비스 | 이 장에서 만드는 REST API      |
| --------- | ------------ | ----------------------- |
| 응답의 중심    | HTML 화면      | JSON 데이터                |
| 클라이언트의 역할 | 받은 화면 표시     | 받은 데이터를 이용해 화면·기능 구성    |
| 활용        | 웹 페이지 제공     | 웹·모바일 앱 등에서 공통으로 데이터 사용 |

예를 들어 `GET /jspbook/api/news`를 요청하면 뉴스 화면 대신 다음과 같은 데이터를 받는다.
```json
[
  { "aid": 6, "title": "뉴스 제목" },
  { "aid": 7, "title": "다른 뉴스 제목" }
]
```

### 1.3. REST API 구성 요소
| 구성 요소  | -            | 예                            |
| ------ | ------------ | ---------------------------- |
| **자원** | **URI**      | 뉴스 기사                        |
| **행위** | **HTTP 메서드** | POST, GET, PUT/PATCH, DELETE |
| **표현** | **데이터 형식**   | JSON, XML                    |

#### 자원과 행위
> URI는 대상, 메서드는 동작
> URI에는 “무엇을”, HTTP 메서드에는 “어떻게 처리할지”를 담는다.

| HTTP 메서드 | 의미    | SQL과의 개념적 대응 | 뉴스 서비스 예    |
| -------- | ----- | ------------ | ----------- |
| GET      | 조회    | SELECT       | 뉴스 목록·상세 조회 |
| POST     | 생성    | INSERT       | 뉴스 등록       |
| PUT      | 전체 수정 | UPDATE       | 기사 전체 수정    |
| PATCH    | 일부 수정 | UPDATE       | 제목만 수정      |
| DELETE   | 삭제    | DELETE       | 특정 기사 삭제    |

> SQL과의 대응은 이해를 위한 비유다.
> API 요청이 반드시 특정 SQL 하나에 대응하는 것은 아니다.
> 이번 뉴스 실습은 등록·조회·삭제를 구현하며, 수정 API는 구현하지 않는다.

같은 경로도 HTTP 메서드가 다르면 다른 기능을 실행한다.
```
GET    /jspbook/api/news/21  → 21번 뉴스 조회
DELETE /jspbook/api/news/21  → 21번 뉴스 삭제
```

#### 표현
같은 자원을 서로 다른 형식으로 표현할 수 있다.

```json
{ "aid": 6, "title": "손흥민 또 멀티골" }
```

```xml
<news>
  <aid>6</aid>
  <title>손흥민 또 멀티골</title>
</news>
```

### 1.4. REST API URI 설계 원칙
| 원칙               | 권장 예                     | 피할 예                                  |
| ---------------- | ------------------------ | ------------------------------------- |
| 동사보다 명사 사용       | `/products`              | `/getProductList`                     |
| 집합에는 복수 명사 권장    | `/students`              | `/studentAll`                         |
| 상태 변경에 GET 사용 금지 | `DELETE /products/1021`  | `GET /products?action=delete&id=1021` |
| 관계는 하위 자원으로 표현   | `/depts/CE/students/홍길동` | 관계가 드러나지 않는 경로                        |
| 오류 정보를 의미 있게 제공  | 상황에 맞는 상태 코드와 오류 정보      | 모든 오류를 단순한 500으로 처리                   |

## 2. JAX-RS
**JAX-RS** (Java API for RESTful Web Services)
- Java에서 REST 서비스를 구현하기 위한 표준 API다.
- 인터페이스와 어노테이션으로 요청 처리 규칙을 선언한다.

- JAX-RS 자체는 인터페이스와 어노테이션 모음인 명세일 뿐, 실제로 동작하게 하는 엔진은 **구현체**이다.
- 이 장에서는 구현체로 **Jersey**를 사용한다.

| 작업 | 서블릿 방식 | JAX-RS 방식 |
|---|---|---|
| 주소 연결 | `@WebServlet` | `@Path` |
| 요청 메서드 구분 | `doGet()`, `doPost()` | `@GET`, `@POST` |
| 쿼리 파라미터 읽기 | `request.getParameter("msg")` | `@QueryParam("msg")` |
| 응답 작성 | 응답 객체의 Writer 사용 | 메서드에서 값 반환 |
| JSON 응답 | 직접 작성하거나 변환 도구 사용 | JSON 변환 제공자를 통해 객체·List 변환 |

### 2.1. 어노테이션
| 어노테이션                      | 역할                 | 예                                       |
| -------------------------- | ------------------ | --------------------------------------- |
| `@ApplicationPath`         | API 전체의 기본 경로      | `@ApplicationPath("/api")`              |
| `@Path`                    | 클래스·메서드의 경로        | `@Path("/news")`                        |
| `@GET`, `@POST`, `@DELETE` | 처리할 HTTP 메서드       | 조회 메서드에 `@GET`                          |
| `@PathParam`               | URL 경로의 변수에서 값 추출  | `/news/21`의 `21`                        |
| `@QueryParam`              | URL의 쿼리 문자열에서 값 추출 | `?msg=Java`의 `msg`                      |
| `@Consumes`                | **요청 본문**으로 받을 형식  | `@Consumes(MediaType.APPLICATION_JSON)` |
| `@Produces`                | **응답** 데이터 형식      | `@Produces(MediaType.APPLICATION_JSON)` |

### 2.2. 전체 URL이 만들어지는 과정
```
http://localhost:8080 + /jspbook + /api + /news + /21
서버 주소              프로젝트   API 기본  클래스  메서드
                       경로       경로      경로    경로
```

- `/jspbook`: 프로젝트의 컨텍스트 경로
- `/api`: `RestConfig`의 `@ApplicationPath`
- `/news`: 서비스 클래스의 `@Path`
- `/21`: 메서드의 `@Path("{aid}")`에 해당

메서드에 별도의 `@Path`가 없으면 클래스 경로에서 요청을 처리한다.

## 실습 12-1: 기본 API 만들기
### 1. 라이브러리 준비
`pom.xml`의 `dependencies`에 다음 세 의존성을 추가한다.
**아래 버전은 강의 기준인 `3.1.9`다.**

| groupId | artifactId | 역할 |
|---|---|---|
| `org.glassfish.jersey.containers` | `jersey-container-servlet` | 서블릿 컨테이너에서 Jersey 실행 |
| `org.glassfish.jersey.inject` | `jersey-hk2` | Jersey의 의존성 주입 지원 |
| `org.glassfish.jersey.media` | `jersey-media-json-jackson` | Java 객체와 JSON 간 변환 |

세 라이브러리의 버전을 맞추고 Maven 의존성을 갱신한다.
전이 의존성 때문에 실제 다운로드되는 라이브러리는 세 개보다 많다.

### 2. API 등록: RestConfig
```java
package ch12;

import java.util.HashMap;
import java.util.Map;
import jakarta.ws.rs.ApplicationPath;
import jakarta.ws.rs.core.Application;

@ApplicationPath("/api")
public class RestConfig extends Application {
    @Override
    public Map<String, Object> getProperties() {
        Map<String, Object> properties = new HashMap<>();
        properties.put("jersey.config.server.provider.packages", "ch12");
        return properties;
    }
}
```

읽는 순서:
1. `Application`을 상속해 설정 클래스를 만든다.
2. API 기본 경로를 `/api`로 지정한다.
3. Jersey가 API 클래스를 검색할 패키지를 `ch12`로 설정한다.

> 패키지 검색 속성은 Jersey 설정이다.
> 이번 설정에서는 `ch12`를 검색 대상으로 등록한다.

### 3. GET과 POST를 처리하는 서비스
```java
package ch12;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;

@Path("/test")
public class RestApiExample {
    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String sayHello() {
        return "Hello API Service";
    }

    @POST
    public String sayHello(@QueryParam("msg") String msg) {
        return msg + " API Service";
    }
}
```

| 요청 | 입력 | 예상 응답 |
|---|---|---|
| `GET /jspbook/api/test` | 없음 | `Hello API Service` |
| `POST /jspbook/api/test?msg=Java%20Web` | 쿼리 파라미터 `msg` | `Java Web API Service` |

**주의**
- POST 요청이라고 해서 모든 입력이 본문에 들어가는 것은 아니다.
- 이 예제는 `@QueryParam`을 쓰므로 URL의 쿼리 문자열을 읽는다.

## 3. REST 클라이언트 (Postman)
**REST 클라이언트**
- API를 사용하는 프로그램이다.
- HTTP 통신이 가능하면 언어와 무관하게 구현할 수 있다.

```
요청 구성 → 서버로 전송 → 응답 수신 → JSON 등을 객체로 변환 → 프로그램에서 사용
```

| 방법 | 용도·특징                       |
|---|---|
| 브라우저 주소창 | 간단한 GET 조회 확인               |
| Java `HttpURLConnection` | 기본 기능으로 구현 가능하지만 처리 코드가 길어짐 |
| Apache HttpClient | 라이브러리로 요청·응답 처리를 간결하게 작성    |
| curl | 명령줄에서 API 테스트               |
| Postman | GUI로 메서드·파라미터·본문 등을 설정해 테스트 |

### 3.1. API 테스트
1. 톰캣을 실행한다.
2. Postman에서 요청을 만든다.
3. GET과 `http://localhost:8080/jspbook/api/test`를 설정한다.
4. Send를 눌러 응답 본문을 확인한다.
5. POST로 변경한다.
6. Params에 `msg` = `Java Web`을 입력한다.
7. Send를 눌러 `Java Web API Service`가 반환되는지 확인한다.

**메서드, URL, Params, Body, 응답**의 역할을 중심으로 이해한다.

## 실습 12-3: 뉴스 REST API
### 1. 기존 코드를 어떻게 재사용하는가?
10장에서 만든 `NewsDAO`와 `NewsDO`를 사용한다.

```
웹 또는 모바일 클라이언트
  → NewsApiService: HTTP 요청을 Java 메서드에 연결
  → NewsDAO: 데이터베이스 작업
  → NewsDO 또는 List<NewsDO>: 결과 데이터
  → JSON 응답
```

API 클래스는 `ch12` 패키지에 만들고 `ch10.NewsDAO`, `ch10.NewsDO`를 가져온다.
생성자에서 `NewsDAO`를 초기화한다. 실행 전에는 MySQL과 톰캣이 모두 준비되어 있어야 한다.

### 2. API 명세 한눈에 보기
기본 주소: `http://localhost:8080/jspbook/api`

| 기능 | 메서드 | 경로 | 요청 데이터 | 구현의 응답          |
|---|---|---|---|---|
| 전체 목록 | GET | `/news` | 없음 | 뉴스 객체들의 JSON 배열 |
| 상세 조회 | GET | `/news/{aid}` | 경로의 기사 ID | 뉴스 한 건의 JSON 객체 |
| 뉴스 등록 | POST | `/news` | JSON 본문 | 성공·실패 문자열       |
| 뉴스 삭제 | DELETE | `/news/{aid}` | 경로의 기사 ID | 성공·실패 문자열       |

### 3. 뉴스 등록
코드의 핵심 부분을 발췌하면 다음과 같다.
아래 메서드들은 생성자에서 `dao`가 초기화된 `NewsApiService` 클래스 안에 작성한다.

```java
@POST
@Consumes(MediaType.APPLICATION_JSON)
public String addNews(NewsDO news) {
    try {
        dao.addNews(news);
    } catch (Exception e) {
        e.printStackTrace();
        return "News API: 뉴스 등록 실패!!";
    }
    return "News API: 뉴스 등록됨!!";
}
```

처리 흐름:
```
JSON 본문 → NewsDO 객체로 변환 → dao.addNews(news) → 결과 문자열
```

Postman에서 POST `/news`를 선택한 뒤 **Body → raw → JSON**으로 본문을 입력한다.

```json
{
  "title": "REST API 학습 뉴스",
  "img": "https://example.com/news-image.jpg",
  "date": "2025.09.28 11:06:34",
  "content": "JAX-RS로 뉴스 API를 구현했습니다."
}
```

- 위 데이터는 학습용 예시이며 이미지 URL은 자리표시자다.
- `aid`는 DB에서 자동으로 처리하므로 요청 본문에서 생략한다.
- 이미지 파일 자체 대신 이미지 URL을 보낸다.
- 요청 `Content-Type`이 `application/json`인지 확인한다.
- 날짜 문자열은 기존 DAO와 DB가 처리하는 형식에 맞춘다.

### 4. 뉴스 삭제
```java
@DELETE
@Path("{aid}")
public String delNews(@PathParam("aid") int aid) {
    try {
        dao.delNews(aid);
    } catch (SQLException e) {
        e.printStackTrace();
        return "News API: 뉴스 삭제 실패!! - " + aid;
    }
    return "News API: 뉴스 삭제됨!! - " + aid;
}
```

`DELETE /news/21`이면 `{aid}`의 문자열 값이 정수형 인자 `aid`로 전달되고 `dao.delNews(21)`이 실행된다.
같은 주소에 GET을 보내면 삭제가 아니라 상세 조회 메서드가 선택된다.

### 5. 뉴스 목록 조회
```java
@GET
@Produces(MediaType.APPLICATION_JSON)
public List<NewsDO> getNewsList() {
    List<NewsDO> newsList = null;
    try {
        newsList = dao.getAll();
    } catch (Exception e) {
        e.printStackTrace();
    }
    return newsList;
}
```

**`List<NewsDO>` → JSON 배열**로 변환된다.
객체를 JSON으로 변환할 제공자가 설정되어 있어야 한다.
이번 실습은 `jersey-media-json-jackson`으로 이 기능을 준비한다.

### 6. 뉴스 상세 조회
```java
@GET
@Path("{aid}")
@Produces(MediaType.APPLICATION_JSON)
public NewsDO getNews(@PathParam("aid") int aid) {
    NewsDO news = null;
    try {
        news = dao.getNews(aid);
    } catch (SQLException e) {
        e.printStackTrace();
    }
    return news;
}
```

**`NewsDO` 한 개 → JSON 객체**로 변환된다.

### 7. 권장 실습 순서
1. **환경 확인**: MySQL, 기존 뉴스 테이블·DAO, 톰캣이 준비되어 있는지 확인한다.
2. **목록 조회**: GET `/news`로 현재 기사와 ID를 확인한다.
3. **등록**: POST `/news`에 JSON 본문을 보낸다.
4. **목록 재조회**: 새 기사와 실제 생성된 `aid`를 확인한다.
5. **상세 조회**: GET `/news/{실제 aid}`로 등록 내용을 확인한다.
6. **삭제**: DELETE `/news/{실제 aid}`를 보낸다.
7. **삭제 검증**: GET `/news`로 해당 기사가 목록에서 사라졌는지 확인한다.

> 성공 문자열만 확인하지 말고 조회 결과로 실제 데이터 변경을 확인한다.
> 코드의 문자열 반환만으로는 존재하지 않는 ID의 처리 결과나 HTTP 오류 상태를 충분히 구분하기 어렵다.

### 설계 원칙과 실습 구현의 차이
적절한 HTTP 상태 코드와 오류 정보 제공을 권장하지만, 뉴스 실습은 학습 편의를 위해 성공·실패 문자열을 반환한다. 조회 예외도 로그만 출력하고 `null`을 반환한다.

> [[HTTP 상태 코드]]

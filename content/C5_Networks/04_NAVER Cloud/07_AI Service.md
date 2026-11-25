---
aliases: []
type:
  - Lecture
tags:
  - 2-2/NAVER-Cloud
draft: false
date: 2025-11-29
prev:
  - "[[06_Docker]]"
next:
  - "[[A1_IT infrastructure]]"
---
> [!summary] 핵심 요약
> - **API**는 소프트웨어 간 상호작용 인터페이스이며, 현대 웹·클라우드 AI 서비스는 주로 HTTP와 JSON 기반의 **REST API**로 제공된다.
> - 네이버 클라우드는 음성·시각 지능(**CLOVA Dubbing · Voice · Speech · OCR**)과 인공신경망 번역(**Papago Text · Website · Image Translation**)을 API 형태로 제공한다.

## 1. API와 REST API 이론
### 1.1. API 정의 및 주요 유형 비교
**API (Application Programming Interface)**는 서로 다른 소프트웨어·시스템 간에 정해진 규격으로 데이터를 주고받게 해 주는 인터페이스이다.

| 유형 | 기반 기술 · 포맷 | 핵심 특징 |
| :--- | :--- | :--- |
| **REST API** | HTTP / 주로 JSON | 리소스(URL) 중심 설계, 현대 웹·클라우드에서 가장 널리 사용 |
| **SOAP API** | XML 기반 프로토콜 | 엄격한 규격과 보안·트랜잭션 처리가 필요한 환경에 사용 |
| **GraphQL** | 쿼리 언어 / JSON | 클라이언트가 필요한 데이터 필드(서브셋)만 정확히 지정해 요청 |

### 1.2. REST API 기본 구조
- **기반 프로토콜**: HTTP
- **주요 HTTP 메서드**: `GET`(조회), `POST`(생성), `PUT`(수정), `DELETE`(삭제)
- **데이터 포맷**: 주로 **JSON** (JavaScript Object Notation)
- **핵심 구성 요소**:
	- **리소스 (Resource)**: API가 다루는 데이터나 서비스 대상
	- **엔드포인트 (Endpoint)**: 특정 리소스에 접근하기 위한 고유 URL 경로

### 1.3. REST의 6가지 아키텍처 원칙
1. **무상태성 (Statelessness)**: 서버는 클라이언트의 이전 요청 상태를 저장하지 않으며, 모든 요청은 독립적으로 처리된다.
2. **클라이언트-서버 구조**: UI(클라이언트)와 데이터 저장·처리(서버)의 책임을 분리해 독립적으로 발전한다.
3. **캐시 가능성 (Cacheable)**: 응답 데이터의 캐시 가능 여부를 명시해 성능을 향상한다.
4. **일관된 인터페이스 (Uniform Interface)**: URL과 표준 HTTP 메서드로 리소스를 일관되게 조작한다.
5. **계층적 시스템 (Layered System)**: 클라이언트는 최종 서버 앞에 프록시·게이트웨이·로드 밸런서 등 중간 계층이 있어도 동일하게 통신한다.
6. **코드 온 디맨드 (Code on Demand, 선택적)**: 필요 시 서버가 실행 가능한 코드를 클라이언트로 전송해 기능을 확장한다.

### 1.4. HTTP 요청 · 응답과 인증 방식
- **요청/응답 구조**:
	- **Header**: 메타데이터 포함 — 데이터 형식(`Content-Type`), 인증 자격(`Authorization`), 클라이언트 정보(`User-Agent`) 등
	- **Body**: 실제 전송 페이로드 (주로 `POST`, `PUT`에서 JSON/XML 또는 바이너리 전송)
	- **상태 코드**: 처리 결과를 나타내는 숫자 코드 → [[HTTP 상태 코드]]

| API 인증 방식 | 원리 및 특징 |
| :--- | :--- |
| **API Key (기본 인증)** | 고유한 문자열 키(Client ID/Secret 등)로 호출자를 식별하는 단순하고 직관적인 방식 |
| **OAuth 2.0** | 제3자 애플리케이션에 안전하게 권한을 위임하는 표준 토큰 기반 인가·인증 프레임워크 |
| **JWT (JSON Web Token)** | 사용자·권한 정보를 JSON으로 인코딩하고 서명해 안전하게 전달하는 자가 수용적(Self-contained) 토큰 |

## 2. NAVER Cloud AI Services
### 2.1. CLOVA AI Services (음성 · 시각 지능)
| 서비스 | 핵심 기능 | 주요 특징 · 파라미터 |
| :--- | :--- | :--- |
| **CLOVA Dubbing** | 텍스트를 AI 보이스로 변환해 동영상에 더빙 | 화자 선택 및 영상 음성 합성 |
| **CLOVA Voice (TTS)** | 텍스트 → 자연스러운 음성 변환 (Text to Speech) | 화자(Speaker), 속도, 볼륨, 피치, 감정 조절 및 mp3/wav 생성 |
| **CLOVA Speech (STT)** | 음성 → 텍스트 변환 (Speech to Text) | 장문 음성 인식, **화자 분리**, **키워드 부스팅** 지원 (Secret Key · Invoke URL 호출) |
| **CLOVA OCR** | 이미지 내 문자 감지 및 텍스트 추출 | • **General 모드**: 이미지 전체 텍스트 추출<br>• **Template 모드**: 지정된 양식의 특정 필드 값 추출<br>• 이미지 URL 또는 **Base64** 인코딩 전송 |

### 2.2. Papago Translation Services (인공신경망 번역)
| 서비스 | 핵심 기능 | 주요 특징 |
| :--- | :--- | :--- |
| **Text Translation** | 문장·텍스트 인공신경망 번역 | 언어 자동 감지, 높임말 옵션, **용어집(Glossary)** 적용 |
| **Website Translation** | 웹페이지 HTML 구조를 유지하며 내용 번역 | 태그 구조를 보존하면서 텍스트 노드만 번역 |
| **Image Translation** | 이미지 속 텍스트를 인식·번역 | • **Image to Text**: 이미지 내 텍스트 추출 후 번역문 반환<br>• **Image to Image (AR 번역)**: 원본 이미지의 텍스트를 번역문으로 시각 대체 |

## 3. AI API 연동 시 주요 데이터 · 프로토콜 개념
| 개념 / 도구 | 역할 |
| :--- | :--- |
| **JSON** | 텍스트 기반의 구조화된 API 요청·응답 표준 포맷 |
| **Base64** | 이미지·음성 등 바이너리 데이터를 텍스트 문자열로 인코딩해 JSON 페이로드로 주고받는 방식 (OCR 요청, Image-to-Image 응답 디코딩 등) |
| **`multipart/form-data`** | HTTP 요청으로 바이너리 파일(이미지 등)과 메타데이터를 함께 전송할 때 사용하는 콘텐츠 타입 |
| **연동 생태계 도구** | `Requests`(HTTP API 호출), `BeautifulSoup`(HTML 파싱), `Playwright`(브라우저 자동화·웹 스크린샷 캡처), `Gradio`(AI API용 경량 웹 데모 UI 구축) |

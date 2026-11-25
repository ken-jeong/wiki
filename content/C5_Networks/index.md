---
aliases:
  - C5_Networks
type:
  - Index
tags: []
draft: false
date: 2026-11-18
prev:
next:
---
# C5_Networks
> 네트워크 통신의 원리와 클라우드 환경에서의 활용을 정리한다.

> [!note]- Network Programming
> - [[C5_Networks/01_Network Programming/01_네트워크 프로그래밍과 소켓의 이해|01_네트워크 프로그래밍과 소켓의 이해]]
> - [[C5_Networks/01_Network Programming/02_소켓의 타입과 프로토콜의 설정|02_소켓의 타입과 프로토콜의 설정]]
> - [[C5_Networks/01_Network Programming/03_주소체계와 데이터 정렬|03_주소체계와 데이터 정렬]]
> - [[C5_Networks/01_Network Programming/04_TCP-based Server & Client 1|04_TCP-based Server & Client 1]]
> - [[C5_Networks/01_Network Programming/05_TCP-based Server & Client 2|05_TCP-based Server & Client 2]]
> - [[C5_Networks/01_Network Programming/06_UDP-based Server & Client|06_UDP-based Server & Client]]
> - [[C5_Networks/01_Network Programming/07_Half-close the socket|07_Half-close the socket]]
> - [[C5_Networks/01_Network Programming/08_Domain Name & IP Address|08_Domain Name & IP Address]]
> - [[C5_Networks/01_Network Programming/09_Socket Options|09_Socket Options]]
> - [[C5_Networks/01_Network Programming/12_IO 멀티플렉싱|12_IO 멀티플렉싱]]
> - [[C5_Networks/01_Network Programming/14_Multicast & Broadcast|14_Multicast & Broadcast]]
> - [[C5_Networks/01_Network Programming/19_Using Threads in Windows|19_Using Threads in Windows]]
> - [[C5_Networks/01_Network Programming/20_Windows에서의 스레드 동기화|20_Windows에서의 스레드 동기화]]
> - [[C5_Networks/01_Network Programming/21_비동기 Notification IO 모델|21_비동기 Notification IO 모델]]
> - [[C5_Networks/01_Network Programming/22_Overlapped IO 모델|22_Overlapped IO 모델]]
> - [[C5_Networks/01_Network Programming/23_IOCP (IO Completion Port)|23_IOCP (IO Completion Port)]]
> - [[C5_Networks/01_Network Programming/24_HTTP Server|24_HTTP Server]]

![Winsock](https://img.shields.io/badge/Winsock-0078D4?logo=windows11&logoColor=white)

> [!note]- Computer Networks
> 네트워크 구조와 전송매체, 계층별 프로토콜을 하향식으로 학습하고 소켓 프로그램으로 메시지 전송 과정을 확인한다.
> - [[C5_Networks/02_Computer Networks/01-1_인터넷, 네트워크 에지|01-1_인터넷, 네트워크 에지]]
> - [[C5_Networks/02_Computer Networks/01-2_네트워크 코어, 패킷 지연|01-2_네트워크 코어, 패킷 지연]]
> - [[C5_Networks/02_Computer Networks/01-3_프로토콜 계층, 보안, 역사|01-3_프로토콜 계층, 보안, 역사]]
> - [[C5_Networks/02_Computer Networks/02-1_네트워크 애플리케이션 원리|02-1_네트워크 애플리케이션 원리]]
> - [[C5_Networks/02_Computer Networks/02-2_웹, HTTP|02-2_웹, HTTP]]
> - [[C5_Networks/02_Computer Networks/02-3_이메일, DNS|02-3_이메일, DNS]]
> - [[C5_Networks/02_Computer Networks/02-4_P2P, 스트리밍, CDN|02-4_P2P, 스트리밍, CDN]]
> - [[C5_Networks/02_Computer Networks/03-1_전송 계층 서비스, 다중화|03-1_전송 계층 서비스, 다중화]]
> - [[C5_Networks/02_Computer Networks/03-2_UDP|03-2_UDP]]
> - [[C5_Networks/02_Computer Networks/03-3_오류 제어|03-3_오류 제어]]
> - [[03-4_TCP 및 흐름 제어|03-4_흐름 제어]]
> - [[C5_Networks/02_Computer Networks/03-5_혼잡 제어|03-5_혼잡 제어]]
> - [[C5_Networks/02_Computer Networks/04-1_네트워크 계층, 라우터 구조|04-1_네트워크 계층, 라우터 구조]]
> - [[C5_Networks/02_Computer Networks/04-2_인터넷 프로토콜|04-2_인터넷 프로토콜]]
> - [[04-3_데이터 평면|04-3_데이터 평면]]

| 영역     | 서비스        | AWS                                             | NCP                                                   |
| ------ | ---------- | ----------------------------------------------- | ----------------------------------------------------- |
| 컴퓨팅    | 가상 서버      | [[03_가상 서버 (AWS EC2)\|EC2]]                     | [[01-2_가상 서버 (NCP Server)\|Server]]                   |
| 〃      | 오토 스케일링    | [[06_오토 스케일링 (AWS Auto Scaling)\|Auto Scaling]] | [[02-3_오토 스케일링 (NCP Auto Scaling)\|Auto Scaling]]     |
| 〃      | 서버리스 함수    | [[14_서버리스 함수 (AWS Lambda)\|Lambda]]             | Cloud Functions                                       |
| 〃      | 관리형 K8s    | EKS                                             | [[05-2_K8s (NCP NKS)\|NKS]]                           |
| 스토리지   | 객체 스토리지    | [[08_객체 스토리지 (AWS S3)\|S3]]                     | [[03-2_객체 스토리지 (NCP Object Storage)\|Object Storage]] |
| 〃      | 아카이브       | S3 Glacier                                      | Archive Storage                                       |
| 〃      | 블록 스토리지    | [[04_블록 스토리지 (AWS EBS)\|EBS]]                   | [[01-3_블록 스토리지 (NCP Block Storage)\|Block Storage]]   |
| 〃      | 공유 파일 시스템  | EFS                                             | [[01-4_공유 파일 시스템 (NCP NAS)\|NAS]]                     |
| 네트워크   | 리전         | 예: `ap-northeast-2`                             | 예: `KR`                                               |
| 〃      | 가상 사설 네트워크 | VPC                                             | VPC                                                   |
| 〃      | 가용 영역      | AZ (예: `a`/`b`/`c`/`d`)                         | Zone (예: `1`/`2`)                                     |
| 〃      | 서브넷        | Subnet                                          | Subnet                                                |
| 〃      | 로드 밸런서     | [[05_로드 밸런서 (AWS ELB)\|ELB]]                    | [[02-2_로드 밸런서 (NCP Load Balancer)\|Load Balancer]]    |
| 〃      | DNS        | [[10_DNS (AWS Route 53)\|Route 53]]             | Global DNS                                            |
| 〃      | CDN        | [[09_CDN (AWS CloudFront)\|CloudFront]]         | Global Edge                                           |
| 〃      | NAT 게이트웨이  | NAT Gateway                                     | NAT Gateway                                           |
| 데이터베이스 | RDB        | [[07_RDB (AWS RDS)\|RDS]]                       | [[04_DB (NCP Cloud DB)\|Cloud DB]]                    |
| 〃      | NoSQL DB   | [[13_NoSQL DB (AWS DynamoDB)\|DynamoDB]]        | [[04_DB (NCP Cloud DB)\|Cloud DB]]                    |
| 보안     | 계정 및 권한 관리 | [[11_계정 및 권한 관리 (AWS IAM)\|IAM]]                | [[03-1_계정 및 권한 관리 (NCP Sub Account)\|Sub Account]]    |
| 〃      | 서버 단위 방화벽  | Security Group                                  | ACG                                                   |
| 〃      | 서브넷 단위 방화벽 | Network ACL                                     | Network ACL                                           |
| 모니터링   | 메트릭 모니터링   | [[12_모니터링 (AWS CloudWatch)\|CloudWatch]]        | [[02-1_모니터링 (NCP Cloud Insight)\|Cloud Insight]]      |
| 개발     | API 관리     | API Gateway                                     | API Gateway                                           |

> [!note]- Cloud Computing
> 클라우드 컴퓨팅의 개념과 가상화 기술을 이해하고, AWS 위에서 IaaS·PaaS·SaaS를 단계적으로 실습한다.
> - [[01_클라우드 컴퓨팅|01_클라우드 컴퓨팅]]
> - [[02_AWS|02_AWS]]
> - [[03_가상 서버 (AWS EC2)]]
> - [[04_블록 스토리지 (AWS EBS)]]
> - [[05_로드 밸런서 (AWS ELB)]]
> - [[06_오토 스케일링 (AWS Auto Scaling)]]
> - [[07_RDB (AWS RDS)]]
> - [[08_객체 스토리지 (AWS S3)]]
> - [[09_CDN (AWS CloudFront)]]
> - [[10_DNS (AWS Route 53)]]
> - [[11_계정 및 권한 관리 (AWS IAM)]]
> - [[12_모니터링 (AWS CloudWatch)]]
> - [[13_NoSQL DB (AWS DynamoDB)]]
> - [[14_서버리스 함수 (AWS Lambda)]]
> - [[15_SDK & CLI]]

![AWS](https://img.shields.io/badge/AWS-232F3E?logo=amazonwebservices&logoColor=white)

> [!note]- NAVER Cloud
> - [[01-1_네트워크 (VPC · Subnet)]]
> - [[01-2_가상 서버 (NCP Server)]]
> - [[01-3_블록 스토리지 (NCP Block Storage)]]
> - [[01-4_공유 파일 시스템 (NCP NAS)]]
> - [[02-1_모니터링 (NCP Cloud Insight)]]
> - [[02-2_로드 밸런서 (NCP Load Balancer)]]
> - [[02-3_오토 스케일링 (NCP Auto Scaling)]]
> - [[03-1_계정 및 권한 관리 (NCP Sub Account)]]
> - [[03-2_객체 스토리지 (NCP Object Storage)]]
> - [[03-3_클라우드 보안]]
> - [[04_DB (NCP Cloud DB)]]
> - [[05-1_Cloud Native]]
> - [[05-2_K8s (NCP NKS)]]
> - [[06_Docker]]
> - [[07_AI Service]]

![NAVER Cloud](https://img.shields.io/badge/NAVER_Cloud-03C75A?logo=naver&logoColor=white)

# 나만의 책장 서비스 (도서 관리 서비스)
[📄 포트폴리오 PDF 보기](.assets/KT%20AIVLE%20School%20Mini%20Project%205.pdf)

*Aivle School 8기 4차 미니 프로젝트*
1반 2조 

-----

## 🚀 프로젝트 소개

**나만의 책장 서비스**는 사용자가 직접 글을 작성해서 나만의 책을 만들고, AI를 이용해 표지 이미지를 생성하며, 외부 도서 정보(YES24)까지 검색할 수 있는 웹 서비스입니다.

본 프로젝트는 단순한 애플리케이션 개발에 그치지 않고, **AWS EKS(Elastic Kubernetes Service)를 기반으로 한 클라우드 네이티브 아키텍처**로 설계되었습니다. 무중단 배포, 자동 확장, 고가용성 보장 등 실제 서비스 운영 환경에 맞춘 인프라 구성과 CI/CD 파이프라인 구축에 중점을 두었습니다.

---

## 🏗️ 서비스 아키텍처 및 인프라 설계

![서비스 아키텍처](./.assets/system_architec.png)

단순한 기능 구현을 넘어, **실제 운영 환경에서 발생할 수 있는 트래픽 증가와 서버 장애에 유연하게 대응**할 수 있도록 AWS EKS(Kubernetes) 기반의 클라우드 네이티브 아키텍처를 설계했습니다.

### 1. 효율적인 트래픽 분산 (AWS Load Balancer & Nginx)
사용자 요청이 증가할 때 단일 서버에 부하가 집중되지 않도록 이중으로 분산 처리 구조를 짰습니다. 외부 진입점으로는 쿠버네티스의 `Service(Type: LoadBalancer)`를 선언하여 **AWS Load Balancer(NLB)**를 자동으로 띄워 외부 트래픽을 안전하게 받아냅니다. 이렇게 들어온 트래픽은 클러스터 내부의 **Nginx(L4/L7 라우터 역할)**로 전달되며, Nginx가 다시 백엔드 API와 프론트엔드 라우트로 나누어 전달(리버스 프록시)하도록 구성했습니다.

### 2. 단일 장애점 극복과 고가용성 (High Availability)
특정 데이터센터(가용 영역)에 시스템 장애가 발생하더라도 전체 서비스가 중단되지 않도록 방어하는 데 집중했습니다. `Topology Spread Constraints` 속성을 적용하여 파드(Pod)들을 여러 가용 영역에 분산 배치하였고, 새로운 코드를 배포할 때도 서버를 멈추지 않게 롤링 업데이트(Rolling Update) 기반의 **무중단 배포**를 적용했습니다.

### 3. 장애 발생 시 자동 복구 (Health Check & Self-Healing)
메모리 누수나 예상치 못한 에러로 애플리케이션이 멈출 경우, 수동 개입 없이 시스템이 스스로 정상화되는 데 초점을 맞췄습니다. 쿠버네티스의 `Liveness Probe`와 `Readiness Probe`를 활용해 주기적으로 헬스 체크를 진행하며, 응답 불가 상태의 파드는 스스로 재시작(Self-Healing)되도록 구성했습니다.

### 4. 트래픽 변화에 따른 유연한 자원 확장 (Auto Scaling)
이벤트 등으로 갑작스러운 트래픽 폭주가 발생해도 서비스가 버틸 수 있도록 유연한 인프라를 구축했습니다. Cluster Autoscaler(CA)를 구성하여 리소스가 한계에 다다르면 자동으로 서버(EC2 노드)를 늘려 대응(Scale-out)하고, 유휴 상태일 때는 자원을 회수하여 비용 효율성을 높였습니다.

### 5. 데이터 영속성과 보안 유지 (Stateful & Secret)
애플리케이션을 구동하는 파드는 언제든 생성/삭제될 수 있어 저장소로 적합하지 않습니다. 따라서 도서 및 사용자 정보를 안전하게 보관하기 위해 `StatefulSet`과 AWS EBS 볼륨을 결합하여 PostgreSQL 데이터베이스 환경을 구성했습니다. 또한 DB 접근 계정과 같은 민감한 정보는 소스코드에서 분리하여 쿠버네티스 `Secret`으로 안전하게 주입되도록 처리했습니다.

---

## 🔄 무중단 자동 배포 파이프라인 (CI/CD)

수동 배포 과정에서 발생할 수 있는 휴먼 에러를 방지하고, 개발 생산성을 극대화하기 위해 **AWS 완전 관리형 서비스(CodePipeline, CodeBuild, ECR)**를 활용한 배포 자동화를 달성했습니다.

**[ 파이프라인 동작 흐름 ]**

1. **Source**: GitHub `main` 브랜치에 코드가 병합(Push)되면 AWS CodePipeline이 이를 자동으로 감지하여 빌드/배포 프로세스를 트리거합니다.
2. **Build & Push**: AWS CodeBuild가 `buildspec.yml` 설정에 맞춰 프론트엔드와 백엔드를 각각 새로운 도커(Docker) 이미지로 빌드합니다. 생성된 이미지는 AWS ECR(프라이빗 컨테이너 레지스트리)로 안전하게 전송됩니다.
3. **Deploy**: ECR에 등록된 최신 이미지를 바탕으로 EKS 클러스터에 배포 명령(`kubectl apply`)을 수행하여 롤링 업데이트 방식으로 실제 운영 환경에 무중단 반영됩니다.

**[ 파이프라인 설계 주안점 ]**
- **인프라 운영 최소화**: Jenkins 등의 별도 CI/CD 서버 구성 없이 AWS 네이티브 서비스를 활용하여 관리 포인트를 대폭 줄였습니다.
- **안정적인 롤백 지원**: 코드 빌드 시점의 Git 커밋 해시(Commit Hash)를 이미지 태그로 사용하여 버전을 식별합니다. 배포 후 예기치 않은 오류가 발생할 경우 확정된 이전 버전 이미지로 신속하게 롤백이 가능합니다.

---

## 💻 어플리케이션 주요 기능 요약

### 1. 사용자 인증 및 관리
- 회원가입 (이메일, 비밀번호 정책, 중복 검증) 및 이메일+비밀번호 기반 로그인 구현 (세션 기반).

### 2. 나만의 책 만들기 (CRUD)
- 도서 등록, 전체/상세 조회, 수정 및 삭제 기능 제공.

### 3. AI 표지 이미지 생성
- 도서 상세 정보 화면에서 **OpenAI API** 연동을 통해 책 제목/내용 기반 프롬프트를 자동 생성하고 이미지를 생성하여 화면에 제공. 선택 시 백엔드 DB와 연동하여 커버 정보(`coverImageUrl`)를 업데이트.

### 4. 실시간 YES24 도서 검색 연동
- K-웹 스크래핑 라이브러리 `Jsoup`을 활용해 사용자가 입력한 키워드로 YES24 외부 쇼핑몰의 책 제목, 저자, 이미지를 파싱하여 실시간 검색 결과 제공.

---

## 🛠 기술 스택

### Infrastructure & DevOps
- **Cloud/Infra**: AWS EKS, AWS ECR, AWS LoadBalancer, IAM, EBS
- **CI/CD**: AWS CodePipeline, AWS CodeBuild (`buildspec.yml`)
- **Container Orchestration**: Kubernetes (Deployment, StatefulSet, ConfigMap, Secret, Service)
- **Auto Scaling**: Kubernetes Cluster Autoscaler

### Backend
- **Java 17, Spring Boot 3.5.8**, Spring Web, Spring Data JPA, Hibernate
- **Database**: PostgreSQL (EKS 환경), H2 (개발 환경)
- **Library**: Jsoup, Lombok, Gradle

### Frontend
- **Next.js (React 기반)**, React Hooks, MUI, Axios



# 프론트엔드 배포 파이프라인

## 개요

![deploy_architecture drawio](https://github.com/user-attachments/assets/10d9a462-8b0d-4d14-a958-252af1adcb48)

GitHub Actions에 워크플로우를 작성해 다음과 같이 배포가 진행되도록 합니다.

1. 저장소를 체크아웃합니다.
2. Node.js 18.x 버전을 설정합니다.
3. 프로젝트 의존성을 설치합니다.
4. Next.js 프로젝트를 빌드합니다.
5. AWS 자격 증명을 구성합니다.
6. 빌드된 파일을 S3 버킷에 동기화합니다.
7. CloudFront 캐시를 무효화합니다.

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://hanghaeplus.s3-website-ap-southeast-2.amazonaws.com/
- CloudFront 배포 도메인 이름: https://dgz74mnk6or08.cloudfront.net/

## 주요 개념

## Github Actions와 CI/CD 도구

- ## CI (Continuous Integration, 지속적 통합)
  - 여러 개발자의 코드 변경사항을 지속적으로 통합
  - 자동 빌드 및 테스트로 코드 품질 유지
  - 빠른 피드백으로 버그 조기 발견
- ## CD (Continuous Delivery/Deployment, 지속적 배포)
  - 지속적 제공(Delivery): 수동 승인 후 배포
  - 지속적 배포(Deployment): 자동으로 프로덕션 환경까지 배포
- ## Github Actions

  - Workflow (워크 플로우)

    - CI/CD 파이프라인 전체 과정
    - .github/workflows 디렉토리에 YAML 파일로 정의

  - Event (이벤트)

    - 워크플로우를 실행시키는 트리거
    - push, pull_request, schedule 등

  - Jobs (작업)

    - 독립적인 가상 머신이나 컨테이너에서 실행
    - 여러 step들로 구성

  - Steps (스텝)

    - job 내의 실행 단위
    - 명령이 실행이나 액션 사용

  - Actions (액션)
    - 워크플로우에서 사용할 수 있는 재사용 가능한 단위

## S3와 스토리지

- 버킷, 객체, 키로 구성되어 잇음
- 스토리지 클래스
  - Standard: 기본 스토리지 클래스, 높은 가용성, 빈번하게 액세스하는 데이터
  - Intelligent-Tiering: 액세스 패턴에 따라 자동 계층 이동, 비용 최적화
  - Standard-IA (Infrequent Access): 덜 자주 액세스하는 데이터, 더 낮은 스토리지 비용
  - One Zone-IA: 단일 가용 영역에 저장, 비용 절감형
  - Glacier: 장기 보관용, 검색에 시간 소요, 매우 저렴한 비용
- S3 주요 기능
  - 1. 정적 웹사이트 호스팅
    - HTML, CSS, JS 파일 호스팅
    - 사용자 지정 도메인 사용 가능
    - HTTPS 지원 (CloudFront 연동)
  - 2. 설정
    - 인덱스 문서 지정
    - 오류 문서 지정
    - 리다이렉션 규칙

## CloudFront와 CDN

- 전 세계에 분산된 서버 네트워크를 통해 컨텐츠를 빠르게 전송하는 시스템
- 작동 방식: 사용자 컨텐츠 요청 -> 가장 가까운 엣지 로케이션으로 라우팅 -> 캐시된 컨텐츠가 있으면 즉시 응답 -> 없으면 오리진 서버에서 가져와 캐시 후 응답
- 주요기능
  - 1. 캐싱 최적화: TTL설정, 캐시 동작 구성, 압축 지원
  - 2. 보안: SSL/TLS 지원, WAF 통합, 지리적 제한, 서명된 URL/쿠키
  - 3. 성능 최적화: HTTP/3 지원, Origin Shield, 실시간 로그

## 캐시 무효화(Cache Invalidation)

- 엣지 로케이션에 저장된 캐시를 강제로 제거하는 작업
- 새로운 컨텐츠를 즉시 반영하기 위해 사용
- TTL 만료 전에 컨텐츠 업데이트 필요 시 활용됨

## Repository secret과 환경변수

- 리포지토리에 안전하게 저장되는 암호화된 환경변수
- API 키, 비밀번호 등 민감한 정보 저장
- Github Actions에서 안전하게 사용 가능함
- 경로: 리포지토리 → Settings → Secrets and variables → Actions → New repository secret

```
# Github CLI 사용시

# Secret 생성
gh secret set AWS_ACCESS_KEY_ID
gh secret set AWS_SECRET_ACCESS_KEY

# Secret 목록 확인
gh secret list

# Secret 삭제
gh secret remove AWS_ACCESS_KEY_ID
```

## S3 / CloudFront 성능 비교 분석

### 아키텍쳐 비교

#### S3 직접 접근 방식

request → [단일 리전의 S3] → response

- 단일 리전에서만 서비스
- 지역적 제약으로 인한 지연 발생
- 예) 도쿄 -> 서울 리전 S3접근 시 상당한 지연 발생

#### CloudFront 접근 방식

request → [가장 가까운 엣지 로케이션] → [캐시 히트 시 즉시 응답] → [캐시 미스 시 S3 접근] → response

- 전 세계 엣지 로케이션 활용
- 캐시를 통한 빠른 응답
- 예) 도쿄 사용자는 도쿄 엣지 로케이션에서 빠른 응답 수신

### 성능 메트릭스 비교

#### 지연 시간

- S3 직접 접근 방식: 50-200ms
- CloudFront 직접 접근 방식: 10-50ms

#### 전송 속도

- S3: 단일 리전 기준 ~50mbps
- CloudFront: 최대 100mbps 이상

#### CloudFront 성능 최적화 기능

- 엣지 캐싱
- 콘텐츠 압축(Gzip / Brotli)
- TCP 연결 최적화
- SSL / TLS 종단점 최적화
- HTTP/2 및 HTTP/3 지원

### 비용구조

#### S3직접 접근

- GET 요청당 비용
- 데이터 전송 비용

#### CloudFront

- 엣지 로케이션 요청 비용
- 오리진 요청 비용
- 데이터 전송 비용

### S3 실제 성능 지표

![S3성능지표](https://github.com/user-attachments/assets/32ec7814-0e4c-4747-9c86-cbe67cd1e45c)

### CloudFront 실제 성능 지표

![CloudFront성능지표](https://github.com/user-attachments/assets/e9087753-cb2d-458a-bd2d-c523dc265dab)

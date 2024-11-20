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
 - Github Actions와 CI/CD 도구:
 - S3와 스토리지:
 - CloudFront와 CDN:
 - 캐시 무효화(Cache Invalidation):
 - Repository secret과 환경변수:

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

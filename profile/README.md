# SurvivAlgo

백준(Solvedac) 사용자들을 위한 알고리즘 문제 풀이 추적 및 경쟁 서비스

## 프로젝트 정보

- **팀 구성**: 2인 팀
  - Backend: [@gonudayo](https://github.com/gonudayo)
  - Frontend: [@develeep](https://github.com/develeep)
- **개발 기간**: 약 1개월
- **기술 스택**: Express.js, MongoDB, React(TypeScript)
- **배포**: Netlify (클라이언트)

## 프로젝트 개요

SurvivAlgo는 백준 사용자들이 그룹을 만들어 함께 문제를 풀고 경쟁할 수 있는 플랫폼입니다. 사용자들은 그룹에 참여하여 일일 문제 풀이 목표를 달성하고, 그룹 및 개인 랭킹을 확인할 수 있습니다.

### 주요 기능

- **그룹 관리**: 그룹 생성, 참여, 일일 문제 풀이 현황 추적
- **랭킹 시스템**: 그룹 랭킹(총 문제 수, 티어 합, 연속 달성일), 개인 랭킹
- **자동 업데이트**: 백준 계정 정보 실시간 동기화
- **사용자 인증**: Solved.ac 프로필 스크래핑으로 인증 코드 확인

## 아키텍처

### 시스템 구성

```
┌─────────────────┐
│   Client        │  React + TypeScript + Vite
│  (Netlify)      │  - 사용자 인터페이스
└────────┬────────┘  - TanStack Query, Zustand
         │
         │ HTTPS/REST API
         │
┌────────▼─────────┐
│  Main Server     │  Node.js + Express
│   (Port 8000)    │  - 모든 API 엔드포인트 처리
└────────┬─────────┘  - 짝수 인덱스 유저 업데이트
         │
         │ MongoDB
         │
┌────────▼──────────────────────────┐
│         MongoDB                   │
│  - 사용자, 그룹, 문제 풀이 데이터     │
└────────▲──────────────────────────┘
         │
         │ MongoDB
         │
┌────────┴─────────┐
│  Worker Server   │  Node.js + Express
│                  │  - 홀수 인덱스 유저 업데이트 전담
└──────────────────┘  - 부하 분산
```

### 분산 처리 전략

유저 정보 업데이트는 solved.ac를 스크래핑하여 이루어지는데, 사용자 수가 증가할수록 다음과 같은 문제가 발생합니다:

1. **업데이트 부하 증가**: 모든 유저를 순차적으로 업데이트하는 데 시간이 오래 걸림
2. **외부 API 제한**: solved.ac API 호출 제한으로 인한 차단 위험

이를 해결하기 위해 **인덱스 기반 부하 분산** 방식을 도입했습니다:

#### 메인 서버 (Main Server)
- 모든 API 요청 처리
- **짝수 인덱스(0, 2, 4, ...)** 유저 업데이트
- 랜덤 딜레이

#### 워커 서버 (Worker Server)
- API 요청 처리 없음
- **홀수 인덱스(1, 3, 5, ...)** 유저 업데이트
- 랜덤 딜레이

```javascript
// 메인 서버
currentIndex = 0
while (true) {
  updateUser(users[currentIndex])  // 0, 1, 2, 3, 4, 5, ...
  currentIndex += 1
  delay(3000 + random(3000))
}

// 워커 서버
currentIndex = 1
while (true) {
  updateUser(users[currentIndex])  // 1, 3, 5, 7, 9, ...
  currentIndex += 2
  delay(3000 + random(3000))
}
```

#### 확장성

- 유저 수가 더 증가하면 워커 서버를 추가로 배포 가능
- 모듈로 연산을 활용하여 더 세밀한 분할 가능
  - 예: 워커1 (i % 3 == 1), 워커2 (i % 3 == 2)

## 레포지토리 구조

### 1. algorithm-survival-client
프론트엔드 애플리케이션

**기술 스택**
- React 19, TypeScript, Vite
- TanStack Query (서버 상태 관리)
- Zustand (클라이언트 상태 관리)
- Tailwind CSS + DaisyUI

**주요 기능**
- 사용자 인증 (회원가입, 로그인)
- 그룹 관리 UI
- 랭킹 조회 및 시각화
- 반응형 디자인

### 2. algorithm-survival-server-main
메인 API 서버

**기술 스택**
- Node.js + Express
- MongoDB + Mongoose
- JWT, Bcrypt
- Puppeteer + Cheerio (웹 스크래핑)

**주요 기능**
- RESTful API 제공
  - `/api/v2/auth` - 인증
  - `/api/v2/users` - 사용자 관리
  - `/api/v2/groups` - 그룹 관리
  - `/api/v2/rankings` - 랭킹 조회
  - `/api/v2/search` - 검색
- 짝수 인덱스 유저 자동 업데이트
- 매일 오전 6시 그룹 초기화 스케줄러

### 3. algorithm-survival-server-worker
워커 서버 (부하 분산)

**기술 스택**
- Node.js + Express
- MongoDB + Mongoose (메인 서버와 DB 공유)
- Puppeteer + Cheerio

**주요 기능**
- 홀수 인덱스 유저 자동 업데이트 전담
- 메인 서버와 부하 분산
- 동일한 그룹 초기화 스케줄러 (이중화)

## 시작하기

### 전체 시스템 실행 순서

1. **MongoDB 실행**
   ```bash
   # MongoDB 서버 시작 (별도 터미널)
   mongod --dbpath /path/to/data
   ```

2. **메인 서버 실행**
   ```bash
   cd algorithm-survival-server-main
   npm install

   # .env 파일 설정
   echo "PORT=8000" > .env
   echo "MONGO_URI=mongodb://localhost:27017/survivalgo" >> .env
   echo "JWT_SECRET=your-secret-key" >> .env
   echo "JWT_EXPIRE=7d" >> .env

   npm run dev
   ```

3. **워커 서버 실행** (선택사항, 부하 분산 필요시)
   ```bash
   cd algorithm-survival-server-worker
   npm install

   # .env 파일 설정
   echo "MONGO_URI=mongodb://localhost:27017/survivalgo" > .env

   npm run dev
   ```

4. **클라이언트 실행**
   ```bash
   cd algorithm-survival-client
   npm install

   # .env 파일 설정
   echo "VITE_API_URL=http://localhost:8000" > .env

   npm run dev
   ```

### 배포 환경

- **클라이언트**: Netlify
- **메인 서버**: Google Cloud VM (Compute Engine)
- **워커 서버**: Google Cloud VM (Compute Engine)
- **데이터베이스**: MongoDB (Google Cloud VM 내 자체 호스팅)

## 기술적 의사결정

### 1. 분산 처리 아키텍처 선택
- **문제**: 유저 수 증가 시 단일 서버로는 모든 유저 정보를 실시간으로 업데이트하기 어려움
- **해결**: 인덱스 기반 부하 분산으로 수평 확장 가능한 구조 설계
- **장점**: 워커 서버 추가만으로 처리량 증가 가능

### 2. 웹 스크래핑 vs API 사용
- **백준 공식 API 부재**: 웹 스크래핑(Puppeteer + Cheerio) 사용
- **solved.ac API**: 보조적으로 활용
- **외부 API 제한 대응**: 랜덤 딜레이 + 분산 처리

### 3. 실시간 업데이트 전략
- WebSocket이 아닌 주기적 폴링 선택
- 이유: 백준 데이터 자체가 실시간이 아니며, 스크래핑 부하 고려
- 사용자는 TanStack Query의 캐싱으로 원활한 UX 제공

## 향후 개선 계획

- [ ] 워커 서버 자동 스케일링
- [ ] Redis를 활용한 캐싱 레이어 추가
- [ ] WebSocket을 통한 실시간 알림
- [ ] 그룹 챌린지 기능 확장

## 주요 구현 사항

### Backend
- **분산 처리 아키텍처**: 인덱스 기반 부하 분산으로 유저 업데이트 처리량 2배 향상
- **RESTful API**: Express.js 기반 5개 주요 엔드포인트 설계 및 구현
- **웹 스크래핑**: Puppeteer + Cheerio를 활용한 백준 사용자 정보 수집
- **자동화**: Node-cron을 활용한 일일 그룹 초기화 스케줄러
- **인증 시스템**: JWT 기반 안전한 인증/인가
- **데이터베이스**: MongoDB + Mongoose 스키마 설계 및 최적화

### Frontend
- **UI/UX**: React + TypeScript 기반 반응형 웹 애플리케이션
- **상태 관리**: TanStack Query (서버 상태), Zustand (클라이언트 상태)
- **폼 관리**: React Hook Form + Zod를 활용한 검증
- **스타일링**: Tailwind CSS + DaisyUI를 통한 일관된 디자인


## 레포지토리

- **Frontend**: [algorithm-survival-client](https://github.com/7seoul/algorithm-survival-client)
- **Main Server**: [algorithm-survival-server-main](https://github.com/7seoul/algorithm-survival-server-main)
- **Worker Server**: [algorithm-survival-server-worker](https://github.com/7seoul/algorithm-survival-server-worker)

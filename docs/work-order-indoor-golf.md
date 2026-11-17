# 작업 지시서: 랜선 인도어 (온라인 실내 골프 연습장) MVP

> 이 문서는 Claude Code에게 주는 작업 지시서다. 위에서부터 순서대로 읽고, **7. 작업 단계**를 한 단계씩 진행한다.
> 기획 원문: 「온라인 인도어 골프 연습장 기획서 (초안)」 (2026-11-17)

---

## 0. 작업 방식 (반드시 지킬 것)

- 7장의 단계를 **순서대로** 진행한다. 단계가 끝날 때마다 완료 기준을 직접 확인하고 커밋한다.
- 커밋 메시지는 `[단계 N] 요약` 형식으로 쓴다.
- 이 문서에 정해진 수치와 규칙은 **임의로 바꾸지 않는다**. 바꿔야 할 이유가 생기면 코드에 반영하기 전에 사용자에게 묻는다.
- 정해지지 않은 사항은 9장의 기본값을 쓰고, 코드에 `// TODO(기획확인):` 주석을 남긴다.
- 게임 규칙에 쓰는 수치(거리, 시간, 상한)는 모두 `packages/shared/src/config.ts` 한 곳에 상수로 모은다.
- 비거리 계산과 기록 저장은 **서버만** 한다. 클라이언트가 보낸 거리 값은 믿지 않는다.
- 실제 회사 로고나 상표는 저장소에 넣지 않는다. 예시와 시드 데이터는 가상 회사 '골빅'(golbig.co.kr)을 쓴다.

---

## 1. 목표

가입 없이 접속하면 이름이 자동으로 붙는 웹 서비스를 만든다. 사용자는 1·2·3층 중 빈 타석을 골라 정해진 시간 동안 공을 친다. 친 공 수와 최장 비거리는 전광판 랭킹에 실시간으로 반영된다. 원하는 사람은 회사 이메일로 인증하고, 인증하면 이름 앞에 회사 배지가 붙는다.

**MVP 완료 = 아래 시나리오가 모바일 브라우저에서 끝까지 동작하는 것**

1. 처음 접속 → 자동 이름 발급(새로고침해도 유지)
2. 층 선택 → 빈 타석 선택 → 입장 (만석이면 대기표)
3. 드라이버 또는 7번 아이언으로 스윙 → 거리 연출
4. 10분이 지나거나 공 100개를 치면 자동 퇴장 → 결과 카드
5. 전광판의 연습량 순위와 비거리 순위에 내 기록이 반영됨
6. 회사 이메일 인증 → 이름 앞에 회사 배지 표시, 기존 기록 유지

---

## 2. 기술 스택

| 영역 | 선택 | 비고 |
| --- | --- | --- |
| 저장소 구조 | pnpm 워크스페이스 모노레포 | `apps/web`, `apps/server`, `packages/shared` |
| 언어 | TypeScript (strict) | 공통 타입과 규칙은 `packages/shared` |
| 웹 | Next.js (App Router) + Tailwind CSS | 모바일 우선 |
| 타석 화면 | HTML Canvas 2D | 외부 게임 엔진 쓰지 않음 |
| 서버 | Node.js + Fastify + Socket.IO | REST + 실시간 이벤트 |
| DB | PostgreSQL + Prisma | 원본 기록 |
| 랭킹·실시간 상태 | Redis | 정렬 집합(ZSET), 타석 점유, 대기열 |
| 메일 | `MailSender` 인터페이스 | 개발: 콘솔 출력, 운영: 환경변수로 공급자 선택 |
| 로컬 실행 | docker-compose (postgres, redis) | `pnpm dev` 하나로 web+server 실행 |
| 테스트 | Vitest(단위·통합), Playwright(E2E) | |

환경변수는 `.env.example`에 모두 적는다. 최소 `DATABASE_URL`, `REDIS_URL`, `SESSION_SECRET`, `EMAIL_ENC_KEY`, `MAIL_PROVIDER`, `APP_TZ=Asia/Seoul`이 필요하다.

---

## 3. 도메인 규칙

### 3.1 익명 사용자와 자동 이름

- 첫 접속 때 서버가 `userId`(UUID)와 자동 이름을 발급하고, 서명된 httpOnly 쿠키(만료 1년)에 저장한다.
- 자동 이름 = 수식어 + 골프 명사. 예: `느린 백스윙`, `뒤땅 마스터`, `새벽 스윙`, `장타 본능`.
  - 수식어와 명사는 각각 40개 이상을 `packages/shared/src/names.ts`에 둔다.
  - 이름은 겹쳐도 된다. 사용자 구분은 `userId`로 한다.
- 별명 변경은 MVP에서 하지 않는다.

### 3.2 층과 타석

| 층 | 이름 | 타석 수 | 쓸 수 있는 클럽(MVP) |
| --- | --- | --- | --- |
| 3층 | 장타 라운지 | 20 | 드라이버 |
| 2층 | 일반 타석 | 40 | 드라이버, 7번 아이언 |
| 1층 | 입문 타석 | 40 | 7번 아이언 |

- 타석 번호는 층마다 1번부터 매긴다. 표기 예: `3F-17`.
- 한 사용자는 동시에 타석 하나만 쓸 수 있다.
- **세션 종료 조건:** 입장 후 10분 또는 공 100개 중 먼저 오는 쪽.
- 종료 1분 전에 클라이언트에 알림 이벤트를 보낸다.
- 연결이 끊기면 60초 동안 자리를 유지한다. 그 안에 돌아오지 않으면 세션을 끝낸다.

### 3.3 대기표

- 층이 만석이면 그 층의 대기열(FIFO)에 들어가 번호표를 받는다.
- 자리가 나면 대기 1번에게 알리고 30초 안에 입장하게 한다. 입장하지 않으면 다음 사람에게 넘어간다.
- 화면에 층별 `사용 중 / 대기` 인원을 실시간으로 보여준다.

### 3.4 스윙과 비거리

조작은 탭 두 번이다.

1. 첫 번째 탭(누르고 있다 떼기): 백스윙 게이지. 뗀 시점이 **파워**(0~1)를 정한다.
2. 두 번째 탭: 되돌아오는 바가 스윗 스팟에서 얼마나 벗어났는지가 **임팩트 오차**(-1~1, 0이 정중앙)를 정한다.

클라이언트는 게이지 시작 시각 기준의 두 탭 시각(ms)만 보낸다. 서버는 클럽별 게이지 속도로 파워와 오차를 다시 계산한다.

| 클럽 | 기본 거리 | 게이지 왕복 시간 | 스윗 스팟 허용 폭 |
| --- | --- | --- | --- |
| 드라이버 | 230m | 1.2초 | 0.10 |
| 7번 아이언 | 140m | 1.6초 | 0.18 |

```
정확도   = max(0, 1 - |오차| / 허용폭)                     (0~1)
파워계수 = 0.6 + 0.4 × 파워
바람     = 균등분포 -3m ~ +3m (서버 난수)
비거리   = round(기본거리 × 파워계수 × (0.7 + 0.3 × 정확도) + 바람, 소수 1자리)
```

- `|오차| > 허용폭 × 2`이면 **미스샷**이다. 오차 부호와 크기에 따라 `뒤땅`, `탑볼`, `생크` 중 하나를 정하고, 비거리는 위 값의 10~40%로 한다.
- 정확도가 0.9 이상이면 `나이스샷`으로 표시한다.
- 미스샷도 공 1개로 센다.
- **어뷰징 방지:**
  - 같은 사용자의 스윙 사이 간격은 최소 2.5초. 더 짧은 요청은 거절한다.
  - 하루 공 수 상한은 2,000개. 넘긴 기록은 저장하되 랭킹에는 반영하지 않는다.
  - 두 탭 시각이 사람이 낼 수 없는 값(음수, 게이지 시간 초과)이면 거절한다.

### 3.5 랭킹

| 항목 | 연습량 순위(왼쪽) | 비거리 순위(오른쪽) |
| --- | --- | --- |
| 값 | 기간 안에 친 공 수 합계 | 기간 안의 단일 샷 최장 거리 |
| 정렬 | 내림차순, 동점이면 그 값에 먼저 도달한 사람 | 내림차순, 동점이면 먼저 친 사람 |
| 기간 | 오늘 / 이번 주 / 전체 | 오늘 / 이번 주 / 전체 |
| 표시 | TOP 20 + 내 순위 | TOP 20 + 내 순위, 클럽 표시 |

- 기간 기준 시간대는 Asia/Seoul이다. 하루는 00:00에 시작하고, 주는 월요일에 시작한다.
- Redis 키 예시: `rank:volume:day:2026-11-17`, `rank:distance:week:2026-W47`, `rank:volume:all`.
- 일·주 키에는 TTL을 둔다(일 8일, 주 5주).
- **회사 순위(가운데):** 인증된 회사별 공 수 합계. 인증 인원 3명 이상인 회사만 올린다. TOP 10.
- 순위가 바뀌면 Socket.IO로 전광판에 내보낸다. 이벤트는 초당 최대 2회로 묶어 보낸다.

### 3.6 회사 인증

1. 사용자가 회사 이메일을 입력한다.
2. 개인 메일 도메인은 거절한다. 최소 `gmail.com`, `naver.com`, `daum.net`, `hanmail.net`, `kakao.com`, `nate.com`, `hotmail.com`, `outlook.com`, `yahoo.com`, `icloud.com`이며, 목록은 설정으로 뺀다.
3. 6자리 숫자 코드를 메일로 보낸다. 유효 시간은 10분, 같은 주소로 하루 5회까지 보낸다. 코드 입력 실패 5회면 그 코드는 폐기한다.
4. 인증되면 현재 익명 `userId`를 그대로 인증 계정으로 승격한다. 기록을 옮길 필요가 없다.
5. 배지 규칙:
   - `companies` 테이블에 도메인이 있고 `logoApproved = true`이면 로고 배지를 단다.
   - 그 외에는 도메인 텍스트 배지를 단다(예: `golbig.co.kr`).
6. 인증은 180일마다 갱신한다. 만료되면 배지를 내린다.
7. **다른 기기에서 이어하기:** 같은 이메일로 코드 인증을 하면 그 계정의 쿠키를 발급한다. 비밀번호는 두지 않는다.
8. **개인정보:**
   - 이메일은 AES-256-GCM으로 암호화해 저장한다(`EMAIL_ENC_KEY`).
   - 조회용으로 HMAC-SHA256 해시를 따로 저장한다.
   - 화면과 API 응답에 이메일을 절대 노출하지 않는다.
   - 인증 전에 수집 목적을 알리는 동의 체크박스를 둔다.
   - 인증 해제(탈퇴) 요청이 오면 이메일 데이터를 즉시 삭제한다.

---

## 4. 데이터 모델 (Prisma 기준 초안)

```prisma
model User {
  id            String   @id @default(uuid())
  displayName   String
  createdAt     DateTime @default(now())
  emailEnc      Bytes?            // 암호화된 회사 이메일
  emailHash     String?  @unique  // 조회용 HMAC
  companyDomain String?
  verifiedAt    DateTime?
  verifyExpires DateTime?
  sessions      BaySession[]
  shots         Shot[]
}

model Company {
  domain       String  @id        // 예: golbig.co.kr
  name         String
  logoUrl      String?
  logoApproved Boolean @default(false)
}

model BaySession {
  id        String    @id @default(uuid())
  userId    String
  floor     Int                   // 1, 2, 3
  bay       Int
  startedAt DateTime  @default(now())
  endedAt   DateTime?
  endReason String?               // time | balls | disconnect | leave
  ballCount Int       @default(0)
  user      User      @relation(fields: [userId], references: [id])
  shots     Shot[]
}

model Shot {
  id         String     @id @default(uuid())
  sessionId  String
  userId     String
  club       String               // driver | iron7
  power      Float
  offset     Float
  distance   Float
  result     String               // nice | normal | duff | top | shank
  ranked     Boolean    @default(true)
  createdAt  DateTime   @default(now())
  session    BaySession @relation(fields: [sessionId], references: [id])
  user       User       @relation(fields: [userId], references: [id])
  @@index([userId, createdAt])
}

model EmailCode {
  id        String   @id @default(uuid())
  emailHash String
  codeHash  String
  expiresAt DateTime
  attempts  Int      @default(0)
  createdAt DateTime @default(now())
  @@index([emailHash, createdAt])
}
```

타석 점유와 대기열은 Redis에만 둔다. 서버를 재시작하면 진행 중이던 세션은 `endReason = disconnect`로 정리한다.

---

## 5. API와 실시간 이벤트

### REST

| 메서드 | 경로 | 설명 |
| --- | --- | --- |
| POST | `/api/me` | 쿠키가 없으면 익명 사용자 생성, 있으면 내 정보 반환 |
| GET | `/api/floors` | 층별 타석 상태, 사용·대기 인원 |
| GET | `/api/rankings?type=volume\|distance\|company&period=day\|week\|all` | TOP N + 내 순위 |
| POST | `/api/verify/request` | `{ email, consent }` → 코드 발송 |
| POST | `/api/verify/confirm` | `{ email, code }` → 인증 완료 |
| DELETE | `/api/verify` | 인증 해제 및 이메일 삭제 |

### Socket.IO

| 방향 | 이벤트 | 내용 |
| --- | --- | --- |
| C→S | `bay:enter` | `{ floor, bay }` |
| C→S | `queue:join` / `queue:leave` | `{ floor }` |
| C→S | `swing` | `{ club, t1, t2 }` (게이지 시작 기준 ms) |
| C→S | `bay:leave` | 자진 퇴장 |
| S→C | `floor:update` | 타석 점유 변화, 사용·대기 인원 |
| S→C | `queue:position` / `queue:call` | 대기 순번, 입장 호출(30초) |
| S→C | `swing:result` | `{ distance, result, ballCount, remainingSec }` |
| S→C | `session:warning` | 종료 1분 전 |
| S→C | `session:end` | `{ reason, summary }` (결과 카드용) |
| S→C | `ranking:update` | 바뀐 순위판 |

---

## 6. 화면

모든 화면은 360px 너비에서 깨지지 않아야 한다.

1. **입구(`/`):** 자동 이름이 적힌 타석 키 카드, 층별 사용·대기 인원 전광판, 1·2·3 엘리베이터 버튼.
2. **층 화면(`/floor/[n]`):** 타석 격자. 빈 타석은 밝게, 사용 중인 타석은 이름과 배지를 표시한다. 만석이면 '대기표 받기' 버튼.
3. **타석 화면(`/bay`):**
   - Canvas에 공 궤적과 거리 표지판(50m 간격)을 그린다.
   - 하단에 게이지와 클럽 선택을 둔다.
   - 남은 시간과 남은 공을 표시한다.
   - 결과 텍스트(나이스샷, 뒤땅 등)를 보여준다.
4. **전광판(`/board`, 입구와 타석 화면에도 축소판):**
   - 왼쪽 연습량 순위, 가운데 회사 순위, 오른쪽 비거리 순위.
   - 상단에 오늘/이번 주/전체 탭, 하단에 내 순위를 고정한다.
5. **결과 카드:**
   - 퇴장할 때 `오늘 공 132개 · 최장 241m · 3F-17` 형식의 1080×1080 PNG를 클라이언트에서 만든다.
   - 저장 버튼과 공유 버튼(Web Share API)을 둔다.
6. **회사 인증 모달:** 이메일 입력 → 동의 → 코드 입력 → 완료. 오류 문구는 한국어로 구체적으로 쓴다.

사운드(타구음)는 MVP에 넣되, 기본 음소거와 토글을 둔다.

---

## 7. 작업 단계와 완료 기준

| 단계 | 할 일 | 완료 기준 |
| --- | --- | --- |
| 1 | 모노레포, docker-compose, Prisma 스키마, `.env.example`, `pnpm dev`/`pnpm test` 스크립트 | 새로 클론해서 README 순서대로 하면 web과 server가 뜬다 |
| 2 | `packages/shared`: 설정 상수, 이름 생성기, 비거리 계산 함수 | 비거리 계산 단위 테스트: 정중앙·최대 파워, 경계값, 미스샷 3종, 바람 범위 |
| 3 | 익명 사용자(`/api/me`), 쿠키 서명 | 새로고침 후 같은 이름이 나온다. 쿠키 위조는 거절한다 |
| 4 | 층·타석 점유, 세션 타이머, 끊김 60초 유예 | 통합 테스트: 동시 입장 경쟁에서 한 명만 성공. 10분·100개 종료 |
| 5 | 대기열과 30초 호출 | 통합 테스트: FIFO 순서, 호출 시간 초과 시 다음 사람으로 넘어감 |
| 6 | 스윙 처리, 어뷰징 검사, Shot 저장 | 2.5초 미만 요청과 비정상 시각 거절. 하루 상한 초과분은 `ranked=false` |
| 7 | 랭킹(ZSET, 기간 키, 동점 규칙, 회사 순위), 실시간 송신 | 단위 테스트: 동점 처리, 날짜·주 경계(Asia/Seoul), 회사 3명 조건 |
| 8 | 프론트: 입구, 층, 타석(Canvas), 전광판 | 1장의 시나리오 1~5가 수동으로 동작 |
| 9 | 회사 인증(요청·확인·해제), 배지 표시, 시드 데이터(골빅) | 개인 도메인 거절, 코드 만료·횟수 제한, 이메일 미노출 확인 |
| 10 | 결과 카드, 사운드 토글 | PNG가 1080×1080으로 저장된다 |
| 11 | Playwright E2E | 1장 시나리오 1~6을 자동으로 검증. 모바일 뷰포트(390×844) 포함 |
| 12 | README 정리 | 실행 방법, 환경변수, 아키텍처 그림(텍스트), 남은 TODO 목록 |

각 단계가 끝나면 `pnpm test`와 `pnpm lint`를 통과시킨 뒤 커밋한다.

---

## 8. 이번 범위에서 하지 않는 것

- 우드·웨지·퍼터, 클럽별 미니 미션, 콤보, 홀인원 연출, 클럽 스킨
- 점심 챌린지, 금요일 장타 대회, 월간 회사 대항전 같은 이벤트
- 결제와 후원, 층 네이밍 스폰서
- 회사 로고 업로드와 승인용 관리자 화면(MVP는 시드 데이터로 대신함)
- 별명 변경, 채팅, 앱스토어 앱

---

## 9. 미확정 사항과 임시 기본값

| 항목 | 임시 기본값 | 확인할 사람 |
| --- | --- | --- |
| 서비스 이름 | 랜선 인도어 | 기획자 |
| 예시 회사 이름 | 골빅(golbig.co.kr, 가상) | 기획자 |
| 연습 단위 | 10분 또는 공 100개 | 기획자 |
| 하루 입장 횟수 제한 | 없음 | 기획자 |
| 운영 메일 공급자 | 미정 (인터페이스만 구현) | 기획자 |
| 배포 환경 | 미정 (Docker 이미지까지만 준비) | 기획자 |

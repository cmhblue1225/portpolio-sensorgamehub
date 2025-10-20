# 🚂 Railway 배포 가이드

## 📋 배포 전 체크리스트

### ✅ 1. Railway 계정 준비
- Railway 계정 생성: https://railway.app
- GitHub 연동 완료
- 프로젝트 생성

### ✅ 2. 환경 변수 준비 (필수!)

Railway 대시보드에서 다음 환경 변수를 설정해야 합니다:

```bash
# Supabase (이미 공개된 값)
SUPABASE_URL=https://rwkgktwdljsddowcxphc.supabase.co
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InJ3a2drdHdkbGpzZGRvd2N4cGhjIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTY4NTk3ODMsImV4cCI6MjA3MjQzNTc4M30.6L8MUwLS7sLjKXSST8fpqp8Qi0F0TMz-z9PyXiQK2Yg

# AI API Keys (실제 값 입력 필요!)
CLAUDE_API_KEY=sk-ant-api03-xxxxxxxxxxxxxxxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 서버 설정
NODE_ENV=production
ALLOWED_ORIGINS=https://your-railway-app.up.railway.app

# PORT는 Railway가 자동 할당 (설정 불필요)
```

---

## 🚀 배포 단계

### Step 1: Railway 프로젝트 생성

1. Railway 대시보드 접속: https://railway.app/dashboard
2. "New Project" 클릭
3. "Deploy from GitHub repo" 선택
4. 이 저장소 선택

### Step 2: 환경 변수 설정

1. 프로젝트 대시보드에서 "Variables" 탭 클릭
2. 위의 환경 변수 추가
3. **중요**: `CLAUDE_API_KEY`와 `OPENAI_API_KEY`는 실제 API 키로 교체!

### Step 3: 배포 설정 확인

Railway는 자동으로 감지하지만, 수동 설정도 가능:

1. "Settings" 탭
2. "Deploy" 섹션
3. **Start Command**: `npm start` (자동 감지됨)
4. **Build Command**: `npm install` (자동 감지됨)

### Step 4: 배포 시작

1. "Deploy" 버튼 클릭
2. 빌드 로그 확인
3. 배포 완료 후 URL 확인 (예: `https://sensorchatbot-production.up.railway.app`)

---

## ⚠️ 알려진 제약사항 및 해결책

### 1. 파일 시스템 쓰기 제한 (Ephemeral Storage)

**문제**: AI 생성 게임이 `public/games/` 폴더에 저장되는데, Railway 재배포 시 삭제됨.

**해결책 A (권장): Supabase Storage 활용**

모든 AI 생성 게임을 Supabase Storage에 저장하고, 필요 시 다운로드:

```javascript
// server/InteractiveGameGenerator.js 수정 예시
async saveGameToStorage(gameId, gameCode) {
  const { data, error } = await this.supabaseClient
    .storage
    .from('generated-games')
    .upload(`${gameId}/index.html`, gameCode, {
      contentType: 'text/html',
      upsert: true
    });

  if (error) throw error;
  return data;
}
```

**해결책 B (임시): 메모리 캐싱**

재배포 전까지만 유효한 임시 저장:
- 현재 구조 유지
- 사용자에게 "게임 다운로드" 기능 제공
- 재배포 시 생성된 게임 삭제됨 (사용자에게 안내)

### 2. WebSocket 연결

**Railway는 WebSocket을 완벽 지원**하므로 Socket.IO가 정상 작동합니다.

단, 다음 설정 확인:

```javascript
// server/index.js (이미 설정됨)
this.io = socketIo(this.server, {
  cors: {
    origin: "*",  // 프로덕션에서는 특정 도메인으로 제한 권장
    methods: ["GET", "POST"]
  },
  transports: ['websocket', 'polling']  // ✅ 폴백 포함
});
```

### 3. 메모리 사용량

**Railway 무료 플랜**: 512MB RAM
**이 프로젝트 예상 사용량**: ~200-300MB (AI 응답 처리 시 증가)

**모니터링 방법**:
1. Railway 대시보드 → "Metrics" 탭
2. 메모리 사용량 체크
3. 필요 시 Starter 플랜 업그레이드 ($5/월)

---

## 🔧 배포 후 테스트

### 1. 기본 기능 테스트

```bash
# 게임 목록 확인
curl https://your-app.up.railway.app/api/games

# 서버 통계 확인
curl https://your-app.up.railway.app/api/stats
```

### 2. WebSocket 연결 테스트

1. 배포된 URL 접속: `https://your-app.up.railway.app`
2. 게임 선택 (예: Cake Delivery)
3. 세션 생성 확인
4. 모바일에서 QR 코드 스캔
5. 센서 데이터 전송 확인

### 3. AI 게임 생성 테스트

1. `/developer` 페이지 접속
2. "AI 게임 생성기" 클릭
3. 게임 아이디어 입력
4. 생성 진행률 확인
5. 생성된 게임 플레이

---

## 📊 비용 예측

### Railway 요금제

| 플랜 | 비용 | RAM | CPU | 적합성 |
|------|------|-----|-----|--------|
| **Trial** | 무료 | 512MB | Shared | ✅ 테스트용 |
| **Starter** | $5/월 | 8GB | 8vCPU | ✅ 개인 프로젝트 |
| **Pro** | $20/월 | 32GB | 32vCPU | 상용 서비스 |

**권장**: Starter 플랜 ($5/월)
- AI 응답 처리에 충분한 메모리
- 동시 접속 50명 이상 지원
- 무제한 배포

### 외부 API 비용

| 서비스 | 비용 | 사용량 |
|--------|------|--------|
| **Claude API** | ~$0.003/1K 토큰 | 게임 생성 시 (64K 토큰 = ~$0.20/게임) |
| **OpenAI Embeddings** | ~$0.0001/1K 토큰 | RAG 검색 시 (거의 무료) |
| **Supabase** | 무료 (500MB DB) | 게임 메타데이터 저장 |

**예상 월 비용**:
- Railway: $5
- AI API: ~$10 (월 50개 게임 생성 시)
- **총 $15/월**

---

## 🔒 보안 체크리스트

### 1. 환경 변수 보호

- ✅ `.env` 파일은 `.gitignore`에 포함
- ✅ Railway 환경 변수로 API 키 관리
- ⚠️ CORS 설정을 특정 도메인으로 제한 (프로덕션)

```javascript
// server/index.js 수정 권장
this.io = socketIo(this.server, {
  cors: {
    origin: [
      "https://your-railway-app.up.railway.app",
      "http://localhost:3000"  // 개발용
    ],
    methods: ["GET", "POST"]
  }
});
```

### 2. Rate Limiting (권장)

AI API 남용 방지:

```javascript
// 추가 패키지 설치
npm install express-rate-limit

// server/index.js에 추가
const rateLimit = require('express-rate-limit');

const aiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15분
  max: 5, // 최대 5번 AI 게임 생성
  message: '너무 많은 요청입니다. 15분 후 다시 시도하세요.'
});

this.app.post('/api/finalize-game', aiLimiter, ...);
```

---

## 🐛 트러블슈팅

### 1. 배포 실패 (Build Error)

**증상**: `npm install` 실패

**해결**:
```bash
# 로컬에서 package-lock.json 재생성
rm -rf node_modules package-lock.json
npm install
git add package-lock.json
git commit -m "Fix package-lock.json"
git push
```

### 2. 환경 변수 인식 안 됨

**증상**: "CLAUDE_API_KEY is not defined" 오류

**해결**:
1. Railway 대시보드 → Variables 탭
2. 환경 변수 재확인
3. "Redeploy" 버튼 클릭

### 3. WebSocket 연결 실패

**증상**: Socket.IO 연결 시 CORS 오류

**해결**:
```javascript
// server/index.js
this.io = socketIo(this.server, {
  cors: {
    origin: "*",  // 또는 Railway 도메인
    credentials: true
  }
});
```

---

## 📚 추가 자료

- **Railway 공식 문서**: https://docs.railway.app
- **Node.js 배포 가이드**: https://docs.railway.app/guides/nodejs
- **환경 변수 관리**: https://docs.railway.app/develop/variables
- **WebSocket 지원**: https://docs.railway.app/reference/websockets

---

## 🎉 배포 성공 후

배포 URL을 다음 파일에 업데이트하세요:

1. **README.md**: 데모 URL 추가
2. **CLAUDE.md**: 배포 정보 업데이트
3. **package.json**: `homepage` 필드 추가

```json
{
  "homepage": "https://your-app.up.railway.app",
  ...
}
```

---

**Happy Deploying! 🚂✨**

Railway 배포 중 문제가 있으면 GitHub Issues 또는 Railway 커뮤니티에 문의하세요.

# AI 모니터 기술 사양서 (초안)

**문서 버전**: v0.1 (2026-07-06) | **작성**: 제품기획 | **관련 이슈**: ISSUE-20260706-01

---

## 1. 개요

AI 모니터는 SSL 가시성 장비와 연동하여 조직의 생성형 AI 사용 트래픽을 복호화 상태로 분석하는 네트워크 기반 AI 보안 제품이다. 프롬프트 수집·민감정보 탐지·Shadow AI 식별을 수행하며, Security Manager(SM)의 검사 규정과 AI 서비스 카탈로그를 공통 참조한다.

**전제 조건**: SSL 가시성 장비(F5 SSLO, A10, Symantec 등)와의 연결이 필수이며, 해당 장비가 TLS 복호화를 담당한다.

---

## 2. 네트워크 배치 구성

AI 모니터를 SSL 가시성 장비에 연결하는 방식은 두 가지다.

### 2.1 인라인 방식 (서비스 체인)

```
내부 사용자 ─→ SSL 가시성 장비 ─→ 방화벽 ─→ 생성형 AI 서비스(인터넷)
                 │        ↑
          복호화 트래픽   검사 후 반환
                 ↓        │
              AI 모니터 (검사 · 차단 · 마스킹)
```

- SSL 가시성 장비의 서비스 체인에 인라인 툴로 참여
- 복호화된 트래픽을 검사한 뒤 반환해야 재암호화·전송이 진행됨 → 실시간 차단·마스킹 가능

### 2.2 모니터링 방식 (Out-of-band)

```
내부 사용자 ─→ SSL 가시성 장비 ─→ 방화벽 ─→ 생성형 AI 서비스(인터넷)
                 ┆
          복호화 미러 트래픽 (단방향)
                 ↓
              AI 모니터 (탐지 · 로깅 · 경보)
```

- 원 트래픽은 그대로 통과, 복호화된 사본만 미러 포트로 수신
- 차단 불가(TCP RST 수준의 사후 대응만 가능), 네트워크 무영향

### 2.3 방식 비교

| 항목 | 인라인 | 모니터링 |
|------|--------|----------|
| 실시간 통제 | 차단·마스킹·정책 강제 가능 | 불가 (탐지·사후 대응) |
| 지연 | 검사만큼 추가 | 없음 |
| 장애 영향 | 통신 단절 위험 → Fail-open/Bypass·HA 필수 | 없음 (미러만 끊김) |
| 처리량 | AI 모니터 성능이 병목 가능 | 원 트래픽 무영향 |
| 도입 | 네트워크 변경·작업 창 필요 | 무중단 도입 |
| 적합 단계 | 정책 시행(유출 차단) | 가시성 확보 · PoC · 초기 도입 |

**권고**: 모니터링 방식으로 시작해 오탐률·정책을 검증한 뒤 인라인으로 전환하는 단계적 도입. 인라인 전환 시 AI 모니터 헬스체크 실패 시 자동 우회(fail-open)를 SSL 가시성 장비에 설정할 것.

---

## 3. 하드웨어 요구사항

| 항목 | 인라인 방식 | 모니터링 방식 |
|------|-------------|----------------|
| NIC | 인라인 포트 쌍(In/Out) × 구간 수, 저지연 고성능 NIC | 미러 수신용 캡처 NIC 1개 (promiscuous) |
| Bypass | 하드웨어 바이패스 NIC **필수** | 불필요 |
| 이중화 | HA 2대 사실상 필수 | 단일 서버 가능 |
| 성능 | 회선속도(line-rate) 처리 → 고사양 CPU, DPDK 등 패킷가속 | 부하 시 패킷 드롭 허용 → 상대적 저사양 |
| 형태 | 전용 어플라이언스 일반적 | 범용 서버/VM 가능 (VM은 SR-IOV 또는 vSwitch 포트 미러링) |

---

## 4. 제공 형태 (S/W 제품화)

| 제공 형태 | 모니터링 | 인라인(물리) | 인라인(ICAP 연동) |
|-----------|----------|--------------|---------------------|
| 순수 S/W 제공 | ◎ | ✕ (Bypass HW 필요) | ◎ |
| 차단 가능 | ✕ | ◎ | ◎ |

- **모니터링 S/W**: 고객사 서버/VM 설치 + SSL 가시성 장비 미러 포트 연결만으로 동작. S/W 장애가 네트워크에 무영향.
- **인라인(물리)**: 바이패스 NIC 없이는 S/W 장애 = 전사 인터넷 단절 → 어플라이언스(HW+SW) 형태 강제.
- **ICAP 연동**: 대부분의 SSL 가시성 장비가 지원하는 ICAP/프록시 체이닝으로 AI 모니터가 ICAP 서버로 동작. 물리 인라인 없이 허용/차단/수정 시행 가능, 장애 시 통과/차단 정책은 장비 측 설정으로 처리.

**S/W 제품 전략**: 미러 수신(모니터링) + ICAP 서버(차단) 2개 모드 지원으로 물리 어플라이언스 없이 전체 시나리오 커버.

---

## 5. Shadow AI 탐지 엔진

부하가 낮은 순서로 걸러내는 4계층 파이프라인. 각 단계에서 판정이 확정되면 다음 단계로 넘기지 않는다.

```
미러 트래픽 → ① 도메인/SNI 매칭 → ② API 패턴 → ③ 행위 분석 → ④ 콘텐츠 분석(샘플링)
```

| 계층 | 원리 | 탐지 대상 | 부하 | 비고 |
|------|------|-----------|------|------|
| ① 도메인/SNI 매칭 | AI 서비스 카탈로그 대조 | 알려진 서비스 | 매우 낮음 | 카탈로그 구독 갱신 필요 |
| ② API 패턴 | `/v1/chat/completions`, `messages`·`model` JSON 필드, SSE 구조 | 미등록·자체구축 LLM (Ollama, vLLM) | 낮음 | OpenAI 호환 API 공통 탐지 |
| ③ 행위 분석 | 요청/응답 크기 비대칭, 토큰 스트리밍 청크 리듬, 장시간 SSE/WebSocket | 복호화 불가(피닝) 트래픽 포함 | 낮음 | 유일한 미복호화 대응 수단 |
| ④ 콘텐츠 분석 | ML/LLM 분류기로 프롬프트성 텍스트 판별 | SaaS 내장 AI 포함 전체 | 높음 | 미확정 트래픽에 샘플링 적용 |

---

## 6. AI 서비스 식별 카탈로그 (주요 LLM 호출 URL)

| 제공사 (모델) | API 도메인 | 채팅 엔드포인트 |
|---------------|------------|------------------|
| OpenAI (GPT) | api.openai.com | `/v1/chat/completions`, `/v1/responses` |
| Anthropic (Claude) | api.anthropic.com | `/v1/messages` |
| Google (Gemini) | generativelanguage.googleapis.com | `/v1beta/models/{모델}:generateContent` |
| Google Vertex AI | {리전}-aiplatform.googleapis.com | `/v1/projects/.../models/{모델}:generateContent` |
| Azure OpenAI | {리소스}.openai.azure.com | `/openai/deployments/{배포명}/chat/completions` |
| AWS Bedrock | bedrock-runtime.{리전}.amazonaws.com | `/model/{모델ID}/invoke`, `/converse` |
| DeepSeek | api.deepseek.com | `/chat/completions`, `/anthropic/v1/messages` |
| xAI (Grok) | api.x.ai | `/v1/chat/completions` |
| Mistral | api.mistral.ai | `/v1/chat/completions` |
| Cohere | api.cohere.com | `/v2/chat` |
| Alibaba (Qwen) | dashscope.aliyuncs.com | `/compatible-mode/v1/chat/completions` |
| 네이버 (HyperCLOVA X) | clovastudio.stream.ntruss.com | `/v3/chat-completions/{HCX-005 등}` |
| OpenRouter | openrouter.ai | `/api/v1/chat/completions` |
| Groq | api.groq.com | `/openai/v1/chat/completions` |
| Perplexity | api.perplexity.ai | `/chat/completions` |
| 자체구축 (Ollama) | 사내 IP:11434 | `/api/chat`, `/v1/chat/completions` |
| 자체구축 (vLLM) | 임의 호스트:8000 등 | `/v1/chat/completions` |

**설계 유의사항**
1. 웹 UI 도메인(chatgpt.com, claude.ai, gemini.google.com 등)은 API 도메인과 다름 — 도메인 카탈로그에 별도 등재.
2. Azure OpenAI·Bedrock은 고객별 리소스명/리전 포함 → 와일드카드 패턴(`*.openai.azure.com`, `bedrock-runtime.*.amazonaws.com`) 필요.
3. 카탈로그는 갱신이 잦으므로 구독형 업데이트 체계 전제.

---

## 7. Security Manager 연동

- **AI 서비스 관리** (ISSUE-20260706-01): SM에서 AI 서비스 목록조회·신규 등록(URL/프로세스명), 정책적용 제품(AI 모니터·PC Proxy·Remote Web Proxy) 중복 선택. 제품 정책에서는 해당 제품에 적용된 AI 서비스만 통제 가능.
- **검사 규정 연동**: 정규식·개인정보 AI·대외비 AI·LLM결과 검사 규정을 SM에서 공통 배포받아 적용.
- **탐지 결과 연계**: Shadow AI 탐지 결과는 통합 모니터링·위험 사용자 스코어에 반영. 미등록 서비스 탐지 시 SM 등록 제안(차기 검토).

---

## 8. 비기능 요구사항 (추정 — 기술검토에서 확정)

| 항목 | 목표 |
|------|------|
| 처리량 | 미러 수신 기준 10Gbps (추정) |
| 세션 처리 | 동시 세션 10만 (추정) |
| 로그 보존 | 프롬프트·탐지 로그 1년 (규제 요건 확인 필요) |
| 가용성 | 모니터링 모드: 단일 노드 허용 / ICAP 모드: Active-Standby |
| 프라이버시 | 프롬프트 원문 저장 시 마스킹·암호화, 접근권한 분리 |

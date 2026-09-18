# 📘 Supertonic 완전 정리 가이드 (한국어)

> 이 문서는 Supertonic 레포지토리를 분석하고 정리한 내용입니다.
> 설치법부터 수익화 아이디어까지 한 번에 볼 수 있도록 구성했습니다.

---

## 🔗 관련 링크 모음

### 이 저장소
| 구분 | 주소 |
|---|---|
| **현재 저장소 (포크)** | https://github.com/bmshin94/supertonic |
| **원본 아카이브** | https://github.com/supertone-oss-archive/supertonic |
| **Python SDK (아카이브)** | https://github.com/supertone-oss-archive/supertonic-py |

### 모델 가중치 (Hugging Face)
| 버전 | 주소 |
|---|---|
| **Supertonic 3** (최신, 31개 언어) | https://huggingface.co/supertone-oss-archive/supertonic-3 |
| Supertonic 2 (5개 언어) | https://huggingface.co/supertone-oss-archive/supertonic-2 |
| Supertonic 1 (영어) | https://huggingface.co/supertone-oss-archive/supertonic |

### 참고 논문
- SupertonicTTS (메인 아키텍처): https://arxiv.org/abs/2503.23108
- Length-Aware RoPE (텍스트-음성 정렬): https://arxiv.org/abs/2509.11084
- Self-Purifying Flow Matching: https://arxiv.org/abs/2509.19091
- RobustSpeechFlow: https://arxiv.org/abs/2605.22083

---

## 1. 🎤 Supertonic이 뭔가요?

> **한 줄 요약: 글자를 넣으면 사람 목소리로 읽어주는 AI를, 인터넷 없이 내 컴퓨터에서 돌리는 프로그램**

```
"안녕하세요" 입력  →  🪄  →  음성 파일(WAV) 출력 🔊
```

만든 곳은 한국 오디오 AI 회사 **수퍼톤(Supertone Inc.)** 입니다.

### 일반 TTS와의 차이

| 항목 | 일반 클라우드 TTS | Supertonic |
|---|---|---|
| 인터넷 | 필요함 | **없어도 됨** ✈️ |
| 비용 | 글자 수만큼 과금 | **무료, 무제한** 💸 |
| 내 텍스트 | 외부 서버로 전송됨 | **내 기기 밖으로 안 나감** 🔒 |
| 속도 | 네트워크 왕복 지연 | **매우 빠름** ⚡ |
| GPU | 서버가 처리 | **CPU만으로 충분** |

### 핵심 스펙

| 포인트 | 내용 |
|---|---|
| ⚡ 속도 | 웹페이지 하나를 1초 안에 음성화할 정도 |
| 🪶 모델 크기 | **약 99M 파라미터** (0.7B~2B급 대비 10~20배 작음) |
| 📱 실행 환경 | 라즈베리파이, 전자책 리더기, 브라우저까지 |
| 🌍 언어 | **31개 언어** (한국어 포함, 모르면 `lang="na"`) |
| 🔊 음질 | 44.1kHz 16bit WAV (스튜디오급) |
| 🎭 감정 표현 | `<laugh>`, `<breath>`, `<sigh>` 등 10개 인라인 태그 |
| 🔒 프라이버시 | 클라우드 전송 없음, API 키 없음 |

---

## 2. 📁 폴더 구조

**핵심: 이 저장소에는 AI 모델 자체가 없습니다.** 모델(`.onnx`)은 Hugging Face에서 따로 받아 `assets/`에 넣는 구조입니다.

```
supertonic/
├── py/          🐍 파이썬 예제 (입문 추천!)
├── nodejs/      🟢 Node.js 서버용
├── web/         🌐 브라우저 (WebGPU/WASM) - 서버 없이 실행
├── java/        ☕ JVM
├── cpp/         ⚙️ C++
├── csharp/      🟣 .NET
├── go/          🔵 Go
├── rust/        🦀 Rust
├── swift/       🍎 macOS
├── ios/         📱 네이티브 iOS 앱 (SwiftUI 예제 앱 포함)
├── flutter/     💙 크로스플랫폼 앱
├── img/         📊 성능 비교 그래프
└── test_all.sh  🧪 전체 언어 예제 일괄 테스트 스크립트
```

> 💡 **폴더가 많아 보이지만 전부 같은 기능입니다.** 라면 조리법을 여러 언어로 써놓은 것과 같아요.
> 입문자는 **`py/` 하나만** 보면 충분합니다.

### 각 폴더의 공통 파일 패턴
- `helper.*` → 실제 엔진 로직 (텍스트 전처리 + ONNX 추론 파이프라인)
- `example_onnx.*` → 이를 사용하는 실행 예제
- `assets` → 루트 `assets/`를 가리키는 심볼릭 링크

### 내부 동작 파이프라인 (`py/helper.py` 기준)

ONNX 모델 **4개**가 순차적으로 동작합니다.

```
텍스트 입력
  → [전처리] 이모지 제거, 숫자·단위 정규화 ($5.2M → "five point two million")
  → duration_predictor.onnx  ⏱️ 각 글자의 발음 길이 예측
  → text_encoder.onnx        📝 텍스트를 의미 벡터로 변환
  → vector_estimator.onnx    🌊 Flow Matching 기반 노이즈 제거 (기본 8스텝)
  → vocoder.onnx             🔊 최종 파형 생성
  → result.wav 🎉
```

목소리는 `assets/voice_styles/M1.json` 같은 JSON 파일로 교체하는 방식입니다. (남성 M1~M5, 여성 F1~F5)

---

## 3. 🛠️ 설치 및 사용법

### 방법 A. 파이썬 (가장 쉬움, 추천)

#### STEP 0. 준비물
- **Python 3.11** (README 권장 버전)
- 디스크 여유 공간 약 500MB~1GB

#### STEP 1. 가상환경 만들기
```bash
cd supertonic
python3.11 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\Activate.ps1
```
> `venv`는 **프로젝트 전용 서랍장**입니다. 여기 설치한 패키지는 시스템 전체에 영향을 주지 않고,
> `.venv` 폴더만 지우면 깨끗하게 정리됩니다.

#### STEP 2. 모델 다운로드
```bash
pip install huggingface_hub

hf download supertone-oss-archive/supertonic-3 \
  --revision aafc6e32416a594460b32413efc49d7fe4ce6d46 \
  --local-dir assets
```

다운로드 후 구조:
```
assets/
├── onnx/
│   ├── duration_predictor.onnx   ⏱️ 길이 예측
│   ├── text_encoder.onnx         📝 텍스트 인코딩
│   ├── vector_estimator.onnx     🌊 음성 생성
│   └── vocoder.onnx              🔊 파형 출력
└── voice_styles/
    ├── M1.json ~ M5.json         👨 남성 목소리 5종
    └── F1.json ~ F5.json         👩 여성 목소리 5종
```

> 🔑 `--revision`의 커밋 해시는 **아카이브 시점 스냅샷을 고정**하기 위한 것입니다. 그대로 사용하세요.

#### STEP 3. 실행
```bash
pip install -r py/requirements.txt
cd py
python example_onnx.py --text "안녕하세요, 반갑습니다" --lang ko --n-test 1
```
→ `py/results/` 폴더에 WAV 파일이 생성됩니다. 🎉

#### 주요 옵션

| 옵션 | 설명 | 기본값 |
|---|---|---|
| `--text` | 읽을 문장 (여러 개 가능) | 영어 샘플 |
| `--lang` | 언어 코드 (`ko`, `en`, `ja`… 모르면 `na`) | `en` |
| `--voice-style` | 목소리 JSON 경로 | `../assets/voice_styles/M1.json` |
| `--speed` | 말 속도 (높을수록 빠름) | `1.05` |
| `--total-step` | 노이즈 제거 스텝 (높을수록 고품질·저속) | `8` |
| `--n-test` | 생성 횟수 | `4` |
| `--use-gpu` | GPU 사용 (없어도 무방) | 꺼짐 |
| `--batch` | 여러 문장 일괄 처리 | 꺼짐 |
| `--onnx-dir` | ONNX 모델 디렉터리 | `../assets/onnx` |
| `--save-dir` | 출력 디렉터리 | `results` |

**여성 목소리 한국어 예시**
```bash
python example_onnx.py \
  --text "오늘 날씨가 정말 좋네요" \
  --lang ko \
  --voice-style ../assets/voice_styles/F1.json \
  --speed 1.0 --n-test 1
```

### 방법 B. 브라우저에서 실행

```bash
cd web
npm install
npm run dev
```
→ 로컬 개발 서버(보통 `http://localhost:3000`)에서 바로 사용 가능합니다.

**서버가 전혀 필요 없습니다.** 브라우저 안에서 AI가 통째로 동작합니다.
- WebGPU 지원 시 → GPU 가속
- 미지원 시 → WebAssembly로 자동 전환

주요 의존성: `onnxruntime-web`, `fft.js`, 빌드 도구는 `vite`

### 방법 C. 다른 언어

```bash
cd nodejs && npm install && npm start          # Node.js
cd java   && mvn clean install && mvn exec:java # Java
cd go     && go mod download && go run example_onnx.go helper.go
cd rust   && cargo build --release && ./target/release/example_onnx
cd csharp && dotnet restore && dotnet run
cd swift  && swift build -c release && .build/release/example_onnx
```

> ⚠️ 일부 언어는 네이티브 런타임이 필요합니다.
> - **Go**: ONNX Runtime C 라이브러리 (`brew install onnxruntime`)
> - **Java**: JRE가 아닌 JDK (`brew install openjdk@17`)
> - **C#**: .NET 9 타겟

---

## 4. 🤔 플러그인? 스킬? MCP? → 전부 아님, **라이브러리**입니다

| 종류 | 정체 | 사용 주체 | 예시 |
|---|---|---|---|
| **플러그인** | 기존 앱에 끼우는 확장 | 앱 사용자 | 크롬 확장, VSCode 익스텐션 |
| **스킬** | AI에게 주는 사용설명서 📄 | AI | `.claude/skills/*.md` |
| **MCP** | AI가 외부 도구를 쓰게 하는 어댑터 🔌 | AI 에이전트 | GitHub MCP |
| **라이브러리** | 프로그램에 넣는 부품 🔩 | **개발자** | **← Supertonic** |

### 비유
- **Supertonic** = 라면 **면발** 🍜 (재료)
- **플러그인** = 끓인 라면에 추가하는 계란
- **MCP** = 로봇이 냄비를 잡게 해주는 집게 손 🦾
- **스킬** = 로봇에게 주는 레시피 카드 📝

### 중요 포인트
**Supertonic을 감싸서 MCP 서버로 만들 수 있습니다.**

```
Supertonic (부품) → 감싸기 → "말하기 MCP 서버" 🔌 → AI가 실제로 소리를 냄 🔊
```

실제 사례: **Aftertone** (Cursor·Claude Code의 답변을 온디바이스로 읽어주는 도구)
→ https://github.com/omarelkhal/aftertone

---

## 5. 🔑 API 토큰이 필요한가요? → **전혀 필요 없습니다**

| 항목 | 필요 여부 |
|---|---|
| Supertonic 사용 | ❌ 불필요 |
| 모델 다운로드 (Hugging Face) | ❌ 로그인조차 불필요 (공개 저장소) |
| 월 요금 | ❌ 0원 |
| 사용량 제한 | ❌ 무제한 |
| 인터넷 | ❌ **최초 다운로드 시에만** |

> README 원문: *"No Hugging Face login, hosted demo, or original Supertone service is required."*

### 비용 비교 예시
```
클라우드 TTS : 100만 글자 → 수만 원 💸 + 인터넷 필수 + 외부 서버 전송
Supertonic   : 100만 글자 → 0원 🎉 + 오프라인 동작 + 로컬 처리
```

> ⚠️ 참고: 예전 `pip install supertonic` 패키지는 자동 다운로드를 시도할 수 있으므로,
> 이 저장소의 `example_onnx.py`를 직접 쓰는 것이 가장 깔끔합니다.

---

## 6. 🌟 GitHub에서 유명한 이유

### ① "이 크기에 이 성능?"
```
일반 오픈소스 TTS : 700M ~ 2,000M 파라미터 🐘
Supertonic        :         99M 파라미터 🐁
```
10~20배 작은데도 벤치마크(Minimax-MLS-test) 성능이 대형 모델과 비슷합니다.
(영어 WER 2.06% — VoxCPM2의 2.11%보다 우수)

### ② GPU가 필요 없음
CPU만으로도 A100 GPU에서 측정한 대형 모델 대비 경쟁력 있는 속도를 냅니다. 진입장벽이 크게 낮아집니다.

### ③ 임팩트 있는 데모 영상
- 🍓 **라즈베리파이**에서 실시간 음성 합성
- 📖 **전자책 리더기**(Onyx Boox Go 6)에서 **비행기 모드**로 RTF 0.3배
- 🌐 **크롬 확장**으로 웹페이지 전체를 1초 내 음성화

### ④ 11개 런타임 예제 완비
Python / Node.js / 브라우저 / Java / C++ / C# / Go / Swift / iOS / Rust / Flutter
→ 어떤 개발자가 와도 바로 쓸 수 있어 사용자층이 넓습니다.

### ⑤ 숫자·단위를 정확하게 읽음

| 입력 | Supertonic | ElevenLabs / OpenAI / Gemini |
|---|:---:|:---:|
| `$5.2M` | ✅ | ❌ |
| `(212) 555-0142 ext. 402` | ✅ | ❌ |
| `30kph` | ✅ | ❌ |

상용 서비스들이 틀리는 케이스를 전처리 없이 정확히 처리합니다.

> 📊 README에 **Trendshift 배지**가 있어, 깃허브 급상승 화제 저장소로 선정된 이력이 있습니다.

---

## 7. 🤖 로컬 AI 에이전트 구축에 도움이 되나요? → **매우 유용**

### 음성 에이전트의 3단 구조
```
👂 귀 (STT)  → Whisper 등 (말 → 글자)
🧠 뇌 (LLM)  → Llama, Claude 등 (생각)
👄 입 (TTS)  → 👉 Supertonic 👈 (글자 → 말)
```
Supertonic은 **"입"** 담당입니다. 귀와 뇌는 별도 모델이 필요합니다.

### 장점
1. **완전 오프라인 파이프라인 구성 가능**
   ```
   Whisper.cpp (로컬) → Ollama (로컬) → Supertonic (로컬)
   = 인터넷 0, 요금 0, 정보 유출 0 🔒
   ```
2. **낮은 지연시간** — 네트워크 왕복이 없어 대화가 자연스러움 ⚡
3. **Node.js 구현체 제공** — `nodejs/helper.js`를 그대로 MCP 서버로 감싸기 쉬움

### 실제 사례 (README "Built with Supertonic")

| 프로젝트 | 설명 | 링크 |
|---|---|---|
| **Aftertone** | Cursor·Claude Code 답변 음성 출력 | https://github.com/omarelkhal/aftertone |
| **VoiceChat** | 브라우저 온디바이스 음성 LLM 챗봇 | https://github.com/irelate-ai/voice-chat |
| **TLDRL** | 웹페이지 읽어주는 크롬 확장 | Chrome Web Store |
| **Read Aloud** | 오픈소스 TTS 브라우저 확장 | https://github.com/ken107/read-aloud |
| **CopiloTTS** | Kotlin Multiplatform TTS SDK | https://github.com/sigmadeltasoftware/CopiloTTS |
| **Supertonic MNN** | MNN 기반 경량 라이브러리 | https://github.com/vra/supertonic-mnn |
| **Transformers.js** | Hugging Face JS 라이브러리 지원 | https://github.com/huggingface/transformers.js/pull/1459 |

---

## 8. 💰 수익화 아이디어

### 8-0. 먼저 냉정한 현실 체크

#### ⚔️ 유일한 무기 3가지

| 무기 | 의미 | 통하지 않는 곳 |
|---|---|---|
| 🔒 **프라이버시** | 데이터가 외부로 안 나감 | 어차피 공개할 콘텐츠 |
| 💸 **한계비용 0원** | 무제한 사용 가능 | 소량만 쓰는 사용자 |
| ✈️ **오프라인** | 인터넷 없이 작동 | 항상 온라인인 환경 |

#### 💀 약점 (반드시 인지)

| 약점 | 심각도 |
|---|---|
| 🎭 **목소리 10개 고정** | ⚠️⚠️⚠️ Voice Builder 종료로 **음성 복제 불가** |
| 🪦 **개발 중단(아카이브)** | ⚠️⚠️ 버그 발생 시 직접 해결해야 함 |
| 😐 **섬세한 감정 연기** | ⚠️ 최상급 상용 서비스 대비 부족 |
| 📦 **모델 용량** | ⚠️ 앱 용량 증가, 웹 첫 로딩 지연 |

#### 🚨 하지 말아야 할 것
```
❌ "ElevenLabs보다 싼 TTS 서비스"  → 목소리 다양성에서 밀림
❌ "AI 성우 마켓플레이스"          → 목소리 10개로는 불가능
❌ "음성 복제 서비스"              → 기능 자체가 없음
❌ 클라우드에 올려 API로 판매       → 로컬의 강점을 스스로 포기
```

> 💡 **핵심 원칙: Supertonic은 "클라우드를 못 쓰는 상황"에서만 이깁니다.**

### 8-1. 수익 모델 유형

| 유형 | 방식 | 장점 | 단점 | 궁합 |
|---|---|---|---|---|
| 🅰️ **1회 구매** | 앱/툴 판매 | 간단, 신뢰 확보 | 반복 수익 없음 | ⭐⭐⭐⭐ |
| 🅱️ **구독** | 월정액 | 안정적 | 지속 업데이트 부담 | ⭐⭐ |
| 🅲 **프리미엄** | 무료+유료 | 사용자 확보 용이 | 전환율 낮음(1~5%) | ⭐⭐⭐ |
| 🅳 **B2B 구축** | 기업 납품 | **단가 최고** 💼 | 영업 난이도 높음 | ⭐⭐⭐⭐⭐ |
| 🅴 **대행 서비스** | 직접 작업 | 즉시 현금화 | 시간이 한계 | ⭐⭐⭐ |

> 🏆 **서버비 0원이라 "한 번 사면 평생 무제한"이라는 파격 조건이 가능합니다.**
> 클라우드 기반 경쟁사는 절대 따라올 수 없는 조건입니다.

### 8-2. 아이디어 상세

#### 💡 ① 웹소설·웹툰 읽어주는 크롬 확장 📖
- **타겟**: 웹소설 헤비 독자, 출퇴근 청취자, 눈이 피로한 직장인
- **문제**: 기존 TTS 확장은 월 구독 또는 글자 수 제한 → 웹소설은 하루 수십만 자라 무료 플랜 순삭
- **수익 모델**
  ```
  무료: 기본 목소리 2개 + 표준 속도
  Pro (약 ₩9,900 평생): 목소리 10종 전체 + 속도 조절 + 북마크 + 자동 스크롤
  ```
- **수익 시뮬레이션** (전환율 1~5% 가정한 예시)

  | 시나리오 | 설치자 | 전환율 | 월 수익(추정) |
  |---|---|---|---|
  | 😢 실패 | 300명 | 2% | 약 6만원 |
  | 🙂 보통 | 3,000명 | 3% | 약 89만원 |
  | 🤩 성공 | 30,000명 | 4% | 약 1,188만원 |

- **난이도**: 🟢🟢⚪⚪⚪ (2/5) — `web/` 코드를 확장으로 이식, 1~2주면 MVP
- **리스크**: TLDRL·Read Aloud 등 경쟁자 존재 → **한국어 웹소설 특화로 좁힐 것**. 모델 용량 대응 필요

#### 💡 ② 쇼츠 제작자용 나레이션 툴 🎬
- **타겟**: 유튜브 쇼츠·릴스 대량 생산자
- **문제**: 구독형 TTS의 글자 수 소진 → 매달 고정비가 수익 잠식
- **수익 모델**: 데스크톱 앱 약 ₩39,000 (평생) — *"월 구독 없음, 무제한 생성"* 이 핵심 문구
- **필수 기능**: 대본 배치 생성(`--batch`), SRT 자막 자동 생성, BGM 믹싱, 속도·톤 프리셋
- **난이도**: 🟡🟡🟡⚪⚪ (3/5) — Python + Electron/Tauri
- **리스크**: 목소리 10개라 "다른 채널과 목소리 중복" 문제 → 속도·피치 변주로 대응

#### 💡 ③ 어린이 학습 앱 (오프라인) 👶
- **타겟**: 유아~초등 학부모, 유치원·학원
- **왜 통하나**: 학부모의 2대 불안 — *"아이 데이터가 서버에 저장되나?"*, *"데이터 요금이 나오나?"* → **둘 다 해결**
- **수익 모델**
  ```
  B2C: 앱 구매 약 ₩12,000 (평생, 광고 없음)
  B2B: 유치원·학원 라이선스 연 30~100만원
  ```
- **확장성**: 31개 언어 지원 → 다국어 학습(한글 자모, 영어 파닉스, 중국어·일본어 발음)
- **난이도**: 🟡🟡🟡⚪⚪ (3/5) — `ios/` 폴더에 SwiftUI 예제 앱 완비
- **리스크**: 아동 카테고리 앱 심사 규정 확인 필수, 발음 검수 필요

#### 💡 ④ 접근성(배리어프리) 솔루션 ♿
- **타겟**: 시각장애인·저시력자, **공공기관·도서관·지자체**
- **강점**: 한국은 공공 웹 접근성 의무로 예산 배정, 사회적기업·장애인기업 지원사업 활용 가능
- **수익 모델**
  ```
  기관 납품: 구축 500만~3,000만원 + 유지보수 연 20%
  개인 사용자: 무료 배포 (공익성이 곧 영업 자산)
  ```
- **난이도**: 🟡🟡🟡🟡⚪ (4/5) — 기술보다 행정·영업이 관건
- **리스크**: 입찰 자격·실적 요구, 긴 결제 사이클 → **무료 오픈소스 배포로 실적 선확보** 권장

#### 💡 ⑤ B2B 온디바이스 TTS 구축 🏢 (최고 단가)
- **타겟**: 클라우드 AI를 못 쓰는 조직

  | 업종 | 클라우드 불가 사유 |
  |---|---|
  | 🏥 병원 | 환자 정보 = 민감정보, 외부 전송 금지 |
  | 🏦 금융 | 망분리·금융보안 규제 |
  | 🏛️ 공공·국방 | 폐쇄망 운영 |
  | 🏭 공장 | 인터넷 없는 현장, 소음 환경 안내방송 |
  | 🚗 차량·선박 | 통신 단절 환경 |

- **수익 규모**
  ```
  초기 구축: 1,000만 ~ 5,000만원
  유지보수:  구축비의 15~25% / 년
  추가 개발: 건당 별도
  ```
- **영업 포인트**
  > "인터넷 연결이 전혀 없어도 작동하며, 입력된 모든 텍스트는 기관 내부 서버 밖으로 나가지 않습니다.
  > 사용량 과금이 없어 아무리 많이 사용해도 추가 비용이 0원입니다."
- **난이도**: 🔴🔴🔴🔴🔴 (5/5) — 기술보다 영업·신뢰·계약이 벽
- **리스크** ⚠️⚠️⚠️: **아카이브 프로젝트 납품 = 유지보수 책임 100% 본인 부담**
  → 계약서에 유지보수 범위 명시, **ONNX 모델 파일 자체 백업 보관 필수**

#### 💡 ⑥ 개발자용 도구 🛠️ (진입 최적)
- **타겟**: 개발자

  | 제품 | 설명 | 가격 |
  |---|---|---|
  | 🔌 음성 MCP 서버 | Claude·Cursor 답변 음성 출력 | 오픈소스 + 후원 |
  | 📢 CI/CD 음성 알림 | 빌드 실패 시 음성 알림 | 무료(홍보용) |
  | 📦 npm/pip 래퍼 | 3줄로 TTS 붙이는 SDK | 오픈소스 + 스폰서 |

- **진짜 목적**
  ```
  오픈소스 공개 → ⭐ GitHub 스타 → 🏆 인지도 → 💼 B2B 문의 유입
                                                ↑ 실제 수익
  ```
- **난이도**: 🟢🟢⚪⚪⚪ (2/5) — 가장 만만함

### 8-3. ⚖️ 라이선스 체크리스트 (필수 확인)

| 항목 | 라이선스 | 상업 이용 |
|---|---|---|
| 이 저장소 **코드** | **MIT** | ✅ 자유 (저작권 표시만) |
| **모델 가중치** | **OpenRAIL-M** | ⭕ 가능하나 **조건부** |

#### OpenRAIL-M 금지 사항
```
❌ 실존 인물 목소리 사칭 / 딥페이크
❌ 사기, 허위정보, 스팸 생성
❌ 상대방이 AI임을 모르게 속이기
❌ 차별·혐오 콘텐츠 생성
```

#### 안전 수칙
1. 📄 제품 약관에 OpenRAIL-M 사용 제한 조항 포함
2. 🤖 "AI가 생성한 음성입니다" 명시
3. ⚖️ 규모가 커지면 변호사 검토 (이 문서는 법률 자문이 아닙니다)
4. 📖 원문 확인: https://huggingface.co/supertone-oss-archive/supertonic-3/blob/main/LICENSE

> ⚠️ **별도 이슈**: 저작권 있는 콘텐츠(웹소설·뉴스 등)를 변환해 **판매**하는 것은 저작권 문제가 있습니다.
> "사용자가 자기 화면의 글을 읽는 도구"는 괜찮지만, "변환물 판매"는 위험합니다.

### 8-4. 🗺️ 추천 로드맵

```
📅 0~1개월: 오픈소스 개발자 도구 1개 공개 (수익 0, 자산 축적) 🌱
            예) 음성 MCP 서버, React용 TTS 훅 패키지
            얻는 것: GitHub 스타, 기술 검증, 포트폴리오, 커뮤니티 피드백
                    ↓
📅 1~3개월: 크롬 확장 or 데스크톱 툴 유료화 (₩9,900~39,000 평생) 💵
            얻는 것: 첫 매출, 결제·배포 파이프라인, 실사용자 피드백
                    ↓
📅 3~6개월: 반응 좋은 쪽으로 집중 📈
            B2C 반응 good → 기능 확장 + 마케팅
            B2B 문의 발생 → 피벗 (단가 10배 이상)
```

**추천 순서: ⑥ 개발자 도구 → ① 크롬 확장 → ⑤ B2B**
각 단계가 다음 단계의 재료가 되어 버리는 것이 없습니다.

### 8-5. ⚠️ 마지막 당부

| 💀 실패 패턴 | 💚 성공 패턴 |
|---|---|
| "ElevenLabs보다 싸게!" → 품질에서 밀림 | "클라우드 못 쓰는 사람"만 노리기 🎯 |
| 완벽하게 만들려다 6개월 경과 | 2주 안에 일단 출시 → 반응 보고 개선 🚀 |
| 기술만 파고 마케팅 0 | "월 구독 없음·무제한·오프라인" 전면 배치 💥 |

---

## 9. ⚛️🐘 React / PHP로 만들 수 있나요?

### ⚛️ React — **가능하며 매우 쉬움**

`web/` 폴더가 이미 브라우저용 + **Vite 기반**이라 이식이 간단합니다.

**현재 의존성**
```json
"dependencies": {
  "onnxruntime-web": "^1.17.0",   // 브라우저 AI 추론 엔진
  "fft.js": "^4.0.3"              // 신호 처리
},
"devDependencies": { "vite": "^5.0.0" }   // React와 동일한 빌드 도구
```

**이식 방법**
```
web/helper.js   → 그대로 복사 (순수 JS, 프레임워크 무관) ✅
web/main.js     → React 컴포넌트로 변환 (UI 부분만)
assets/         → public/assets/ 로 이동
```

**사용 예시**
```jsx
const { speak, isLoading } = useSupertonic();

<button onClick={() => speak("안녕하세요", "ko")} disabled={isLoading}>
  🔊 말하기
</button>
```

- ✅ **장점**: 서버 불필요, Vercel·Netlify 정적 배포만으로 동작
- ⚠️ **주의**: 모델 로딩 몇 초 소요 → 로딩 UI 필수 / `SharedArrayBuffer` 관련 **COOP·COEP 헤더** 설정 필요할 수 있음 / 모델 파일 용량 확인

### 🐘 PHP — **직접은 불가, 우회 필요**

ONNX Runtime은 Python·JS·Java·C++·C#·Go·Rust·Swift는 지원하지만 **PHP 공식 바인딩이 없습니다.**

#### 우회 방법 3가지

**방법 1. PHP가 파이썬 호출** (가장 간단)
```php
<?php
$text = escapeshellarg($_POST['text']);   // 🔒 보안 필수
exec("cd /path/py && python example_onnx.py --text $text --lang ko --n-test 1");
echo "<audio controls src='/results/output.wav'></audio>";
```
> ⚠️ 사용자 입력을 그대로 셸에 넘기면 명령어 삽입 취약점이 생깁니다. `escapeshellarg()` 필수.
> ⚠️ 요청마다 모델을 새로 로딩해 느립니다.

**방법 2. 파이썬 API 서버 분리** (권장 ⭐⭐⭐)
```
[PHP 웹사이트] --HTTP--> [Python TTS 서버] --> WAV
   화면 담당                AI 담당 (모델 상주)
```
- 모델이 메모리에 상주해 훨씬 빠름 ⚡
- 역할 분리로 유지보수 용이
- 참고: 과거 Python SDK에 `supertonic serve` (로컬 HTTP 서버, `/v1/tts` 및 OpenAI 호환 `/v1/audio/speech`) 기능이 있었습니다

**방법 3. PHP는 화면만, 추론은 브라우저에서** (효율 최고 ⭐⭐)
```
[PHP] → HTML 제공
          ↓
[사용자 브라우저] → onnxruntime-web으로 직접 음성 생성 🔊
```
- **서버 부하 0**, 서버 비용 절감

---

## 10. ⚠️ 반드시 알아둘 주의사항

| 항목 | 내용 |
|---|---|
| 🪦 **아카이브 상태** | 2026년 7월 개발·지원 종료. 업데이트·버그픽스·**보안 패치 없음** |
| 🔗 **Voice Builder 종료** | 2026년 8월 31일 종료 → **커스텀 음성 생성 불가**, 프리셋 10종만 사용 가능 |
| 📜 **이중 라이선스** | 코드 MIT / 모델 OpenRAIL-M — 상업화 시 반드시 구분해서 확인 |
| 📌 **리비전 고정** | `--revision` 해시는 아카이브 스냅샷 지정용, 변경 금지 |
| 💾 **모델 백업** | 상업 이용 시 ONNX 파일을 직접 백업 보관 권장 |

> 그래도 **이미 받아둔 모델은 계속 정상 작동합니다.** 새 기능이 추가되지 않을 뿐,
> 사용에는 문제가 없습니다.

---

## 11. 📊 한눈에 보는 요약

| 질문 | 답변 |
|---|---|
| 이게 뭐야? | 오프라인·무료 **TTS(글자→음성) 엔진** 🎤 |
| 설치는? | Python 3.11 + 모델 다운로드 → 약 5분 ✅ |
| 정체는? | 플러그인❌ 스킬❌ MCP❌ → **라이브러리(부품)** 🔩 |
| API 토큰? | **전혀 불필요**, 무료 무제한 🎉 |
| 유명한 이유? | 작고·빠르고·GPU 불필요·11개 런타임 지원 🌟 |
| 로컬 에이전트? | **"입(TTS)" 역할로 최적** 👄 |
| 수익화? | 오프라인·프라이버시 시장 공략 필수 💰 |
| React? | **매우 쉬움** ⚛️ (브라우저 예제 존재) |
| PHP? | 직접 불가 → **Python 서버 분리**로 우회 🐘 |

---

## 12. 🚀 다음에 해볼 만한 것

- [ ] 🔌 **음성 MCP 서버** — Claude Code 답변을 음성으로 읽어주는 도구
- [ ] ⚛️ **React 훅 패키지** — `useSupertonic()` 형태의 재사용 컴포넌트
- [ ] 🌐 **크롬 확장** — 웹페이지·웹소설 읽어주기 (한국어 특화)
- [ ] 📱 **iOS 앱** — `ios/ExampleiOSApp` 기반으로 개조
- [ ] 🧪 **test_all.sh 실행** — 11개 런타임 전체 동작 확인

---

*이 문서는 Supertonic 저장소 분석 결과를 정리한 것입니다.*
*저장소: https://github.com/bmshin94/supertonic*

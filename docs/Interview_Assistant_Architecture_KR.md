# 인터뷰 도우미 프로젝트 아키텍처 설계

> 실시간 인터뷰 지원 + AI 기반 인터뷰 분석 리포트 생성

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [시스템 요구사항](#2-시스템-요구사항)
3. [Phase 1: 실시간 인터뷰](#3-phase-1-실시간-인터뷰)
4. [Phase 2: 인터뷰 후 분석](#4-phase-2-인터뷰-후-분석)
5. [모델 서빙 구성](#5-모델-서빙-구성)
6. [코드 구조](#6-코드-구조)
7. [핵심 구현 코드](#7-핵심-구현-코드)
8. [Tenstorrent P100 활용 가이드](#8-tenstorrent-p100-활용-가이드)
9. [시작 가이드](#9-시작-가이드)

---

## 1. 프로젝트 개요

### 1.1 목표

인터뷰 프로세스 전반을 AI로 지원하는 시스템 구축:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    인터뷰 도우미 프로젝트 개요                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 1: 실시간 인터뷰 진행 지원                                     │
│  ════════════════════════════════                                   │
│  • 실시간 STT → Live Transcript 표시                                │
│  • LLM이 답변 분석 → 팔로업 질문 제안                                │
│  • 요구사항: 낮은 지연시간 (실시간성)                                 │
│                                                                     │
│  Phase 2: 인터뷰 후 분석 및 리포트                                    │
│  ════════════════════════════════                                   │
│  • 영상 → VLM → 비언어적 표현 분석 (표정, 제스처)                     │
│  • 음성 → STT → 타임스탬프 포함 전체 전사                            │
│  • LLM → 두 결과 종합 → 최종 리포트 생성                             │
│  • 요구사항: 정확도 (시간 여유 있음)                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 필요한 AI 모델

| 용도 | 모델 타입 | 권장 모델 |
|------|----------|----------|
| 실시간 음성 인식 | STT | Whisper large-v3-turbo |
| 배치 음성 전사 | STT | Whisper large-v3 |
| 팔로업 질문 생성 | LLM | Llama3-8B, Qwen2.5-7B |
| 비언어 분석 | VLM | LLaVA-13B, Qwen2-VL |
| 리포트 생성 | LLM | Llama3-70B, Qwen2.5-32B |

---

## 2. 시스템 요구사항

### 2.1 하드웨어 권장사양

| 구성 | 최소 | 권장 | 쾌적 |
|------|------|------|------|
| **GPU VRAM** | 16GB | 24GB | 48GB+ |
| **System RAM** | 32GB | 64GB | 128GB |
| **Storage** | 100GB SSD | 500GB NVMe | 1TB NVMe |
| **GPU 예시** | RTX 4080 | RTX 4090 | 2x RTX 4090 |

### 2.2 Phase별 메모리 사용량

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GPU 메모리 사용량 예측                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 1 (실시간)                                                    │
│  ┌────────────────────────────────────────────┐                    │
│  │ RealtimeSTT (Whisper)  │████ 2GB           │                    │
│  │ Ollama LLM (8B)        │██████████ 5GB     │                    │
│  │ 버퍼/시스템             │██ 1GB             │                    │
│  └────────────────────────────────────────────┘                    │
│  총: ~8GB                                                           │
│                                                                     │
│  Phase 2 (배치) - 순차 실행                                          │
│  ┌────────────────────────────────────────────┐                    │
│  │ faster-whisper         │██████ 3GB         │                    │
│  │ Ollama VLM (13B)       │████████████████ 8GB│                   │
│  │ Ollama LLM (70B-q4)    │████████████████████████ 12GB│          │
│  └────────────────────────────────────────────┘                    │
│  총: ~12-20GB (순차 실행 시)                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Phase 1: 실시간 인터뷰

### 3.1 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Phase 1: 실시간 인터뷰 흐름                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   🎤 마이크 입력                                                     │
│      │                                                              │
│      ▼                                                              │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  RealtimeSTT                                                 │  │
│   │  • Whisper large-v3-turbo                                   │  │
│   │  • Silero VAD (음성 활동 감지)                                │  │
│   │  • 실시간 부분 결과 + 문장 완성 결과                           │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│            ┌─────────────────┼─────────────────┐                   │
│            │                 │                 │                   │
│            ▼                 ▼                 ▼                   │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐         │
│   │ Live        │   │ 답변 버퍼   │   │ 녹화/녹음 저장   │         │
│   │ Transcript  │   │ (문장 누적) │   │ (Phase 2용)     │         │
│   │ (UI 표시)   │   └──────┬──────┘   └─────────────────┘         │
│   └─────────────┘          │                                       │
│                            ▼ (문장 완성 시)                         │
│                   ┌─────────────────┐                              │
│                   │    Ollama LLM   │                              │
│                   │  (llama3:8b)    │                              │
│                   │                 │                              │
│                   │  프롬프트:       │                              │
│                   │  • 인터뷰 맥락   │                              │
│                   │  • 이전 대화     │                              │
│                   │  • 현재 답변     │                              │
│                   └────────┬────────┘                              │
│                            │                                       │
│                            ▼                                       │
│                   ┌─────────────────┐                              │
│                   │ 팔로업 질문 제안 │                              │
│                   │ (UI 표시)       │                              │
│                   └─────────────────┘                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 주요 컴포넌트

| 컴포넌트 | 기술 | 역할 |
|----------|------|------|
| **RealtimeSTT** | Python 라이브러리 | 실시간 음성→텍스트 변환 |
| **Silero VAD** | RealtimeSTT 내장 | 음성 활동 감지 (말할 때만 처리) |
| **Ollama** | LLM 서버 | 팔로업 질문 생성 |
| **녹화 모듈** | OpenCV + PyAudio | 영상/음성 저장 |

### 3.3 지연시간 목표

| 구간 | 목표 | 비고 |
|------|------|------|
| 음성 → 텍스트 | < 500ms | 실시간 부분 결과 |
| 문장 완성 감지 | < 1초 | VAD 기반 |
| 팔로업 질문 생성 | < 3초 | LLM 추론 |

---

## 4. Phase 2: 인터뷰 후 분석

### 4.1 파이프라인 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Phase 2: 인터뷰 후 분석 파이프라인                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   📁 입력 파일                                                       │
│   ├── interview.mp4 (영상)                                          │
│   ├── interview.wav (음성)                                          │
│   └── context.json (인터뷰 맥락 정보)                                 │
│                                                                     │
│   ══════════════════════════════════════════════════════════════   │
│                                                                     │
│   [Stage 1: 음성 분석] ─────────────────────────────────────────    │
│                                                                     │
│   interview.wav                                                     │
│        │                                                            │
│        ▼                                                            │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  faster-whisper (large-v3)                                   │  │
│   │  • word_timestamps=True                                      │  │
│   │  • 화자 분리 (선택적)                                         │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  타임스탬프 포함 전사 결과                                     │  │
│   │  ─────────────────────────────────────────────────────────  │  │
│   │  [00:00:05.2] 면접관: 자기소개 부탁드립니다.                    │  │
│   │  [00:00:08.5] 지원자: 네, 안녕하세요. 저는...                  │  │
│   │  [00:00:45.1] 면접관: 그 경험에서 무엇을 배우셨나요?            │  │
│   │  [00:00:48.3] 지원자: 그때 저는...                            │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   ══════════════════════════════════════════════════════════════   │
│                                                                     │
│   [Stage 2: 영상 분석] ─────────────────────────────────────────    │
│                                                                     │
│   interview.mp4                                                     │
│        │                                                            │
│        ▼                                                            │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  프레임 추출 (OpenCV/ffmpeg)                                  │  │
│   │  • 2초 간격 키프레임                                          │  │
│   │  • 표정 변화 감지 시 추가 추출 (선택적)                         │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  Ollama VLM (llava:13b / qwen2-vl)                           │  │
│   │                                                              │  │
│   │  분석 항목:                                                   │  │
│   │  • 표정 (자신감, 긴장, 미소, 진지함)                           │  │
│   │  • 시선 (정면 응시, 회피, 생각하는 모습)                        │  │
│   │  • 제스처 (손 움직임, 고개 끄덕임)                             │  │
│   │  • 자세 (바른 자세, 기울임, 방어적 자세)                        │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  비언어적 분석 결과                                           │  │
│   │  ─────────────────────────────────────────────────────────  │  │
│   │  [00:00:08] 자신감 있는 미소, 정면 응시, 바른 자세              │  │
│   │  [00:00:48] 잠시 시선 회피, 생각하는 표정                      │  │
│   │  [00:01:15] 적극적인 손 제스처, 열정적인 표현                   │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   ══════════════════════════════════════════════════════════════   │
│                                                                     │
│   [Stage 3: 종합 리포트 생성] ──────────────────────────────────    │
│                                                                     │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│   │ 음성 전사    │  │ 비언어 분석  │  │ 맥락 정보    │               │
│   │ 결과        │  │ 결과        │  │ (context)   │               │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘               │
│          │                │                │                       │
│          └────────────────┼────────────────┘                       │
│                           │                                        │
│                           ▼                                        │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  Ollama LLM (llama3:70b-q4 / qwen2.5:32b)                    │  │
│   │                                                              │  │
│   │  리포트 구성:                                                 │  │
│   │  1. 인터뷰 전체 요약                                          │  │
│   │  2. 언어적 분석 (내용, 논리성, 구체성, STAR 기법 활용도)        │  │
│   │  3. 비언어적 분석 (자신감, 진정성, 일관성)                      │  │
│   │  4. 타임라인별 주요 포인트                                     │  │
│   │  5. 강점 및 개선점                                            │  │
│   │  6. 종합 평가 및 제안                                         │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  📊 최종 인터뷰 분석 리포트 (Markdown/PDF)                     │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 분석 항목 상세

#### 언어적 분석

| 분석 항목 | 설명 | 평가 기준 |
|----------|------|----------|
| **내용 충실도** | 질문에 대한 답변의 적절성 | 질문 의도 파악, 핵심 전달 |
| **논리성** | 답변의 구조와 흐름 | 두괄식/미괄식, 인과관계 |
| **구체성** | 예시와 수치의 활용 | STAR 기법, 정량적 성과 |
| **표현력** | 어휘 선택과 문장 구성 | 전문용어, 명확한 표현 |

#### 비언어적 분석

| 분석 항목 | 설명 | 긍정 신호 | 부정 신호 |
|----------|------|----------|----------|
| **표정** | 얼굴 표현 | 자연스러운 미소, 진지함 | 경직, 과도한 긴장 |
| **시선** | 눈 맞춤 | 적절한 아이컨택 | 지속적 회피, 두리번거림 |
| **제스처** | 손/몸 움직임 | 적절한 강조 | 과도/부족, 방어적 |
| **자세** | 앉은 자세 | 바른 자세, 약간 앞으로 | 기울임, 팔짱 |

---

## 5. 모델 서빙 구성

### 5.1 Option A: 단일 서버 구성 (권장)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   단일 서버 구성 (24GB+ VRAM)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                     Application Layer                        │  │
│   │  ┌─────────────────┐      ┌─────────────────┐              │  │
│   │  │ Phase 1 App     │      │ Phase 2 App     │              │  │
│   │  │ (실시간 인터뷰)  │      │ (배치 분석)      │              │  │
│   │  └────────┬────────┘      └────────┬────────┘              │  │
│   └───────────┼─────────────────────────┼───────────────────────┘  │
│               │                         │                          │
│   ┌───────────┼─────────────────────────┼───────────────────────┐  │
│   │           │    Model Serving Layer  │                        │  │
│   │           ▼                         ▼                        │  │
│   │  ┌─────────────────┐      ┌─────────────────┐              │  │
│   │  │  RealtimeSTT    │      │  faster-whisper │              │  │
│   │  │  (내장 Whisper) │      │  (Python)       │              │  │
│   │  │  ~2GB           │      │  ~3GB           │              │  │
│   │  └─────────────────┘      └─────────────────┘              │  │
│   │                                                              │  │
│   │  ┌───────────────────────────────────────────────────────┐  │  │
│   │  │                    Ollama Server                       │  │  │
│   │  │                    (Port 11434)                        │  │  │
│   │  │                                                        │  │  │
│   │  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │  │  │
│   │  │  │ llama3:8b    │ │ llava:13b    │ │ llama3:70b-q4│  │  │  │
│   │  │  │ (팔로업 질문) │ │ (비언어 분석) │ │ (리포트)     │  │  │  │
│   │  │  │ ~5GB         │ │ ~8GB         │ │ ~12GB        │  │  │  │
│   │  │  └──────────────┘ └──────────────┘ └──────────────┘  │  │  │
│   │  │                                                        │  │  │
│   │  │  OLLAMA_MAX_LOADED_MODELS=2  (동시 로드 모델 수)        │  │  │
│   │  └───────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   ⚠️ Phase 1과 Phase 2는 동시 실행하지 않음 (메모리 절약)             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Option B: 분리 서버 구성 (다중 GPU)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   분리 서버 구성 (다중 GPU)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  GPU 0: STT 전용                   GPU 1: LLM/VLM 전용       │  │
│   │  ┌───────────────────┐            ┌───────────────────┐     │  │
│   │  │  STT Server       │            │  Ollama Server    │     │  │
│   │  │  (Port 8001)      │            │  (Port 11434)     │     │  │
│   │  │                   │            │                   │     │  │
│   │  │  • RealtimeSTT    │◄──────────►│  • llama3:8b      │     │  │
│   │  │  • faster-whisper │   API      │  • llama3:70b-q4  │     │  │
│   │  │                   │            │  • llava:13b      │     │  │
│   │  └───────────────────┘            └───────────────────┘     │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   ✅ Phase 1과 Phase 2 동시 실행 가능                                │
│   ✅ 각 서버 독립적 스케일링 가능                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 설치 명령어

```bash
# 1. Ollama 설치
curl -fsSL https://ollama.ai/install.sh | sh

# 2. 모델 다운로드
ollama pull llama3:8b           # Phase 1: 팔로업 질문
ollama pull llama3:70b-q4       # Phase 2: 리포트 (양자화)
ollama pull llava:13b           # Phase 2: 비언어 분석

# 3. Python 환경 설정
pip install RealtimeSTT         # 실시간 STT
pip install faster-whisper      # 배치 STT
pip install openai              # Ollama API 클라이언트
pip install opencv-python       # 영상 처리
pip install pyaudio             # 오디오 캡처

# 4. Ollama 서버 설정
export OLLAMA_MAX_LOADED_MODELS=2
ollama serve
```

---

## 6. 코드 구조

```
interview-assistant/
│
├── config/
│   ├── __init__.py
│   └── settings.py              # 모델, 서버, 프롬프트 설정
│
├── services/
│   ├── __init__.py
│   ├── stt/
│   │   ├── __init__.py
│   │   ├── realtime_stt.py      # Phase 1: 실시간 STT
│   │   └── batch_stt.py         # Phase 2: 배치 STT (타임스탬프)
│   │
│   ├── llm/
│   │   ├── __init__.py
│   │   ├── client.py            # Ollama 클라이언트 래퍼
│   │   ├── followup.py          # Phase 1: 팔로업 질문 생성
│   │   └── report.py            # Phase 2: 리포트 생성
│   │
│   └── vlm/
│       ├── __init__.py
│       └── video_analyzer.py    # Phase 2: 비언어 분석
│
├── pipelines/
│   ├── __init__.py
│   ├── interview_session.py     # Phase 1 파이프라인
│   └── post_analysis.py         # Phase 2 파이프라인
│
├── utils/
│   ├── __init__.py
│   ├── video.py                 # 프레임 추출, 영상 처리
│   ├── audio.py                 # 오디오 처리, 녹음
│   └── recorder.py              # 인터뷰 녹화 관리
│
├── prompts/
│   ├── followup_question.txt    # 팔로업 질문 프롬프트
│   ├── nonverbal_analysis.txt   # 비언어 분석 프롬프트
│   └── report_generation.txt    # 리포트 생성 프롬프트
│
├── outputs/                     # 생성된 리포트 저장
│
├── main.py                      # CLI 엔트리포인트
├── requirements.txt
└── README.md
```

---

## 7. 핵심 구현 코드

### 7.1 Phase 1: 실시간 인터뷰 세션

```python
# pipelines/interview_session.py

from RealtimeSTT import AudioToTextRecorder
from openai import OpenAI
from datetime import datetime
import threading
import queue

class InterviewSession:
    """실시간 인터뷰 진행 지원 클래스"""

    def __init__(self, context: dict, ollama_url: str = "http://localhost:11434/v1"):
        self.context = context  # 인터뷰 맥락 (직무, 질문 목록 등)
        self.ollama = OpenAI(base_url=ollama_url, api_key="ollama")

        self.transcript = []  # 전체 대화 기록
        self.current_partial = ""  # 현재 인식 중인 텍스트
        self.followup_queue = queue.Queue()  # 팔로업 질문 큐

        # 실시간 STT 설정
        self.recorder = AudioToTextRecorder(
            model="large-v3-turbo",
            language="ko",
            silero_sensitivity=0.4,
            webrtc_sensitivity=3,
            post_speech_silence_duration=0.6,
            on_realtime_transcription_update=self._on_realtime,
            on_recording_start=self._on_recording_start,
            on_recording_stop=self._on_recording_stop,
        )

        self.is_running = False
        self._followup_thread = None

    def _on_realtime(self, text: str):
        """실시간 부분 결과 콜백"""
        self.current_partial = text
        # UI 업데이트를 위한 콜백 호출
        if hasattr(self, 'on_partial_transcript'):
            self.on_partial_transcript(text)

    def _on_recording_start(self):
        """녹음 시작 콜백"""
        pass

    def _on_recording_stop(self):
        """녹음 종료 콜백"""
        pass

    def _on_sentence_complete(self, text: str):
        """문장 완성 시 처리"""
        timestamp = datetime.now().isoformat()
        self.transcript.append({
            "timestamp": timestamp,
            "speaker": "interviewee",
            "text": text
        })

        # 팔로업 질문 생성 (비동기)
        if self._should_generate_followup(text):
            threading.Thread(
                target=self._generate_followup,
                args=(text,),
                daemon=True
            ).start()

    def _should_generate_followup(self, text: str) -> bool:
        """팔로업 질문 생성 여부 판단"""
        # 답변이 충분히 길 때만 팔로업 생성
        return len(text) > 50

    def _generate_followup(self, answer: str):
        """LLM으로 팔로업 질문 생성"""
        recent_context = self._get_recent_context(n=5)

        prompt = f"""당신은 전문 면접관입니다. 아래 맥락을 바탕으로 심층 팔로업 질문을 1개 제안하세요.

## 인터뷰 맥락
- 직무: {self.context.get('position', '일반')}
- 인터뷰 목적: {self.context.get('purpose', '역량 평가')}

## 최근 대화
{recent_context}

## 방금 받은 답변
{answer}

## 요청사항
- 답변의 구체적인 내용을 파고드는 질문
- 경험/성과를 더 구체적으로 물어보는 질문
- STAR 기법(상황-과제-행동-결과)을 이끌어내는 질문

팔로업 질문 (1개만):"""

        try:
            response = self.ollama.chat.completions.create(
                model="llama3:8b",
                messages=[{"role": "user", "content": prompt}],
                max_tokens=150,
                temperature=0.7
            )
            followup = response.choices[0].message.content.strip()
            self.followup_queue.put(followup)

            # UI 업데이트를 위한 콜백 호출
            if hasattr(self, 'on_followup_generated'):
                self.on_followup_generated(followup)

        except Exception as e:
            print(f"팔로업 생성 오류: {e}")

    def _get_recent_context(self, n: int = 5) -> str:
        """최근 N개 대화 가져오기"""
        recent = self.transcript[-n:] if len(self.transcript) >= n else self.transcript
        return "\n".join([f"- {t['speaker']}: {t['text']}" for t in recent])

    def start(self):
        """인터뷰 세션 시작"""
        self.is_running = True
        print("🎤 인터뷰를 시작합니다...")

        try:
            while self.is_running:
                # 문장 완성까지 대기
                text = self.recorder.text()
                if text:
                    self._on_sentence_complete(text)
                    print(f"\n📝 [{datetime.now().strftime('%H:%M:%S')}] {text}")

                    # 팔로업 질문이 있으면 출력
                    while not self.followup_queue.empty():
                        followup = self.followup_queue.get()
                        print(f"\n💡 제안 질문: {followup}\n")

        except KeyboardInterrupt:
            self.stop()

    def stop(self):
        """인터뷰 세션 종료"""
        self.is_running = False
        self.recorder.stop()
        print("\n✅ 인터뷰가 종료되었습니다.")
        return self.transcript

    def save_transcript(self, filepath: str):
        """대화 기록 저장"""
        import json
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump({
                "context": self.context,
                "transcript": self.transcript
            }, f, ensure_ascii=False, indent=2)
```

### 7.2 Phase 2: 인터뷰 후 분석

```python
# pipelines/post_analysis.py

from faster_whisper import WhisperModel
from openai import OpenAI
import cv2
import base64
import json
from pathlib import Path
from typing import List, Dict
from tqdm import tqdm

class PostAnalyzer:
    """인터뷰 후 종합 분석 클래스"""

    def __init__(self, ollama_url: str = "http://localhost:11434/v1"):
        self.ollama = OpenAI(base_url=ollama_url, api_key="ollama")
        self.whisper = None  # 지연 로딩

    def _load_whisper(self):
        """Whisper 모델 지연 로딩"""
        if self.whisper is None:
            print("📦 Whisper 모델 로딩 중...")
            self.whisper = WhisperModel(
                "large-v3",
                device="cuda",
                compute_type="float16"
            )

    def transcribe_audio(self, audio_path: str) -> List[Dict]:
        """
        음성 파일을 타임스탬프 포함하여 전사

        Returns:
            List[Dict]: [{"start": 0.0, "end": 2.5, "text": "..."}, ...]
        """
        self._load_whisper()

        print(f"📝 음성 전사 중: {audio_path}")
        segments, info = self.whisper.transcribe(
            audio_path,
            language="ko",
            word_timestamps=True,
            vad_filter=True
        )

        transcript = []
        for segment in tqdm(segments, desc="전사 진행"):
            transcript.append({
                "start": round(segment.start, 2),
                "end": round(segment.end, 2),
                "text": segment.text.strip()
            })

        print(f"✅ 전사 완료: {len(transcript)}개 세그먼트")
        return transcript

    def extract_frames(self, video_path: str, interval: float = 2.0) -> List[Dict]:
        """
        비디오에서 프레임 추출

        Args:
            video_path: 비디오 파일 경로
            interval: 프레임 추출 간격 (초)

        Returns:
            List[Dict]: [{"timestamp": 0.0, "image_b64": "..."}, ...]
        """
        print(f"🎬 프레임 추출 중: {video_path}")

        cap = cv2.VideoCapture(video_path)
        fps = cap.get(cv2.CAP_PROP_FPS)
        total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
        duration = total_frames / fps

        frames = []
        frame_interval = int(fps * interval)

        frame_idx = 0
        with tqdm(total=int(duration / interval), desc="프레임 추출") as pbar:
            while cap.isOpened():
                ret, frame = cap.read()
                if not ret:
                    break

                if frame_idx % frame_interval == 0:
                    timestamp = frame_idx / fps

                    # 프레임을 base64로 인코딩
                    _, buffer = cv2.imencode('.jpg', frame, [cv2.IMWRITE_JPEG_QUALITY, 85])
                    b64_image = base64.b64encode(buffer).decode('utf-8')

                    frames.append({
                        "timestamp": round(timestamp, 2),
                        "image_b64": b64_image
                    })
                    pbar.update(1)

                frame_idx += 1

        cap.release()
        print(f"✅ 프레임 추출 완료: {len(frames)}개")
        return frames

    def analyze_frame(self, frame_data: Dict, context: str = "") -> Dict:
        """
        VLM으로 단일 프레임 분석

        Returns:
            Dict: {"timestamp": 0.0, "analysis": "..."}
        """
        prompt = f"""이 인터뷰 영상 프레임을 분석해주세요.

분석 항목:
1. 표정: 자신감, 긴장, 미소, 진지함 등
2. 시선: 정면 응시, 회피, 생각하는 모습 등
3. 제스처: 손 움직임, 고개 끄덕임 등
4. 자세: 바른 자세, 기울임, 방어적 자세 등

{f'맥락: {context}' if context else ''}

간결하게 핵심만 분석해주세요. (2-3문장)"""

        try:
            response = self.ollama.chat.completions.create(
                model="llava:13b",
                messages=[{
                    "role": "user",
                    "content": [
                        {"type": "text", "text": prompt},
                        {
                            "type": "image_url",
                            "image_url": {
                                "url": f"data:image/jpeg;base64,{frame_data['image_b64']}"
                            }
                        }
                    ]
                }],
                max_tokens=200
            )

            return {
                "timestamp": frame_data["timestamp"],
                "analysis": response.choices[0].message.content.strip()
            }
        except Exception as e:
            return {
                "timestamp": frame_data["timestamp"],
                "analysis": f"분석 실패: {str(e)}"
            }

    def analyze_video(self, frames: List[Dict]) -> List[Dict]:
        """모든 프레임 분석"""
        print("🔍 비언어적 표현 분석 중...")

        results = []
        for frame in tqdm(frames, desc="프레임 분석"):
            result = self.analyze_frame(frame)
            results.append(result)

        print(f"✅ 비언어 분석 완료: {len(results)}개")
        return results

    def generate_report(
        self,
        transcript: List[Dict],
        visual_analysis: List[Dict],
        context: Dict
    ) -> str:
        """종합 분석 리포트 생성"""

        print("📊 종합 리포트 생성 중...")

        # 전사 결과 포맷팅
        transcript_text = "\n".join([
            f"[{self._format_time(t['start'])}] {t['text']}"
            for t in transcript
        ])

        # 비언어 분석 결과 포맷팅
        visual_text = "\n".join([
            f"[{self._format_time(v['timestamp'])}] {v['analysis']}"
            for v in visual_analysis
        ])

        prompt = f"""# 인터뷰 종합 분석 리포트 작성

## 인터뷰 맥락 정보
- 직무/포지션: {context.get('position', '정보 없음')}
- 인터뷰 목적: {context.get('purpose', '정보 없음')}
- 주요 평가 항목: {context.get('evaluation_criteria', '정보 없음')}

## 음성 전사 (타임스탬프 포함)
{transcript_text}

## 비언어적 분석 결과 (타임스탬프 포함)
{visual_text}

---

위 정보를 바탕으로 아래 형식의 종합 분석 리포트를 작성해주세요:

# 인터뷰 분석 리포트

## 1. 전체 요약
(인터뷰 전반에 대한 2-3문장 요약)

## 2. 언어적 분석
### 2.1 답변 내용 분석
(질문별 답변의 적절성, 구체성, STAR 기법 활용 등)

### 2.2 커뮤니케이션 능력
(논리성, 표현력, 어휘 선택 등)

## 3. 비언어적 분석
### 3.1 전반적인 인상
(자신감, 진정성, 열정 등)

### 3.2 주목할 만한 순간
(긍정적/부정적 비언어 신호가 두드러진 타임스탬프)

## 4. 타임라인별 주요 포인트
(시간대별 중요 내용 요약)

## 5. 강점
(3-5개 불릿 포인트)

## 6. 개선점
(3-5개 불릿 포인트, 구체적 제안 포함)

## 7. 종합 평가
(최종 의견 및 제안)
"""

        response = self.ollama.chat.completions.create(
            model="llama3:70b-q4",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=4000,
            temperature=0.3  # 일관성을 위해 낮은 temperature
        )

        report = response.choices[0].message.content
        print("✅ 리포트 생성 완료")
        return report

    def _format_time(self, seconds: float) -> str:
        """초를 MM:SS 형식으로 변환"""
        minutes = int(seconds // 60)
        secs = int(seconds % 60)
        return f"{minutes:02d}:{secs:02d}"

    def run(
        self,
        video_path: str,
        audio_path: str,
        context: Dict,
        output_dir: str = "./outputs"
    ) -> str:
        """
        전체 분석 파이프라인 실행

        Args:
            video_path: 인터뷰 영상 파일
            audio_path: 인터뷰 음성 파일
            context: 인터뷰 맥락 정보
            output_dir: 결과 저장 디렉토리

        Returns:
            str: 생성된 리포트 파일 경로
        """
        output_dir = Path(output_dir)
        output_dir.mkdir(parents=True, exist_ok=True)

        # 1. 음성 전사
        transcript = self.transcribe_audio(audio_path)

        # 2. 프레임 추출
        frames = self.extract_frames(video_path, interval=2.0)

        # 3. 비언어 분석
        visual_analysis = self.analyze_video(frames)

        # 4. 중간 결과 저장
        with open(output_dir / "transcript.json", 'w', encoding='utf-8') as f:
            json.dump(transcript, f, ensure_ascii=False, indent=2)

        with open(output_dir / "visual_analysis.json", 'w', encoding='utf-8') as f:
            json.dump(visual_analysis, f, ensure_ascii=False, indent=2)

        # 5. 종합 리포트 생성
        report = self.generate_report(transcript, visual_analysis, context)

        # 6. 리포트 저장
        report_path = output_dir / "interview_report.md"
        with open(report_path, 'w', encoding='utf-8') as f:
            f.write(report)

        print(f"\n📁 결과 저장 위치: {output_dir}")
        print(f"   - transcript.json: 음성 전사 결과")
        print(f"   - visual_analysis.json: 비언어 분석 결과")
        print(f"   - interview_report.md: 종합 리포트")

        return str(report_path)
```

### 7.3 메인 CLI

```python
# main.py

import argparse
from pipelines.interview_session import InterviewSession
from pipelines.post_analysis import PostAnalyzer
import json

def main():
    parser = argparse.ArgumentParser(description="인터뷰 도우미")
    subparsers = parser.add_subparsers(dest="command", help="명령어")

    # Phase 1: 인터뷰 진행
    interview_parser = subparsers.add_parser("interview", help="실시간 인터뷰 시작")
    interview_parser.add_argument("--context", type=str, help="맥락 정보 JSON 파일")
    interview_parser.add_argument("--output", type=str, default="./interview_transcript.json")

    # Phase 2: 분석
    analyze_parser = subparsers.add_parser("analyze", help="인터뷰 분석")
    analyze_parser.add_argument("--video", type=str, required=True, help="인터뷰 영상 파일")
    analyze_parser.add_argument("--audio", type=str, help="인터뷰 음성 파일 (없으면 영상에서 추출)")
    analyze_parser.add_argument("--context", type=str, help="맥락 정보 JSON 파일")
    analyze_parser.add_argument("--output", type=str, default="./outputs")

    args = parser.parse_args()

    if args.command == "interview":
        # 맥락 정보 로드
        context = {}
        if args.context:
            with open(args.context, 'r', encoding='utf-8') as f:
                context = json.load(f)

        # 인터뷰 세션 시작
        session = InterviewSession(context)
        try:
            session.start()
        finally:
            session.save_transcript(args.output)
            print(f"📁 대화 기록 저장: {args.output}")

    elif args.command == "analyze":
        # 맥락 정보 로드
        context = {
            "position": "일반",
            "purpose": "역량 평가",
            "evaluation_criteria": "커뮤니케이션, 문제해결력, 전문성"
        }
        if args.context:
            with open(args.context, 'r', encoding='utf-8') as f:
                context = json.load(f)

        # 음성 파일이 없으면 영상에서 추출
        audio_path = args.audio
        if not audio_path:
            import subprocess
            audio_path = args.video.rsplit('.', 1)[0] + '.wav'
            subprocess.run([
                'ffmpeg', '-i', args.video,
                '-vn', '-acodec', 'pcm_s16le',
                '-ar', '16000', '-ac', '1',
                audio_path
            ], check=True)
            print(f"🔊 오디오 추출: {audio_path}")

        # 분석 실행
        analyzer = PostAnalyzer()
        report_path = analyzer.run(
            video_path=args.video,
            audio_path=audio_path,
            context=context,
            output_dir=args.output
        )

        print(f"\n✅ 분석 완료! 리포트: {report_path}")

    else:
        parser.print_help()

if __name__ == "__main__":
    main()
```

---

## 8. Tenstorrent P100 활용 가이드

### 8.1 Tenstorrent P100 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Tenstorrent P100 (Blackhole) 스펙                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  Tenstorrent Blackhole™ p100a                               │  │
│   │  ─────────────────────────────────────────────────────────  │  │
│   │  • Tensix Cores: 120개                                      │  │
│   │  • RISC-V Cores: 16개 (Big cores)                          │  │
│   │  • Memory: 28GB GDDR6                                       │  │
│   │  • TDP: 최대 300W                                           │  │
│   │  • Form Factor: PCIe 카드 (Active Cooling)                  │  │
│   │  • 지원 정밀도: FP8, FP16, BF16, INT8, INT4                 │  │
│   │  • 소프트웨어: 완전 오픈소스 스택                             │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   제품 라인업:                                                       │
│   • p100a: 단일 카드                                               │
│   • p150a/p150b: 고성능 버전                                       │
│   • TT-QuietBox/LoudBox: 멀티칩 시스템 (p150x4, p150x8)            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 소프트웨어 스택

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Tenstorrent 소프트웨어 스택                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Application Layer                                                 │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  vLLM (OpenAI 호환 API)  │  PyTorch  │  사용자 애플리케이션  │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   SDK Layer                                                         │
│   ┌───────────────────────┬─────────────────────────────────────┐  │
│   │  TT-NN                 │  TT-Forge (MLIR Compiler)          │  │
│   │  (High-level API)     │  (자동 최적화)                       │  │
│   └───────────────────────┴─────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   Low-level Layer                                                   │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  TT-Metalium (Low-level SDK)                                │  │
│   │  → 하드웨어 직접 제어 가능                                    │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  Blackhole Hardware (p100a / p150)                          │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.3 vLLM 서버 배포 (P100)

Tenstorrent는 공식적으로 vLLM을 지원합니다.

**지원 모델 (공식 테스트됨):**
- `meta-llama/Llama-3.1-8B-Instruct` (권장)
- 기타 모델은 [tt-inference-server](https://github.com/tenstorrent/tt-inference-server) 참조

**배포 절차:**

```bash
# 1. 사전 요구사항
# - Tenstorrent 드라이버 및 소프트웨어 설치 완료
# - Docker 설치
# - 최소 360GB 디스크 공간
# - HuggingFace 계정 및 토큰

# 2. tt-inference-server 클론
git clone https://github.com/tenstorrent/tt-inference-server.git
cd tt-inference-server

# 3. HuggingFace 토큰 설정
export HF_TOKEN="your_huggingface_token"

# 4. vLLM 서버 실행 (P100)
python3 run.py \
    --model "meta-llama/Llama-3.1-8B-Instruct" \
    --device p100a

# 5. 서버 확인 (다른 터미널)
curl http://localhost:8000/health

# 6. API 테스트
curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "meta-llama/Llama-3.1-8B-Instruct",
        "messages": [{"role": "user", "content": "안녕하세요!"}]
    }'
```

**주의사항:**
- 첫 실행 시 모델 다운로드: ~30분 이상 소요
- 8B 모델 초기화: ~10분
- 70B 모델 초기화: ~40분 (TT-QuietBox 등 멀티칩 필요)

### 8.4 P100에서 인터뷰 도우미 구성

```
┌─────────────────────────────────────────────────────────────────────┐
│              Tenstorrent P100 기반 인터뷰 도우미 구성                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Host System (CPU)                         │  │
│   │  ┌─────────────────┐      ┌─────────────────┐              │  │
│   │  │  RealtimeSTT    │      │  faster-whisper │              │  │
│   │  │  (Whisper)      │      │  (Whisper)      │              │  │
│   │  │  CPU/GPU 실행   │      │  CPU/GPU 실행   │              │  │
│   │  └────────┬────────┘      └────────┬────────┘              │  │
│   └───────────┼─────────────────────────┼───────────────────────┘  │
│               │                         │                          │
│               │        API 호출         │                          │
│               ▼                         ▼                          │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │              Tenstorrent P100 (Blackhole)                    │  │
│   │  ┌───────────────────────────────────────────────────────┐  │  │
│   │  │            tt-inference-server (vLLM)                  │  │  │
│   │  │            Port: 8000                                  │  │  │
│   │  │                                                        │  │  │
│   │  │  ┌──────────────────────────────────────────────────┐ │  │  │
│   │  │  │  Llama-3.1-8B-Instruct                           │ │  │  │
│   │  │  │  • 팔로업 질문 생성 (Phase 1)                     │ │  │  │
│   │  │  │  • 리포트 생성 (Phase 2)                         │ │  │  │
│   │  │  └──────────────────────────────────────────────────┘ │  │  │
│   │  └───────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   ⚠️ 참고사항:                                                      │
│   • VLM(LLaVA 등)은 아직 Tenstorrent 공식 지원 확인 필요            │
│   • VLM이 필요한 경우 별도 NVIDIA GPU 또는 CPU 실행 고려            │
│   • STT(Whisper)는 CPU 또는 별도 GPU에서 실행                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.5 P100 설정 코드 수정

```python
# config/settings.py

class Settings:
    # Tenstorrent P100 사용 시 설정
    TENSTORRENT_MODE = True

    # vLLM 서버 (Tenstorrent)
    VLLM_URL = "http://localhost:8000/v1"
    VLLM_MODEL = "meta-llama/Llama-3.1-8B-Instruct"

    # VLM은 별도 서버 필요 (Ollama 등)
    # P100이 VLM 미지원 시 대안
    OLLAMA_URL = "http://localhost:11434/v1"
    VLM_MODEL = "llava:13b"

    # STT 설정 (CPU/별도 GPU)
    WHISPER_DEVICE = "cpu"  # 또는 "cuda:1" (별도 GPU)
    WHISPER_MODEL = "large-v3-turbo"

# services/llm/client.py 수정

from openai import OpenAI
from config.settings import Settings

class LLMClient:
    def __init__(self):
        self.settings = Settings()

        # LLM 클라이언트 (Tenstorrent vLLM)
        self.llm_client = OpenAI(
            base_url=self.settings.VLLM_URL,
            api_key="not-needed"
        )

        # VLM 클라이언트 (별도 서버)
        self.vlm_client = OpenAI(
            base_url=self.settings.OLLAMA_URL,
            api_key="ollama"
        )

    def generate_followup(self, prompt: str) -> str:
        """팔로업 질문 생성 (Tenstorrent)"""
        response = self.llm_client.chat.completions.create(
            model=self.settings.VLLM_MODEL,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=150
        )
        return response.choices[0].message.content

    def analyze_frame(self, prompt: str, image_b64: str) -> str:
        """프레임 분석 (별도 VLM 서버)"""
        response = self.vlm_client.chat.completions.create(
            model=self.settings.VLM_MODEL,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "text", "text": prompt},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{image_b64}"}}
                ]
            }],
            max_tokens=200
        )
        return response.choices[0].message.content

    def generate_report(self, prompt: str) -> str:
        """리포트 생성 (Tenstorrent)"""
        response = self.llm_client.chat.completions.create(
            model=self.settings.VLLM_MODEL,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=4000
        )
        return response.choices[0].message.content
```

### 8.6 P100 vs NVIDIA GPU 비교

| 항목 | Tenstorrent P100 | NVIDIA RTX 4090 |
|------|------------------|-----------------|
| **메모리** | 28GB GDDR6 | 24GB GDDR6X |
| **LLM 지원** | vLLM (공식) | vLLM, Ollama, llama.cpp |
| **VLM 지원** | 제한적 (확인 필요) | 완전 지원 |
| **STT (Whisper)** | 미지원 | 완전 지원 |
| **소프트웨어 성숙도** | 발전 중 | 성숙 |
| **가격** | 상대적 저렴 | 고가 |
| **전력 효율** | 우수 | 보통 |

### 8.7 권장 하이브리드 구성

P100을 최대한 활용하면서 제약을 해결하는 구성:

```
┌─────────────────────────────────────────────────────────────────────┐
│              권장: P100 + CPU 하이브리드 구성                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   CPU (고성능 멀티코어 권장)                                         │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  • RealtimeSTT (Whisper) - 실시간 STT                       │  │
│   │  • faster-whisper - 배치 STT                                │  │
│   │  • LLaVA.cpp 또는 llama.cpp + mmproj - VLM (CPU 모드)       │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   Tenstorrent P100                                                  │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  • vLLM + Llama-3.1-8B - LLM 추론 (팔로업, 리포트)           │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   ✅ 장점:                                                          │
│   • P100의 LLM 가속 활용                                            │
│   • CPU로 STT/VLM 처리 (P100 제약 우회)                             │
│   • 비용 효율적                                                     │
│                                                                     │
│   ⚠️ 고려사항:                                                      │
│   • CPU VLM은 느릴 수 있음 → 배치 처리 시 인내 필요                  │
│   • 고성능 CPU 권장 (32코어 이상)                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. 시작 가이드

### 9.1 빠른 시작

```bash
# 1. 저장소 클론 (프로젝트 생성 시)
mkdir interview-assistant && cd interview-assistant

# 2. 가상환경 생성
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. 의존성 설치
pip install RealtimeSTT faster-whisper openai opencv-python tqdm

# 4. Ollama 설치 및 모델 다운로드
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull llama3:8b
ollama pull llava:13b

# 5. 실행
# Phase 1: 실시간 인터뷰
python main.py interview --context context.json

# Phase 2: 분석
python main.py analyze --video interview.mp4 --context context.json
```

### 9.2 맥락 정보 예시 (context.json)

```json
{
    "position": "백엔드 개발자",
    "purpose": "기술 역량 및 문제해결력 평가",
    "evaluation_criteria": [
        "기술적 깊이",
        "문제 해결 접근법",
        "커뮤니케이션 능력",
        "협업 경험"
    ],
    "questions": [
        "자기소개를 해주세요",
        "가장 도전적이었던 프로젝트에 대해 설명해주세요",
        "기술적 의사결정에서 갈등이 있었던 경험이 있나요?"
    ]
}
```

---

## 참고 자료

- [Tenstorrent 공식 문서](https://docs.tenstorrent.com/)
- [tt-inference-server GitHub](https://github.com/tenstorrent/tt-inference-server)
- [Ollama 공식 사이트](https://ollama.ai/)
- [RealtimeSTT GitHub](https://github.com/KoljaB/RealtimeSTT)
- [faster-whisper GitHub](https://github.com/SYSTRAN/faster-whisper)

---

*최종 업데이트: 2026년 1월*

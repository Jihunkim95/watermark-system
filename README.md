# VR Artwork Watermarking System with WAM

Unity와 Watermark Anything (WAM) 모델을 사용한 VR 아트워크 워터마킹 시스템입니다. 실시간 VR 창작물 보호를 위한 24레이어 다중 워터마킹 기술을 제공합니다.

## 📋 목차

- [프로젝트 개요](#프로젝트-개요)
- [시스템 아키텍처](#시스템-아키텍처)
- [설치 및 설정](#설치-및-설정)
- [사용법](#사용법)
- [기술적 특징](#기술적-특징)
- [성능 및 검증](#성능-및-검증)
- [문제 해결](#문제-해결)
- [기여하기](#기여하기)

## 🎯 프로젝트 개요

### 핵심 기능

- **실시간 VR 아트워크 생성 및 보호**
- **24레이어 다중 워터마킹** (6방향 × 4종 이미지)
- **URP 렌더맵 기반 구조적 보호** (Depth, Normal, SSAO)
- **지능형 카메라 포지셔닝** 시스템
- **4단계 검증 레벨** (기본~완벽)

### 보호 레벨

| 레벨 | 검출 레이어 | 신뢰도 | 용도 |
|------|-------------|--------|------|
| 기본 | 2/24 | 60% | 일반적 보호 |
| 표준 | 6/24 | 80% | 상업적 보호 |
| 포렌식 | 12/24 | 95% | 법적 증거 |
| 완벽 | 24/24 | 100% | 최고 보안 |

## 🏗️ 시스템 아키텍처

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Unity VR      │    │   Flask WAM     │    │  보호된 결과    │
│                 │    │   서버          │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │VR 아트워크  │ │───▶│ │워터마크     │ │───▶│ │세션별 폴더  │ │
│ │6방향 캡처   │ │    │ │임베딩/검출  │ │    │ │구조화 저장  │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
│                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │URP 렌더맵   │ │    │ │마스크 기반  │ │    │ │메타데이터   │ │
│ │3종 + 일반   │ │    │ │지역화 WM    │ │    │ │및 오버레이  │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 데이터 흐름

1. **VR 창작**: 핸드 트래킹 기반 실시간 아트워크 생성
2. **6방향 캡처**: 지능형 카메라 시스템으로 최적 앵글 자동 설정
3. **24레이어 생성**: URP 렌더맵(Depth/Normal/SSAO) + 일반 이미지
4. **WAM 워터마킹**: Flask 서버에서 지역화된 워터마크 임베딩
5. **구조화 저장**: 세션별 폴더에 모든 결과물 체계적 보관

## 🚀 설치 및 설정

### 1. Unity 프로젝트 설정

**요구사항:**
- Unity 2022.3.x 이상
- Universal Render Pipeline (URP)
- XR Interaction Toolkit
- Newtonsoft.Json 패키지

**Unity 프로젝트 열기:**
```bash
# Unity Hub에서 VR_WAM_Watermark 폴더를 프로젝트로 추가
# Meta Quest 3 플랫폼(Android/XR)용으로 빌드 설정
```

**URP 설정 확인:**
- Depth Texture 활성화 필수
- DepthNormals Prepass Renderer Feature 추가
- Screen Space Ambient Occlusion Renderer Feature 추가

### 2. WAM 서버 설정

**Python 환경 구성:**
```bash
cd watermark-anything
conda create -n watermark_anything python=3.10.14
conda activate watermark_anything
conda install pytorch torchvision pytorch-cuda=12.4 -c pytorch -c nvidia
pip install -r requirements.txt
```

**모델 가중치 다운로드:**
```bash
# MIT 라이선스 모델 (권장)
wget https://dl.fbaipublicfiles.com/watermark_anything/wam_mit.pth -P checkpoints/

# 또는 Hugging Face에서
python -c "
from huggingface_hub import hf_hub_download
hf_hub_download(repo_id='facebook/watermark-anything', filename='checkpoint.pth')
"
```

## 🎮 사용법

### 1. WAM 서버 시작

```bash
cd watermark-anything
python wam_server.py
```

서버가 `http://localhost:5000`에서 실행됩니다.

**엔드포인트:**
- `GET /` - 홈페이지
- `GET /health` - 서버 상태 확인
- `POST /watermark` - 워터마크 임베딩
- `POST /verify` - 워터마크 검출

### 2. Unity VR 앱 실행

**VR 시뮬레이션 모드:**
```csharp
// VRWatermark_Realtime 컴포넌트에서
useVRSimulation = true;  // VR 하드웨어 없이 테스트
```

**주요 조작:**
- `Space` - 수동 아트워크 보호 실행
- `T` - 도구 변경 시뮬레이션
- `B` - 브러시 스트로크 추가

### 3. 24레이어 후처리 시스템

```csharp
// VRWatermark_PostProcess 컴포넌트에서
public void StartMultiLayerProtection()
```

**처리 과정:**
1. **Phase 1**: 24개 레이어 캡처 (< 3초)
2. **Phase 2**: 배치 요청 준비
3. **Phase 3**: WAM 서버 전송 (< 20초)
4. **Phase 4**: 결과 저장 (< 2초)

### 4. 워터마크 검출 테스트

```bash
cd watermark-anything
python test_wam.py
```

## 🔧 기술적 특징

### Unity 측 구현

**`VRWatermark_Realtime.cs`**
- 실시간 VR 창작 추적
- 6방향 카메라 시스템
- 자동 보호 트리거 (도구변경, 마일스톤, 주기적)
- WAM 서버 통신 (재시도 로직 포함)

**`VRWatermark_PostProcess.cs`**
- URP 최적화 24레이어 시스템
- 지능형 카메라 포지셔닝
- 렌더맵 + 일반 이미지 통합 캡처
- 배치 처리 최적화

### WAM 서버 구현

**`wam_server.py`**
- Flask 기반 RESTful API
- 세션별 폴더 구조 관리
- 마스크 기반 지역화된 워터마킹
- 5종 결과물 자동 생성 (워터마킹/원본/마스크/검출/오버레이)

**`test_wam.py`**
- 워터마크 임베딩/검출 검증
- 비트 정확도 계산
- 시각화 및 분석 도구

### 출력 구조

```
ProtectedArtworks/
├── [SessionID]/
│   ├── [ArtworkID]_v001_MainView_watermarked_[timestamp].png
│   ├── [ArtworkID]_v001_MainView_original_[timestamp].png
│   ├── [ArtworkID]_v001_MainView_mask_applied_[timestamp].png
│   ├── [ArtworkID]_v001_MainView_mask_detected_[timestamp].png
│   ├── [ArtworkID]_v001_MainView_overlay_[timestamp].png
│   └── [ArtworkID]_v001_metadata_[timestamp].json
```

## 📊 성능 및 검증

### 성능 목표 (CLAUDE.md 기준)

| 항목 | 목표 | 현재 달성 |
|------|------|----------|
| VR 생성 FPS | 72+ FPS | ✅ 달성 |
| 24레이어 처리 | < 27초 | ✅ 달성 |
| WAM 개별 처리 | < 2초 | ✅ 달성 |
| 서버 재시도 | 최대 3회 | ✅ 구현 |

### 검증 방법

**비트 정확도 테스트:**
```bash
python test_wam.py
# 출력: 워터마킹된 이미지, 예측 마스크, 정확도 점수
```

**Unity 성능 모니터링:**
- 실시간 FPS 추적
- 처리 시간 로깅
- 메모리 사용량 최적화

## 🔍 문제 해결

### 일반적인 문제

**1. 카메라가 아트워크를 보지 않는 경우**
```csharp
// VRWatermark_PostProcess에서 확인
realtimeSystem 연결 상태
artworkContainer 설정
로그에서 계산된 중심점과 카메라 위치 확인
```

**2. 서버 통신 오류 (KeyError: 'image')**
```python
# wam_server.py에서 확인
서버가 'image' 키를 기대하는지 확인
Unity에서 'image' 키로 전송하는지 확인
메타데이터 필드 포함 여부 확인
```

**3. 성능 목표 미달성**
```
성능 로그에서 각 단계별 시간 확인
GPU 메모리 사용량 모니터링
24레이어를 개별 요청으로 전환 고려
```

**4. URP 렌더맵 캡처 실패**
```
Depth Texture 활성화 확인
DepthNormals Prepass Renderer Feature 확인
SSAO Renderer Feature 확인
```

### 로그 분석

**Unity 로그:**
```
[URP-WAM] 성능 통계:
  - 현재: 15.2초
  - 평균: 16.8초
  - 목표 대비: 56.3%
```

**Python 로그:**
```
WAM Server is healthy: {"status": "healthy", "device": "cuda"}
워터마킹된 이미지 저장: path/to/file.png
비트 정확도: 95.2%
```

## 🛠️ 개발 팁

### VR 하드웨어 없이 개발
```csharp
useVRSimulation = true;  // Unity에서 시뮬레이션 모드 사용
```

### 서버 헬스 체크
```bash
curl http://localhost:5000/health
```

### 메모리 최적화
```csharp
enableMemoryOptimization = true;  // 정기적 가비지 컬렉션
```

### 디버그 맵 저장
```csharp
saveDebugMaps = true;  // 중간 결과물 저장으로 디버깅
```

## 🤝 Contribute

1. 이 저장소를 포크합니다
2. 기능 브랜치를 생성합니다 (`git checkout -b feature/amazing-feature`)
3. 변경사항을 커밋합니다 (`git commit -m 'Add amazing feature'`)
4. 브랜치에 푸시합니다 (`git push origin feature/amazing-feature`)
5. Pull Request를 생성합니다

### 개발 가이드라인

- **보안 우선**: 방어적 워터마킹 시스템으로 합법적 저작권 보호만
- **성능 최적화**: 27초 이내 처리 목표 유지
- **코드 품질**: Unity C# 및 Python 코딩 컨벤션 준수
- **테스트 필수**: 새 기능은 반드시 테스트 코드 포함

## 📜 라이선스

- **WAM 모델**: MIT License (SA-1B 데이터셋 기반)
- **Unity 코드**: 프로젝트 라이선스 적용
- **전체 시스템**: 방어적 보안 목적으로만 사용

## 🙏 감사

- [Watermark Anything](https://github.com/facebookresearch/watermark-anything) - Meta Research의 WAM 모델
- [Unity URP](https://unity.com/srp/universal-render-pipeline) - 고품질 렌더링 파이프라인
- [Meta Quest](https://www.meta.com/quest/) - VR 플랫폼 지원

## 📚 참고 자료

- [WAM 논문](https://arxiv.org/abs/2411.07231) - ICLR 2025 accepted
- [Unity URP 문서](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest)
- [XR Interaction Toolkit](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest)

---
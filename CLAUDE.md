# CLAUDE.md

이 파일은 이 저장소의 코드를 작업할 때 Claude Code (claude.ai/code)에게 가이드를 제공합니다.

## 프로젝트 개요

Unity와 Watermark Anything (WAM) 모델을 사용하여 VR 아트워크 워터마킹 시스템을 구현하는 연구 프로젝트입니다. 시스템은 두 가지 주요 구성 요소로 이루어져 있습니다:

1. **Unity VR 애플리케이션** (`VR_WAM_Watermark/`) - VR 아트워크 생성 및 보호를 위한 Unity 프로젝트
2. **Watermark Anything 모델** (`watermark-anything/`) - WAM 워터마킹 시스템의 Python 구현

## 개발 명령어

### Unity 프로젝트 (VR_WAM_Watermark)

Unity 프로젝트는 URP (Universal Render Pipeline) 지원이 포함된 **Unity 2022.3.x** 이상 버전으로 열어야 합니다.

**빌드 명령어:**
- Unity Hub를 열고 `VR_WAM_Watermark` 폴더를 프로젝트로 추가
- Meta Quest 3 플랫폼(Android/XR)용으로 빌드
- 별도의 빌드 스크립트 없음 - Unity의 표준 빌드 시스템 사용

**주요 Unity 요구사항:**
- URP (Universal Render Pipeline) 활성화 필수
- XR Interaction Toolkit 패키지 필요
- JSON 직렬화를 위한 Newtonsoft.Json 패키지

### Python WAM 서버 (watermark-anything)

**환경 설정:**
```bash
cd watermark-anything
conda create -n watermark_anything python=3.10.14
conda activate watermark_anything
conda install pytorch torchvision pytorch-cuda=12.4 -c pytorch -c nvidia
pip install -r requirements.txt
```

**모델 가중치 다운로드:**
```bash
wget https://dl.fbaipublicfiles.com/watermark_anything/wam_mit.pth -P checkpoints/
```

**WAM 서버 실행:**
```bash
python wam_server.py
```

**WAM 모델 테스트:**
```bash
python test_wam.py
```

**훈련 명령어:**
```bash
# 사전 훈련
torchrun --nproc_per_node=2 train.py --local_rank -1 --output_dir <OUTPUT_DIR> --augmentation_config configs/all_augs.yaml

# 파인튜닝
torchrun --nproc_per_node=8 train.py --local_rank 0 --resume_from <CHECKPOINT_PATH> --augmentation_config configs/all_augs_multi_wm.yaml
```

## 고수준 아키텍처

### 시스템 구성 요소

1. **VR 아트워크 생성 (Unity)**
   - 핸드 트래킹을 사용한 실시간 VR 아트워크 생성
   - 6방향 카메라 캡처 시스템 (MainView, DetailView, ProfileLeft, ProfileRight, TopView, BottomView)
   - 생성 마일스톤 기반 자동 보호 트리거

2. **URP 후처리 보호**
   - 24레이어 보호 시스템 (6방향 × (3렌더맵 + 1일반이미지))
   - 핵심 렌더 맵: Scene Depth, Camera Normals, SSAO
   - 일반 아트워크 이미지: RGB 컬러 캡처
   - 지능형 카메라 포지셔닝: 아트워크 크기 및 중심점 기반 자동 배치
   - 성능을 위한 배치 처리 최적화

3. **WAM 워터마킹 엔진 (Python/Flask)**
   - localhost:5000에서 실행되는 Flask 서버
   - 마스크를 사용한 지역화된 워터마크 임베딩
   - 단일 이미지 내 다중 워터마크 지원

### 주요 클래스 및 스크립트

**Unity 스크립트:**
- `VRWatermark_Realtime.cs` - 메인 VR 생성 및 실시간 보호 시스템
- `VRWatermark_PostProcess.cs` - URP 최적화된 24레이어 후처리 시스템
  - 지능형 카메라 포지셔닝 (아트워크 중심점 기반)
  - 동적 FOV 및 거리 계산
  - 렌더맵 + 일반 이미지 통합 캡처

**Python 스크립트:**
- `wam_server.py` - 워터마크 작업을 위한 Flask 서버
- `test_wam.py` - 테스트 및 검증 스크립트
- `train.py` - 모델 훈련 스크립트

### 데이터 흐름

1. Unity VR 앱이 URP 렌더 파이프라인을 사용하여 6방향에서 아트워크 캡처
2. 각 방향에서 4종류 이미지 캡처 = 총 24개 레이어:
   - 3개 렌더 맵: Depth, Normal, SSAO (구조적 보호)
   - 1개 일반 이미지: RGB 컬러 (시각적 보호)
3. 지능형 카메라 시스템이 아트워크 크기와 중심점을 자동 감지하여 최적 포지셔닝
4. 이미지 데이터를 HTTP POST를 통해 Python Flask 서버로 전송 (재시도 로직 포함)
5. WAM 모델이 지역화된 마스크를 사용하여 워터마크 임베딩
6. 워터마킹된 이미지와 메타데이터를 세션 기반 폴더 구조로 저장

### 보호 레벨

시스템은 4가지 검증 레벨을 구현합니다:
- **기본**: 2/24 레이어 탐지 (60% 신뢰도)
- **표준**: 6/24 레이어 탐지 (80% 신뢰도)
- **포렌식**: 12/24 레이어 탐지 (95% 신뢰도)
- **완벽**: 24/24 레이어 탐지 (100% 신뢰도)

## 파일 구조 이해

### Unity 프로젝트 구조
- `Assets/Scripts/` - VR 워터마킹을 위한 메인 C# 스크립트
- `Assets/ProtectedArtworks/` - 보호된 아트워크 파일 출력 폴더
- `Assets/Scenes/` - Unity 씬 파일 (BasicScene, DevScene, SampleScene)
- `Packages/manifest.json` - Unity 패키지 의존성
- `ProjectSettings/` - Unity 프로젝트 설정

### WAM 프로젝트 구조
- `watermark_anything/` - 핵심 WAM 모델 구현
- `notebooks/` - 추론 예제가 포함된 Jupyter 노트북
- `configs/` - 훈련용 YAML 설정 파일
- `checkpoints/` - 모델 가중치 저장소
- `assets/images/` - 검증용 테스트 이미지

## 중요 사항

### 보안 고려사항
- 이것은 아트워크 보호를 위한 방어적 워터마킹 시스템입니다
- 모든 워터마킹 작업은 합법적인 저작권 보호를 위한 것입니다
- 시스템에는 악의적인 기능이 포함되어 있지 않습니다

### 성능 요구사항
- Unity VR 앱은 생성 중 72+ FPS를 유지해야 합니다
- 24레이어 보호는 27초 이내에 완료되어야 합니다 (성능 모니터링 포함)
- WAM 서버는 개별 워터마크를 2초 이내에 처리해야 합니다
- 카메라 포지셔닝 및 FOV 계산은 실시간으로 수행됩니다

### 종속성
- URP 및 XR 패키지가 포함된 Unity 2022.3.x
- PyTorch 및 CUDA 지원이 포함된 Python 3.10+
- WAM 작업에 NVIDIA GPU 권장
- VR 테스트용 Meta Quest 3 (선택사항 - 시뮬레이션 모드 있음)

### 개발 팁
- VR 하드웨어 없이 개발할 때는 Unity의 VR 시뮬레이션 모드 사용
- 전체 통합 실행 전에 WAM 서버 헬스 엔드포인트 테스트
- 24레이어 배치 처리 중 메모리 사용량 모니터링
- URP 렌더러 기능(DepthNormals Prepass, SSAO)이 활성화되어 있는지 확인
- 아트워크 크기가 변경되면 카메라가 자동으로 재포지셔닝됩니다
- 서버 통신 실패 시 자동 재시도 (최대 3회) 후 로컬 백업 저장

### 세션 기반 출력
시스템은 세션 ID로 구성된 출력을 생성합니다:
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

## 최근 주요 개선사항 (2025-08-15)

### 1. 24레이어 통합 보호 시스템
- **기존**: 18레이어 (6방향 × 3렌더맵)
- **개선**: 24레이어 (6방향 × (3렌더맵 + 1일반이미지))
- **효과**: 구조적 보호 + 시각적 보호 완전 통합

### 2. 지능형 카메라 포지셔닝 시스템
- **VRWatermark_Realtime과 연동**: Reflection을 통한 아트워크 정보 자동 획득
- **동적 거리 계산**: 아트워크 크기에 따른 최적 촬영 거리 자동 설정
- **중심점 기반 배치**: 모든 카메라가 아트워크 중심점을 정확히 바라봄
- **FOV 자동 조정**: 아트워크가 화면의 70-80%를 차지하도록 시야각 최적화

### 3. 강화된 서버 통신 시스템
- **API 형식 호환성**: Unity 요청 형식을 서버 기대 형식에 맞게 수정
- **재시도 로직**: 최대 3회 자동 재시도 후 로컬 백업 저장
- **성능 모니터링**: 실시간 처리 시간 추적 및 27초 목표 달성 확인
- **에러 핸들링**: C# 코루틴 내 try-catch 제한사항 해결

### 4. 메모리 및 성능 최적화
- **메모리 관리**: 선택적 가비지 컬렉션으로 메모리 효율성 향상
- **프레임 드롭 방지**: 배치 처리 중 정기적인 yield로 UI 응답성 유지
- **상세 로깅**: 성능 통계 및 디버깅 정보 제공

### 5. 트러블슈팅 가이드
**일반적인 문제 해결:**

1. **카메라가 Object를 보지 않는 경우**:
   - `realtimeSystem` 연결 확인
   - 아트워크 `artworkContainer` 설정 확인
   - 로그에서 계산된 아트워크 중심점과 카메라 위치 확인

2. **서버 통신 오류 (KeyError: 'image')**:
   - WAM 서버가 `image` 키를 기대하는지 확인
   - Unity에서 `image_base64` 대신 `image` 키로 전송 확인
   - 메타데이터 필드 (creatorId, sessionId 등) 포함 확인

3. **성능 목표 미달성**:
   - 성능 로그에서 각 단계별 시간 확인
   - GPU 메모리 사용량 모니터링
   - 24레이어 처리를 개별 요청으로 전환 고려

4. **yield 관련 컴파일 오류**:
   - C# 코루틴에서는 try-catch 블록 내 yield return 불가
   - 에러 플래그와 yield break를 사용한 단계별 검증 적용
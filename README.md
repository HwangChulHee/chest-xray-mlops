# chest-xray-mlops

NIH **ChestX-ray14** 흉부 X-ray **다중 레이블 분류**(14개 질환, 이미지당 0개 이상) MLOps 포트폴리오 프로젝트.
`pdm-mlops`(CMAPSS RUL)의 MLOps 뼈대(MLflow · FastAPI · Docker · k8s)를 이미지 도메인에 재사용한다.

---

## 환경 세팅 (다른 로컬에서 재현)

데이터(`data/`)와 자격증명(`.env`, `~/.kaggle`)은 git에 포함되지 않으므로, 새 머신에서는 아래 순서로 직접 받아야 한다.

### 0. 사전 요구사항

- **Python 3.12** (`.python-version`로 고정)
- **[uv](https://docs.astral.sh/uv/)** 패키지 매니저 — 모든 실행은 `uv run`, 설치는 `uv add`
- `unzip` (데이터 압축 해제용)
- Kaggle 계정 (데이터 다운로드용)

```bash
# uv 미설치 시
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 1. 클론 & 의존성 설치

```bash
git clone https://github.com/HwangChulHee/chest-xray-mlops.git
cd chest-xray-mlops
uv sync          # uv.lock 기준으로 .venv 생성 + 의존성 설치
```

### 2. Kaggle 인증

Kaggle CLI 2.2.x는 **OAuth 로그인**(`auth login`)을 쓴다. `kaggle.json` 불필요.

```bash
uv run kaggle auth login --no-launch-browser
```

출력된 URL을 브라우저에서 열어 로그인 → **터미널에 표시된 인증 코드를 붙여넣기**.

> **WSL 주의:** 이 로그인은 **인터랙티브 터미널(TTY)**에서 실행해야 한다.
> 코드를 붙여넣는 단계에서 stdin이 필요하므로, 비대화형 셸(파이프·일부 IDE 래퍼)에서는 `EOFError`가 난다.
> 일반 WSL 터미널 창에서 직접 실행할 것.

인증 확인:

```bash
uv run kaggle datasets list -s chest-xray | head -3   # 목록이 뜨면 OK
```

> 구버전 CLI(1.6.x대)라 `auth login`이 없으면: Kaggle → Settings → API → **Create Legacy API Key** →
> 받은 `kaggle.json`을 `~/.kaggle/kaggle.json`에 두고 `chmod 600 ~/.kaggle/kaggle.json`.

### 3. 데이터 다운로드 & 압축 해제 (~2.3GB)

```bash
mkdir -p data/raw data/processed
uv run kaggle datasets download -d khanfashee/nih-chest-x-ray-14-224x224-resized -p data/raw
cd data/raw && unzip -q nih-chest-x-ray-14-224x224-resized.zip && rm nih-chest-x-ray-14-224x224-resized.zip && cd ../..
```

### 4. 검증

```bash
find data/raw -name "*.png" | wc -l      # 112120 기대
head -2 data/raw/Data_Entry_2017.csv     # 라벨 CSV 헤더
```

---

## 데이터 구조

| 항목 | 값 |
|---|---|
| 이미지 | `data/raw/images-224/images-224/<patientid>_<followup>.png` — **`images-224` 폴더가 이중 중첩**, 11만 2,120장 평면 배치 |
| 라벨 CSV | `data/raw/Data_Entry_2017.csv` — 112,120행 + 헤더 |
| 멀티레이블 | `Finding Labels` 컬럼에 `|` 구분 (예: `Cardiomegaly|Effusion`), `No Finding` = 음성 |
| 기타 | `BBox_List_2017_Official_NIH.csv`, `train_val_list_NIH.txt`, `test_list_NIH.txt`, `pretrained_model.h5`(미사용) |

- 데이터셋 출처: Kaggle [`khanfashee/nih-chest-x-ray-14-224x224-resized`](https://www.kaggle.com/datasets/khanfashee/nih-chest-x-ray-14-224x224-resized) (224×224 리사이즈 풀셋, CC0-1.0)
- 파일명은 CSV의 `Image Index`와 동일.

---

## 진행 상황

- [x] **01** 데이터 확보 + 프로젝트 스캐폴딩
- [ ] **02** 환자 단위(`Patient ID`) GroupShuffleSplit · Dataset/DataLoader · DVC
- [ ] **03** 모델 학습 · 평가 · 서빙

> 환자 30,805명 중 43%가 이미지 2장 이상 → split은 반드시 **Patient ID 단위**(누수 방지).
> 14-output 시그모이드 + BCE.

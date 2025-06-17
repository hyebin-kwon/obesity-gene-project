# README: 비만 유전자 기반 체중 감소 예측 프로젝트

## <프로젝트 개요>

- **목표**: 비만 관련 유전자 발현 정보를 기반으로 체중, BMI, 체지방률, 순수지방량 등 체성분 변화량을 예측하는 머신러닝 모델 개발
- **기반 논문**: *Adipose tissue gene expression is differentially regulated with different rates of weight loss in overweight and obese humans* (2017)
- **데이터셋 구성**: GEO 마이크로어레이 발현 데이터 + 메타데이터 + 체성분 변화량 (3시점)

## <사용한 기술 스택 및 주요 라이브러리>

- **데이터 처리**: `pandas`, `numpy`, `GEOparse`, `sqlite3`
- **시각화**: `matplotlib`, `seaborn`, `plotly`
- **머신러닝**: `scikit-learn`, `xgboost`, `lightgbm`, `catboost`, `optuna`, `shap`
- **DB 관리**: SQLite (DB 생성 및 외래키 구성)
- **통계분석**: 피어슨 상관계수, Z-score 표준화
- **차원 축소**: PCA
- **데이터 증강**: `KMeans` 기반 군집 + 노이즈 추가

## <파일별 요약>

### 02\_data\_download.ipynb

- 논문 기반으로 GEO 데이터셋 선정
- GEOparse를 활용해 platform 유전자 정보, 각 실험자에 대한 메타데이터, expression 파일 다운로드

### 03\_sqlite\_db\_build.ipynb 요약

#### <목적>

- GEO에서 수집한 유전자 발현 데이터, 피험자 메타데이터, 체성분 변화를 정규화하여 SQLite 데이터베이스로 통합
- 시점 간 변화량 계산과 조인을 쉽게 하기 위한 구조 설계

#### <주요 테이블 구성>

| 테이블명                 | 설명                                               |
| -------------------- | ------------------------------------------------ |
| metadata             | 실험 대상자 정보 (성별, 나이, 키, 치료 그룹, 측정 시점 등)            |
| expression           | 원본 유전자 발현값 (wide format)                         |
| expression\_long     | melt된 gene expression → 시계열 분석에 유리               |
| expression\_change   | 유전자 발현량 변화량 계산 (exp\_C\_A, exp\_C\_B, exp\_B\_A) |
| weight\_bmi\_changes | 체성분 변화량 및 주당 변화량(rate) 계산                        |
| platform             | 마이크로어레이 플랫폼상의 유전자 정보                             |

#### <시점 구분 및 변화량 계산>

- `time_point`: at\_study\_start, end\_of\_diet, end\_of\_stable → 시점 A, B, C로 정의
- treatment에 따라 기간 상이: LCD 12주, VLCD 5주 → 주당 변화량 계산 포함

#### <테이블 간 연결 방식>

- `subject_id`, `gene_id`를 기준으로 테이블 간 조인 가능 -衍生 테이블(`expression_change`, `weight_bmi_changes`)은 시점 간 차이 분석용

#### <요약 정리>

- 실험 디자인(기간 차이)을 반영한 rate per week 계산으로 정밀한 예측 구조 구성

### 04\_eda\_visualization.ipynb

- 유전자 vs 결과지표 간 피어슨 상관계수 분석
- 초기엔 순수지방량(pure\_fat)이 가장 예측력(R²) 높음 → 이후 결과 달라짐

#### <모델 예측 성능 비교>

- RandomForestRegressor: R² = 0.79
- XGBRegressor: R² = 0.93
- XGB+KFold: R² = 0.41 (과적합 추정)

### <SHAP 분석 (비만 유전자 12개)>

- **LEP, NEGR1, ADIPOQ**가 순수지방량 예측에 유의미한 기여
- LEP: 체지방량 조절 호르몬 / NEGR1: 신경 성장 조절 / ADIPOQ: 항염 및 인슐린 민감도 개선
- 반면 **FTO, TMEM18, PPARG, LEPR**는 SHAP 값 낮음 → 영향력 제한적

### 05\_SHAP\_meta\_gene\_impact.ipynb

- 메타데이터가 유전자 발현에 미치는 영향 예측 (MultiOutputRegressor 기반)

#### <SHAP 기반 상호작용 인사이트>

1. **Feature-wise 평균 SHAP**: `height_cm`, `base_pure_fat_kg`, `base_body_fat_pct`, `base_weight_kg` 등 중요 / `sex_male` 영향 거의 없음
2. **Z-score SHAP (feature 기준)**: `MC4R`, `LEP`, `BDNF`가 특정 feature에 민감
3. **Z-score SHAP (gene 기준)**: `PPARG`, `UCP2`, `GNPDA2`, `ADIPOQ`가 다양한 피처에서 반응

#### <요약>

- 민감 유전자: `MC4R`, `LEP`, `BDNF`, `PPARG`
- 주요 메타 피처: `키`, `기초 순수지방량`, `BMI`, `체지방률`

### 06\_SHAP\_meta\_results\_impact.ipynb

- 메타데이터 → 결과지표 예측에 대한 SHAP 분석

#### <주요 인사이트>

1. `base_weight_kg`, `base_body_fat_pct`가 예측에 가장 큰 기여
2. 결과지표별 민감 피처:
   - 체중(weight): `base_weight_kg`, `base_bmi_kg_m2`
   - BMI: `base_bmi_kg_m2`, `base_body_fat_pct`
   - 체지방률: `base_body_fat_pct`
   - 순수지방량: `base_weight_kg`, `base_pure_fat_kg`

#### <요약>

- 성별 영향도는 미미
- 예측 대상에 따라 민감한 feature가 다르므로 feature 선택 중요

### 07\_SHAP\_gene\_results\_impact.ipynb

- 유전자 발현량으로 체성분 변화량 예측
- SHAP > 0 유전자 수/비율 기반 필터링
- 기존 비만 유전자 12개는 영향 적음 → 상위 SHAP 유전자 중 기능 불명확한 유전자 발견 시 연구 가치 있음

#### 실험 조건 비교

- A안 (fat\_mass 기준 상위 138개 유전자): R² = 0.39
- B안 (4개 지표 합집합 475개 유전자): R² = 0.33 → **A안 채택**

### 08\_최종 모델 (CatBoost + Optuna)

- **전처리**: X, y 구성에 복잡한 전처리 필요
- **파이프라인**:
  - KMeans + Noise 데이터 증강 (3배)
  - StandardScaler → PCA 차원 축소
  - Optuna 하이퍼파라미터 튜닝 적용

#### <Best 성능>

- Best R²: **0.9703**
- Best Params:
  ```
  {'depth': 4, 'learning_rate': 0.0419998814673207, 'l2_leaf_reg': 1.4499985542901122}
  ```

#### <Test R² (지표별 성능)>

| 결과지표             | R²     |
| ---------------- | ------ |
| weight\_kg\_C\_A | 0.9803 |
| weight\_kg\_C\_B | 0.9668 |
| weight\_kg\_B\_A | 0.9800 |
| bmi\_C\_A        | 0.9706 |
| bmi\_C\_B        | 0.9496 |
| bmi\_B\_A        | 0.9859 |
| fat\_C\_A        | 0.9672 |
| fat\_C\_B        | 0.9811 |
| fat\_B\_A        | 0.9540 |
| pure\_fat\_C\_A  | 0.9667 |
| pure\_fat\_C\_B  | 0.9764 |
| pure\_fat\_B\_A  | 0.9650 |

→ **전체 평균 R²: 0.9703** → 매우 높은 예측 성능 달성

---

## 외부 대용량 파일 다운로드

이 프로젝트에서 사용된 `.db`, `.csv`, `.npy` 등의 대용량 파일은 GitHub 용량 제한으로 인해  
아래 Google Drive 링크를 통해 별도로 제공

👉 [Download external_data.zip (Google Drive)][([https://drive.google.com/file/d/1a2b3cXYZ/view?usp=sharing](https://drive.google.com/file/d/1_Iw_u2hCMIvgNgdS16fDgWrFgiXS5X2A/view?usp=sharing))](https://drive.google.com/file/d/1_Iw_u2hCMIvgNgdS16fDgWrFgiXS5X2A/view?usp=sharing)

## 참고 문헌

- RG Vink et al. *International Journal of Obesity*, 2017. "Adipose tissue gene expression is differentially regulated with different rates of weight loss..."



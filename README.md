# security-log-risk-prediction

보안 로그(`full_log`) 텍스트를 기반으로 위험도 등급(`level`)을 예측하는 텍스트 분류 프로젝트입니다.
TF-IDF 벡터화 + 트리 기반 모델(RandomForest / XGBoost / LightGBM)을 이용한 Baseline 코드를 제공합니다.

## 📁 폴더 구조

```
security-log-risk-prediction/
├── RandomForest/
├── XGBoost/
│   └── baseline_xgboost.ipynb
├── LightGBM/
└── README.md
```

## 🎯 문제 정의

- **Task**: 보안 로그 텍스트(`full_log`)를 입력으로 위험도 등급(`level`, 다중 클래스)을 예측하는 분류 문제
- **Input**: `full_log` (로그 원문 텍스트)
- **Target**: `level` (0~6 등급)
- **평가지표**: Accuracy, Macro F1, Weighted F1

## 📊 데이터

| 파일 | shape |
|---|---|
| `security_log_train.csv` | (472,972, 3) |
| `security_log_test.csv` | (1,418,916, 2) |
| `sample_submission.csv` | (1,418,916, 2) |

- Train/Valid 분리: `test_size=0.2`, `stratify=level`, `random_state=42`
- 결측치 처리: `full_log`는 빈 문자열로 채움, `level` 결측 행은 제거

### Class Distribution (train)
클래스 불균형이 심한 데이터셋입니다 (`level 0`, `1`이 대부분을 차지, `level 2/4/6`은 극소수).

## ⚙️ Baseline 파이프라인

모든 모델(RandomForest / XGBoost / LightGBM)이 동일한 파이프라인을 공유하며, **모델만 교체**한 구조입니다.

1. 데이터 로드 및 결측치 처리
2. 기본 EDA (클래스 분포, 로그 길이 분포 확인)
3. Train/Valid 분리 (stratified)
4. **TF-IDF 벡터화**
   - `max_features=5000`, `ngram_range=(1, 2)`, `min_df=3`, `max_df=0.95`
5. 모델 학습 (`n_estimators=100`, `max_depth=4`, `random_state=42` 등 공통 세팅 유지)
6. Validation 평가 (Accuracy / Macro F1 / Weighted F1 / Confusion Matrix)
7. 전체 데이터로 재학습 후 test 예측
8. (선택) 예측 확률(`predict_proba`)이 threshold 미만인 샘플을 `level 7`(미상 위험도)로 재분류하는 후처리 실험
9. `submission.csv` 생성

## 📈 모델별 Baseline 성능 (Validation)

| Model | Accuracy | Macro F1 | Weighted F1 |
|---|---|---|---|
| RandomForest | 0.9973 | 0.7977 | 0.9973 |
| XGBoost | 0.9980 | **0.8042** | 0.9980 |
| LightGBM | 0.9878 | 0.4991 | 0.9890 |

> Accuracy는 세 모델 모두 높지만, 클래스 불균형이 심해 소수 클래스(`level 2, 4, 6` 등)의 recall/precision 편차가 큽니다.
> **Macro F1 기준으로는 XGBoost > RandomForest ≫ LightGBM** 순으로, 소수 클래스 예측 성능은 XGBoost가 가장 우수합니다.
> LightGBM은 특히 `level 3`, `level 5`, `level 6`에서 낮은 precision을 보여 소수 클래스 예측이 불안정합니다.

## 🚀 실행 방법

```bash
# 필요 라이브러리 설치
pip install pandas numpy matplotlib seaborn scikit-learn lightgbm xgboost

# 각 모델 폴더의 노트북 실행
jupyter notebook XGBoost/baseline_xgboost.ipynb
```

각 노트북과 동일한 디렉토리에 아래 3개 파일이 필요합니다.
- `security_log_train.csv`
- `security_log_test.csv`
- `sample_submission.csv`

실행 결과는 각 노트북 경로에 `submission.csv`로 저장됩니다.

## 🔧 주요 하이퍼파라미터

| Model | 주요 설정 |
|---|---|
| RandomForest | `n_estimators=100`, `class_weight='balanced'`, `n_jobs=-1` |
| XGBoost | `objective='multi:softprob'`, `n_estimators=100`, `max_depth=4`, `learning_rate=0.1`, `subsample=0.8`, `colsample_bytree=0.8` |
| LightGBM | `objective='multiclass'`, `n_estimators=100`, `max_depth=4`, `learning_rate=0.1`, `subsample=0.8`, `colsample_bytree=0.8` |

## 💡 향후 개선 방향

- 클래스 불균형 완화: oversampling(SMOTE 등), class weight 튜닝, focal loss 적용
- TF-IDF 대신 로그 특화 토크나이저(정규식 기반 필드 분리) 또는 임베딩(BERT 계열) 적용
- 소수 클래스(`level 2, 4, 6`) 대상 별도 후처리/앙상블 전략
- `predict_proba` threshold 기반 `level 7`(미상) 분류 로직 최적화
- 모델 간 앙상블(Soft Voting/Stacking)로 Macro F1 개선

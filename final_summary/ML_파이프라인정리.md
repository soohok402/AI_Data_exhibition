## 데이터 전처리 파이프라인

1. **파생변수 생성 및 Parquet 저장**
2. **데이터셋 분포 비교** — 이상치 탐색
3. **상관성 분석** — 변수 관계 분석 및 이상치 탐색
4. **이상치 제거** — 확인된 데이터 오류 일괄 처리
5. **파생변수 재생성(필요시)** — 오류 처리에 따른 정합성 확보

### 파생변수 재생성 기준
1. **원자료 값 수정:** 수정된 값의 영향을 받는 파생변수 재계산
2. **행 제거(Long Format):** 시간 간격을 유지하도록 결측 시각을 복원하거나 시간 기준으로 연결한 후 재계산
3. **행 제거(Wide Format):** 각 행의 시계열 정보가 독립적이고 다른 행에 영향이 없다면 재계산 불필요

### 운영 원칙

- 실험 중에는 단계 간 반복 분석을 허용하되, 확인된 오류는 4단계에서 일괄 처리
- 최종 파이프라인은 확정된 오류 처리 규칙을 파생변수 생성 전에 적용하는 방향으로 정리
- 불필요한 데이터 버전 생성과 전체 파이프라인 재실행 최소화

---

## 모델링 파이프라인 (방법론 확정)

### 1. 변수군 Ablation: 변수의 필요성 검증, 변수선택 및 모델개선 목적
- 원본변수 및 파생변수 조합별 RMSE 비교
- 수위차분변수 채택
- 강우강도변수 보류

### 2. OHE 적용 여부 비교
- `station_id` OHE 포함 / 제외 모델 비교
- 성능 비교 후 최종 독립변수 구성 결정

### 3. Model Selection
- LightGBM, XGBoost, RandomForest, ExtraTrees 등 비교
- 동일한 Train/Val 조건에서 RMSE 평가
- 최종 후보 모델 선정

### 4. K-Fold 교차검증
- `GroupKFold` 활용 (`event_id` 기준, 호우사상 분리)
- 후보 모델의 성능 안정성 및 일반화 성능 확인
- 강우강도변수 포함해서 돌려보기
- 교차검증 결과를 바탕으로 최종 방법론 확정

### 5. 변수 기여도 평가: 변수별 예측에 대한 기여도와 방향(부호, 예측값을 높이는지 낮추는지)
- 최종 모델의 SHAP 분석
- 모델마다 실행법이 다름
- KF가 최종모델이면 K개 모델에 대해서 모두 시행, 변수별 절대값의 평균으로 산출

### 6. 최종 모델 학습
- 확정된 변수 조합 및 모델 사용
- Train + Val 전체 데이터로 재학습
- 필요시 OHE 적용 후 최종 데이터셋 저장

```
# =====================================================
# OneHotEncoder (희소행렬 안쓰고)
# =====================================================

import pandas as pd

# 원핫인코딩
X_train = pd.get_dummies(X_train, dtype=int)  # dtype=int(필수는 아님, 정수로 저장, 원래는 불리언)
X_val = pd.get_dummies(X_val, dtype=int)

# train 기준으로 컬럼명과 순서 일치
# Val에만 있는 컬럼은 제거, Train에만 있는 컬럼은 Val에 추가하여 0으로 채움
# OneHotEncoder(handle_unknown='ignore')와 유사한 처리
X_val = X_val.reindex(columns=X_train.columns, fill_value=0)
```

### (엑셀 업뎃용)리스트(list)와 딕셔너리(dict) 요소 추가·조회·변경
| 구분 | 리스트(list) | 딕셔너리(dict) |
|---|---|---|
| 데이터 구조 | 순서가 있는 요소 | key : value |
| 요소 조회 | `lst[index]` | `d[key]` |
| 요소 1개 추가 | `lst.append(value)` | `d[key] = value` |
| 여러 요소 추가 | `lst.extend([값1, 값2])` | `d.update({key1: 값1, key2: 값2})` |
| 기존 값 변경 | `lst[index] = value` | `d[key] = value` |
| 인덱스/키 | 0부터 시작하는 정수 위치 | 고유한 key |

**주의사항**
- `append([30, 40])` → 리스트 자체를 하나의 요소로 추가(중첩 리스트)
- `extend([30, 40])` → 30, 40을 각각 별도 요소로 추가(일반적으로 원하는 것)
- 리스트는 존재하지 않는 인덱스에 값을 넣으면 오류 발생
- 딕셔너리는 존재하지 않는 키를 넣으면 신규 키 생성



## Series.map() 사용법

### 1. 핵심 개념
- `Series.map()`은 원본 Series의 값을 매핑 규칙에 따라 **치환한 새로운 Series를 반환**한다.
- 원본 Series는 변경되지 않으며, 인덱스도 그대로 유지된다.
- 매핑되지 않는 값은 `NaN`이 된다. (일반적인 딕셔너리·Series 매핑 기준)

### 2. 기본 사용법

**① 딕셔너리로 매핑**
```python
new_series = df['기준컬럼'].map({'A': 1, 'B': 2})
```

**② Series로 매핑**
- `Series.map()` 안에는 **기준값이 인덱스이고 치환할 값이 데이터인 Series**를 넣는다.
- 원본 Series의 값을 매핑용 Series의 인덱스와 비교하여, 대응하는 값으로 치환한다.

```python
new_series = df['기준컬럼'].map(
    mapping_df.set_index('기준컬럼')['가져올컬럼']
)
```

### 3. 실제 적용 예시: event_id → fold_num

```python
# 원본 Series: event_id 값
kf_groups = train['event_id']

# 매핑 규칙: event_id를 인덱스로 설정하고 fold_num만 Series로 추출
# map(): event_id 값을 대응하는 fold_num 값으로 치환
kf_fold_num = kf_groups.map(
    kf_fold_map.set_index('event_id')['fold_num']
)

assert kf_fold_num.notna().all()

# 원본 DataFrame에 새 컬럼 추가
train['fold_num'] = kf_fold_num
```

**처리 과정**
1. `set_index('event_id')` : 매핑 기준값을 인덱스로 설정
2. `['fold_num']` : 치환할 값만 Series로 추출
3. `.map()` : 원본 `event_id` 값을 대응하는 `fold_num` 값으로 치환
4. `kf_fold_num` : 기존 행 인덱스가 유지된 새로운 Series 반환
5. `train['fold_num'] = kf_fold_num` : 인덱스 기준으로 자동 매칭하여 컬럼 추가

### 4. 주의사항
- 매핑용 Series의 인덱스(기준값)는 중복되지 않아야 한다.
- `Series.map()`은 값을 치환하며, 원본 DataFrame에 컬럼을 자동으로 추가하지 않는다.
- 새로운 Series를 원본 DataFrame에 추가할 때 인덱스가 일치하면 `reindex()`는 필요 없다.
- `DataFrame.map()`은 각 원소에 함수를 적용하는 별도의 기능이다.

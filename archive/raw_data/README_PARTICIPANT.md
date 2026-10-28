# 호우 시 유역환경을 반영한 6시간 최대 하천수위 상승량 예측

2026 KISTI DATA·AI 분석 경진대회 — 참가자 안내

---

## 1. 문제

현재 시점 `t` 까지의 **과거 수위**, **위성강우(IMERG)**, **지상강수**, **레이더 반사도**,
**유역환경** 정보를 이용하여 향후 6시간 동안의 **최대 하천수위 상승량**을 예측합니다.

한 행 = 하나의 `(수위관측소, 시각)` 예측 단위입니다.

---

## 2. Target

```
target_maxrise_6h = max(WL(t+1h), WL(t+2h), ..., WL(t+6h)) − WL(t)
```

- 단위: **m**
- **음수 값도 유효합니다** (예측 구간에 수위가 하강한 경우)
- 미래 강우·미래 수위는 입력변수에 포함되어 있지 않습니다

---

## 3. 제공 데이터

| 파일 | 행 수 | 컬럼 수 | 설명 |
|---|---|---|---|
| `train.csv.gz` | **749,657** | **26** | 학습 데이터. target 포함 |
| `test.csv.gz` | **186,263** | **25** | Hidden Test 입력. target 비공개 |
| `sample_submission.csv` | 186,263 | 2 | 리더보드 제출 양식 |
| `data_dictionary.csv` | 26 | - | 변수 정의 및 단위 설명 |

Train 과 Test 를 합한 전체 가공 데이터는 **935,920행** 입니다 (749,657 + 186,263).

- 수위관측소 수: **465개** (Train / Test 동일)
- 결측치(NaN) 없음, 중복 행 없음
- Train 과 Test 는 호우사상(event) 단위로 완전히 분리되어 있습니다

참가자는 Train 데이터를 이용하여 자유롭게 학습 및 검증 전략을 구성할 수 있습니다.

---

## 4. 컬럼 구성 (26개)

| 구분 | 개수 | 컬럼 |
|---|---|---|
| 식별정보 | 3 | `row_id`, `timestamp`, `station_id` |
| 과거 수위 | 6 | `wl_t`, `wl_t_minus_1h`, `wl_t_minus_3h`, `wl_t_minus_6h`, `wl_t_minus_12h`, `wl_t_minus_24h` |
| 위성강우(IMERG) | 3 | `imerg_rain_1h`, `imerg_rain_6h`, `imerg_rain_24h` |
| 지상강수 | 3 | `sfc_rain_1h`, `sfc_rain_6h`, `sfc_rain_24h` |
| 레이더 반사도 | 3 | `radar_reflectivity_mean`, `radar_reflectivity_p95`, `radar_reflectivity_ge20_fraction` |
| 유역환경 | 7 | `cat_area`, `wsarea`, `mean_elevation`, `mean_slope`, `urban_ratio`, `forest_ratio`, `stream_order` |
| Target | 1 | `target_maxrise_6h` — **Train 에만 있습니다** |

### 단위 안내

| 변수 | 단위 |
|---|---|
| `wl_*`, `target_maxrise_6h` | m |
| `imerg_rain_*` | mm |
| `sfc_rain_*` | **공식 단위 미확인** |
| `radar_reflectivity_mean`, `radar_reflectivity_p95` | **공식 단위 미기재** (dBZ 로 표기하지 않으며 강우량으로 변환된 값이 아님) |
| `radar_reflectivity_ge20_fraction` | 0~1 비율 |
| `mean_elevation` | m |
| `urban_ratio`, `forest_ratio` | % |
| `cat_area`, `wsarea`, `mean_slope` | 원자료 단위 유지 |
| `stream_order` | 하천차수. 소권역 평균값이라 비정수 가능 |

자세한 정의는 `data_dictionary.csv` 를 확인하세요.

---

## 5. timestamp / station_id 비식별화 안내

### timestamp

`timestamp` 는 **실제 관측일시가 아니라 비식별화된 합성 시각**입니다.

- 동일 호우사상 내부의 **시간 순서와 간격(1시간)** 은 유지되므로 시계열 구조는 보존됩니다
- 다만 **실제 연·월·일 및 절대 시각 정보는 보존되지 않습니다**
- 서로 다른 호우사상 사이의 실제 시간 관계도 유지되지 않습니다
- 합성 시각의 **시(hour) 값도 실제 관측 시각을 보존하지 않습니다**

따라서 실제 날짜·계절·시간대를 이용한 모델링은 의미가 없습니다.

### station_id

`station_id` 는 **실제 WAMIS 관측소 코드가 아닌 익명 ID** (`ST0001` ~ `ST0465`) 입니다.

Train 과 Test 에서 **동일 관측소는 항상 동일한 익명 ID** 를 사용하므로,
관측소별 특성을 학습하는 것은 그대로 가능합니다.

### row_id

`row_id` 는 실제 관측소·날짜 정보를 포함하지 않는 익명 식별자입니다.

---

## 6. 평가 방법

주 평가지표: **Global RMSE** (Root Mean Squared Error)

```
RMSE = sqrt(mean((prediction - target)^2))
```

- 단위: m
- **낮을수록 우수**
- Hidden Test **전체 행**에 대한 단일 RMSE 로 순위를 결정합니다
- 예측값 clipping 없음, 관측소·호우사상별 가중치 없음

---

## 7. 제출 형식

Test 데이터의 각 `row_id` 에 대한 6시간 최대 수위상승량 예측값을 CSV 형식으로 제출합니다.

```
row_id,pred_maxrise_6h
TE_000000001,0.123
TE_000000002,-0.045
TE_000000003,0.318
```

| 항목 | 값 |
|---|---|
| 제출 컬럼 | `row_id`, `pred_maxrise_6h` (정확히 2개) |
| 제출 행 수 | **186,263행** |
| 행 순서 | 무관 (`row_id` 기준으로 대조) |

---

## 8. 유의사항

- **음수 예측 허용** — target 자체가 음수를 가질 수 있습니다
- **결측(NaN) 예측 금지** — 모든 행에 값이 있어야 합니다
- **±inf, 숫자가 아닌 값 금지**
- **`row_id` 를 변경하거나 임의로 추가·삭제하지 마세요**
- `test.csv.gz` 의 모든 `row_id` 가 정확히 한 번씩 포함되어야 합니다
- **외부에 공개된 WAMIS 등 원자료와 평가 데이터를 대조하여 실제 관측소·시각을 식별하거나
  정답 수위를 직접 조회·복원하는 행위를 금지합니다**

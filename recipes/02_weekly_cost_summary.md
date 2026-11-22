# 원가 데이터 주간 집계

원가 내역 원천을 SQL로 집계해 전주 대비 변동을 요약한다. 집계 로직을 SQL로 남겨 두면 다음 주에도 같은 방식으로 다시 돌릴 수 있다.

## 입력과 산출물

- 입력: 원가 내역 CSV 또는 Excel (`data/`에 둔다). 최소 열은 일자, 원가 항목, 금액
- 산출물: `workbooks/weekly_cost.xlsx` (시트 `summary`, `variance`, `sql`)

## 단계

| 순서 | 도구 | 하는 일 |
|---|---|---|
| 1 | `duckdb` | 원천 파일을 그대로 읽어 행 수, 기간, 결측, 중복을 먼저 확인한다 |
| 2 | `duckdb` | 원가 항목을 분류체계에 매핑한다. 매핑되지 않은 항목은 `미분류`로 남긴다 |
| 3 | `duckdb` | 주차별·분류별 합계와 전주 대비 증감을 계산한다 |
| 4 | `excel` | `summary`에 집계표, `variance`에 증감 상위 항목, `sql`에 실행한 쿼리를 그대로 쓴다 |
| 5 | 모델 | 변동 요약을 쓴다. 숫자는 `summary` 시트의 셀을 인용한다 |

## 집계 쿼리 예시

```sql
SELECT
    date_trunc('week', cost_date) AS week,
    category,
    sum(amount) AS amount,
    sum(amount) - lag(sum(amount)) OVER (PARTITION BY category ORDER BY date_trunc('week', cost_date)) AS wow_change
FROM costs
GROUP BY 1, 2
ORDER BY 1, 2;
```

## 검증

- 집계 합계가 원천 파일의 전체 합계와 일치하는지 확인한다. 차이가 나면 매핑이나 조인에서 행이 빠지거나 불어난 것이다.
- `미분류` 금액의 비중을 요약에 함께 적는다. 비중이 크면 분류별 증감 해석을 미룬다.
- 요약 문장의 숫자를 `summary` 시트와 하나씩 대조한다. 모델이 쓴 문장에는 표에 없는 숫자가 들어가면 안 된다.
- 변동의 원인은 데이터로 확인한 것(`실측`)과 추정(`가정`)을 구분해 쓴다.

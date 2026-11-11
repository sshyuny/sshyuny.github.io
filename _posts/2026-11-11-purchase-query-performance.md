---
title: "구매 목록 조회 범위를 넓히면서 쿼리 비용을 줄인 과정"
description: "실행계획을 비교하며 구매 목록 조회의 JOIN·집계, offset, 날짜 조건과 정렬 비용을 개선한 과정을 정리합니다."
categories: [개발 경험, 성능 개선]
tags: [sql, mysql, query-optimization, pagination]
---

> 이 글은 실무에서 목록 조회 성능을 개선한 경험을 구매 목록 조회 사례로 재구성했습니다. 회사 정보 보호를 위해 업무 배경, 테이블·컬럼명, 쿼리 등 일부 내용을 생략하거나 재구성하고, 구체적인 운영 수치는 제외했습니다. 문제를 분석한 관점과 해결 과정, 기술적 선택의 이유를 중심으로 설명합니다.

## 1. 배경

### 1.1. 조회 범위 확대와 응답 속도

관리 화면에서 조회할 수 있는 구매 내역의 기간을 확대하면서 응답 속도도 개선해야 하는 요구가 있었다.
기존 조회 쿼리는 기간이 넓어질수록 부담이 커졌기 때문에, 쿼리 구조를 분석했다.

### 1.2. 검색 조건과 상세 정보

구매 목록에는 기간 외에도 구매 상태 등의 검색 조건이 있었고, 상품 종류 수와 총 구매 수량 같은 관련 정보도 함께 보여줘야 했다.
조회 속도를 개선하면서도 이러한 검색 조건과 상세 정보 제공을 함께 고려해야 했다.

## 2. 기존 구매 목록 조회 쿼리와 분석 과정

기존 조회는 관련 데이터를 JOIN하고 구매 건별로 집계한 뒤, 구매 ID 역순 결과에 offset을 적용하는 구조였다.

```sql
SELECT p.purchase_id,
       SUM(l.quantity) AS total_quantity
FROM purchases p
JOIN purchase_lines l ON l.purchase_id = p.purchase_id
WHERE p.recorded_at >= :from_time
  AND p.recorded_at < :to_time
GROUP BY p.purchase_id
ORDER BY p.purchase_id DESC
LIMIT :offset, :page_size;
```

LLM과 함께 기존 쿼리의 구조와 실행계획을 살펴보며 비용이 발생하는 지점을 나눠 분석했다.
각 문제에 대해 개선 쿼리 후보를 만들고, 후보별 실행계획을 비교하면서 읽는 범위와 JOIN·집계 대상, 정렬 방식이 어떻게 달라지는지 확인했다.

실행계획을 바탕으로 쿼리를 수정하고 다시 비교하는 과정을 반복하며, 각 비용을 줄일 수 있는 방법을 하나씩 찾아갔다.

## 3. 문제별 개선 과정

### 3.1. 뒤쪽 페이지를 건너뛰는 비용

#### 3.1.1. 문제

기존 조회는 뒤쪽 페이지로 갈수록 offset만큼 앞선 결과를 건너뛰어야 했다. 구매 ID를 한 페이지 분량만 먼저 고르는 파생테이블 중간안에서도 이 비용은 남았다. 같은 조회 조건에서 얕은 페이지와 깊은 페이지의 실행계획을 비교하며 이를 확인했다.

실행계획을 비교한 결과, 깊은 페이지에서는 구매 ID를 고르는 쿼리에서 읽을 것으로 추정된 행 수(`EXPLAIN`의 `rows` 값)가 늘어났다.

```sql
-- 반환하는 결과는 25건이지만, 앞선 결과 5,000건을 건너뛰고있다.
SELECT purchase_id
FROM purchases
ORDER BY purchase_id DESC
LIMIT 5000, 25;
```

#### 3.1.2. 해결 방법 — offset 대신 커서 사용

커서를 도입하여 이전 페이지에서 마지막으로 선택한 ID를 다음 요청의 커서로 전달했다.

```sql
SELECT purchase_id
FROM purchases
WHERE purchase_id < :last_seen_id
ORDER BY purchase_id DESC
LIMIT :page_size;
```

이로써 offset만큼 앞선 결과를 건너뛰는 비용을 없앴다. 검색 조건에 맞는 구매 ID를 찾는 비용과 추가 개선 후의 실행계획은 3.3절에서 다룬다.

### 3.2. 상세 JOIN과 집계 비용

#### 3.2.1. 문제

기존 쿼리의 실행계획을 확인하니, 한 페이지 분량을 고르기 전에 후보 구매 ID를 대상으로 상세 JOIN과 집계를 수행하고 있었다.
바깥 쿼리에는 `Using temporary; Using filesort`도 나타났다.

조회 기간이 넓어지면 후보 구매 ID가 많아지고, 이에 따라 상세 JOIN과 집계의 처리량도 늘어날 수 있었다.

#### 3.2.2. 시도한 방법 — 파생테이블에서 구매 ID 먼저 선택

상세 JOIN과 집계의 대상을 줄이기 위해 파생테이블에서 구매 ID를 먼저 고르는 방법을 검토했다. 앞 절의 커서를 적용하면 다음과 같은 구조다.

```sql
-- 파생테이블 방법 예시
SELECT p.purchase_id,
       SUM(l.quantity) AS total_quantity
FROM (
    SELECT purchase_id
    FROM purchases
    WHERE recorded_at >= :from_time
      AND recorded_at < :to_time
      AND purchase_id < :cursor -- 첫 페이지에서는 이 조건을 생략한다.
    ORDER BY purchase_id DESC
    LIMIT :page_size
) page -- 구매 ID를 먼저 고른 파생테이블
JOIN purchases p ON p.purchase_id = page.purchase_id
JOIN purchase_lines l ON l.purchase_id = p.purchase_id
GROUP BY p.purchase_id
ORDER BY p.purchase_id DESC;
```

하나의 쿼리 안에서 안쪽 쿼리로 검색 조건에 맞는 구매 ID를 한 페이지 분량만큼 고른 뒤, 바깥 쿼리에서 해당 ID의 상세 데이터를 JOIN하고 집계하는 방식이었다.

그러나 이 구조를 사용한 초기 커서 구현에서는 상세 JOIN을 거친 결과의 개수로 마지막 페이지인지 판단하고 있었다.
먼저 고른 구매 ID 중 일부가 INNER JOIN 과정에서 제외되면, 상세 조회 결과가 한 페이지 분량보다 적어질 수 있다.
이때 마지막 페이지라고 판단하면 뒤에 구매 내역이 남아 있어도 탐색을 중단할 수 있다는 문제점이 있었다.

#### 3.2.3. 최종 해결 방법 — 구매 ID 조회와 상세 조회 분리

이를 방지하기 위해 최종적으로 구매 ID 조회와 상세 조회를 별도 쿼리로 분리했다. 처음 고른 ID 목록을 기준으로 다음 페이지 조회 여부와 커서를 결정하도록 했다.

사용자가 입력한 검색 조건에 맞는 구매 ID를 한 페이지 분량만큼 먼저 조회하고, 해당 ID에 대해서만 상세 조회 쿼리를 실행했다.

```sql
-- 상세 조회 쿼리 예시
SELECT p.purchase_id,
       SUM(l.quantity) AS total_quantity
FROM purchases p
JOIN purchase_lines l ON l.purchase_id = p.purchase_id
WHERE p.purchase_id IN (:id_a, :id_b, :id_c) -- 앞에서 조회한 구매 ID
GROUP BY p.purchase_id
ORDER BY p.purchase_id DESC;
```

먼저 조회한 구매 ID로 상세 조회 범위를 제한해, 한 페이지 분량의 구매 건에 대해서만 JOIN과 집계를 수행하도록 했다.

### 3.3. 조회할 구매 내역을 고르는 비용

#### 3.3.1. 문제

상세 JOIN과 집계의 대상을 줄이고 offset을 커서로 바꾸더라도, 검색 조건에 맞는 구매 ID를 찾아 정렬하는 비용은 별도로 고려해야 했다.

이 비용은 offset을 유지한 파생테이블 중간안의 실행계획을 분석하면서 확인했다.

```sql
-- 중간안의 파생테이블에서 구매 ID를 고르는 쿼리 예시
SELECT purchase_id
FROM purchases
WHERE recorded_at >= :from_time
  AND recorded_at < :to_time
ORDER BY purchase_id DESC
LIMIT :offset, :page_size;
```

날짜 조건을 유지한 채 구매 ID 역순으로 정렬하니, 조회 기간에 따라 인덱스와 정렬 방식이 달라졌다.

- 짧거나 중간인 조회 기간에서는 날짜 조건을 지원하는 인덱스의 범위 조회와 함께 `Using filesort`가 나타났다. 날짜 조건으로 후보를 좁힐 수 있지만, 구매 ID 역순으로 반환하기 위한 추가 정렬 비용이 있었다.
- 긴 조회 기간에서는 구매 ID 인덱스의 역방향 스캔이 선택됐다. 구매 ID 역순으로 읽을 수 있어 추가 정렬을 피하는 데 유리한 계획이었다.

이처럼 조회 기간에 따라 추가 정렬이 필요한 계획과 필요 없는 계획이 달라졌다. 날짜 조건에 맞는 데이터를 찾는 기준과 정렬 기준이 달랐기 때문이다. 추가 정렬은 첫 페이지에서도 나타났으므로, 구매 ID를 고르는 쿼리의 범위 조건과 정렬 기준을 함께 개선해야 했다.

#### 3.3.2. 해결 방법 — 범위 조건과 정렬 기준을 구매 ID로 통일

날짜 범위를 먼저 구매 ID의 하한과 상한으로 바꿨다. 구한 하한과 상한을 페이지 조회 쿼리에 적용해, 범위 조건과 정렬 기준을 구매 ID로 통일했다. 이는 ID와 날짜 순서가 대체로 같다는 가정에 의존하는 선택이었으며, 그 한계는 뒤에서 다룬다.

```sql
-- 시작 경계에 해당하는 ID
SELECT purchase_id
FROM purchases
WHERE recorded_at >= :from_time
ORDER BY recorded_at ASC
LIMIT 1;

-- 종료 경계에 해당하는 ID: 이 값은 범위에 포함하지 않는다.
SELECT purchase_id
FROM purchases
WHERE recorded_at >= :to_time
ORDER BY recorded_at ASC
LIMIT 1;
```

구한 하한과 상한에 커서 조건을 함께 적용해 구매 ID를 조회했다.

```sql
SELECT purchase_id
FROM purchases
WHERE purchase_id >= :lower_id
  AND purchase_id < :upper_id
  AND purchase_id < :cursor -- 첫 페이지에서는 이 조건을 생략한다.
ORDER BY purchase_id DESC
LIMIT :page_size;
```

실행계획은 두 쿼리를 대상으로 확인했다.

- 첫 페이지 쿼리: 구매 ID 하한과 기본 상태 조건을 적용했다. 상한 조건은 포함하지 않았다.
- 후속 페이지 형태의 쿼리: 첫 페이지 쿼리에 임의의 커서 값을 적용했다.

확인 결과, 두 쿼리 모두 `Backward index scan`으로 구매 ID 역순으로 읽어 추가 정렬을 피했으며, 실행계획에도 `Using filesort`는 나타나지 않았다. 후속 페이지 형태의 쿼리에서 읽을 것으로 추정된 행 수(`EXPLAIN`의 `rows` 값)도 깊은 offset을 사용한 중간안보다 작았다.

## 4. 선택한 방법의 한계

### 4.1. 날짜와 ID의 순서가 다를 수 있다

날짜 범위를 구매 ID 경계로 바꾸는 방식은, 구매 ID가 증가하는 순서와 구매 내역이 기록된 날짜 순서가 같다는 가정에 의존한다. 두 순서가 다르면 검색 기간에 맞는 구매가 누락되거나, 기간 밖의 구매가 포함될 수 있다.

### 4.2. 모든 검색 조건에서 비용이 일정하지는 않다

조건에 맞는 구매 내역이 드물면 한 페이지를 채우기 위해 많은 후보를 읽어야 할 수 있다. 구매 건당 상세 데이터가 많으면 상세 조회 비용도 증가한다.

### 4.3. 화면 동작과 조회 일관성도 달라진다

커서 방식은 번호가 지정된 임의 페이지로 바로 이동하는 요구와 잘 맞지 않는다. 전체 건수를 생략하면 전체 페이지 수도 표시할 수 없다.

추가로, ID 선택과 상세 조회를 분리하는 설계에서는 두 조회 사이의 데이터 변경도 고려해야 한다.

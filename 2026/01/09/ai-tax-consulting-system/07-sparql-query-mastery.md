# SPARQL 쿼리 마스터하기

> **작성일**: 2026년 1월 9일
> **카테고리**: SPARQL, Query, Knowledge Graph
> **키워드**: SPARQL, RDF 쿼리, 그래프 패턴, 재무 분석 쿼리
> **시리즈**: [온톨로지 + AI 에이전트: 세무 컨설팅 시스템 아키텍처](https://blog.imprun.dev/120) (7부/총 20부)
> **대상 독자**: 온톨로지에 입문하는 시니어 개발자

---

## 핵심 질문

> **그래프 데이터를 어떻게 질의하는가?**

이 질문에 답하기 위해 이번 편에서 다루는 내용:
- SQL 조인 vs SPARQL 그래프 패턴 매칭의 본질적 차이
- 트리플 패턴과 변수 바인딩 메커니즘
- FILTER, BIND, OPTIONAL의 설계 의도
- 집계 함수와 서브쿼리로 재무비율 계산
- AI 에이전트가 SPARQL을 도구로 활용하는 방법

---

## 요약

SPARQL(SPARQL Protocol and RDF Query Language)은 RDF 데이터를 조회하는 W3C 표준 쿼리 언어다. SQL이 관계형 데이터베이스를 조회한다면, SPARQL은 그래프 데이터베이스(트리플 스토어)를 조회한다. 이 글에서는 SPARQL의 기본 문법부터 실제 세무 데이터 분석에 필요한 복잡한 쿼리까지 다룬다. 부채비율 계산, 연도별 매출 추세 분석, 비용 항목 비교 등 실무적인 쿼리를 작성한다.

---

## 이전 내용 복습

- **5부**: RDF로 세무 데이터를 트리플로 표현
- **6부**: OWL로 TBox(스키마) 정의

이제 저장된 데이터를 **조회**할 차례다.

---

## SPARQL이란?

### SQL과 SPARQL 비교

**SQL (관계형 DB)**
```sql
SELECT company_name, revenue
FROM income_statements
WHERE fiscal_year = 2024;
```

**SPARQL (그래프 DB)**
```sparql
SELECT ?company ?revenue
WHERE {
  ?is a fin:IncomeStatement ;
      fin:fiscalYear 2024 ;
      fin:belongsTo ?company ;
      acc:revenue ?revenue .
}
```

### 핵심 차이점

| 구분 | SQL | SPARQL |
|------|-----|--------|
| 데이터 구조 | 테이블 (행/열) | 그래프 (트리플) |
| 조인 | 명시적 JOIN | 패턴 매칭으로 암묵적 조인 |
| 스키마 | 고정 스키마 | 유연한 스키마 |
| 결과 | 테이블 | 바인딩 집합 |

---

## SPARQL 기본 문법

### 기본 구조

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>
PREFIX tax: <http://example.org/tax/>

SELECT ?변수1 ?변수2
WHERE {
  # 그래프 패턴 (트리플 패턴)
  ?주어 ?술어 ?목적어 .
}
```

### 트리플 패턴

SPARQL의 핵심은 **트리플 패턴**이다. 변수(`?`로 시작)와 상수를 조합하여 매칭되는 트리플을 찾는다.

```sparql
# 모든 손익계산서 찾기
?is a fin:IncomeStatement .

# A노무법인의 모든 재무제표 찾기
?fs fin:belongsTo tax:Company_A노무법인 .

# 2024년 손익계산서의 매출 찾기
?is a fin:IncomeStatement ;
    fin:fiscalYear 2024 ;
    acc:revenue ?revenue .
```

---

## 실습: 기본 SELECT 쿼리

### 테스트 환경 설정 (Python)

```python
from rdflib import Graph

# 그래프 로드
g = Graph()
g.parse("tax_tbox.ttl", format="turtle")
g.parse("tax_knowledge_graph.ttl", format="turtle")

def run_query(query):
    """SPARQL 쿼리 실행 및 결과 출력"""
    results = g.query(query)
    for row in results:
        print(row)
    return results
```

### 쿼리 1: 모든 회사 조회

```sparql
PREFIX tax: <http://example.org/tax/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?company ?name
WHERE {
  ?company a tax:Company ;
           rdfs:label ?name .
}
```

**결과**:
```
(tax:Company_A노무법인, "A노무법인")
```

### 쿼리 2: 특정 연도 재무 데이터 조회

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

SELECT ?fs ?year ?revenue ?netIncome
WHERE {
  ?fs a fin:IncomeStatement ;
      fin:fiscalYear ?year ;
      acc:revenue ?revenue ;
      acc:netIncome ?netIncome .
}
ORDER BY DESC(?year)
```

**결과**:
```
fin:IS_A노무법인_2024  2024  5297933879  341124233
fin:IS_A노무법인_2023  2023  4733703948  630485442
fin:IS_A노무법인_2022  2022  3012297097  514169773
```

### 쿼리 3: 재무상태표 요약

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

SELECT ?year ?assets ?liabilities ?equity
WHERE {
  ?bs a fin:BalanceSheet ;
      fin:fiscalYear ?year ;
      acc:totalAssets ?assets ;
      acc:totalLiabilities ?liabilities ;
      acc:totalEquity ?equity .
}
ORDER BY DESC(?year)
```

**결과**:
```
2024  2194433171  486333117   1708100054
2023  2579529430  1212553609  1366975821
```

---

## 필터링과 조건

### FILTER 절

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

# 매출 40억 이상인 연도만 조회
SELECT ?year ?revenue
WHERE {
  ?is a fin:IncomeStatement ;
      fin:fiscalYear ?year ;
      acc:revenue ?revenue .
  FILTER(?revenue >= 4000000000)
}
```

### 비교 연산자

| 연산자 | 설명 | 예시 |
|--------|------|------|
| `=`, `!=` | 같음, 다름 | `?year = 2024` |
| `<`, `>`, `<=`, `>=` | 크기 비교 | `?revenue > 5000000000` |
| `&&`, `\|\|` | AND, OR | `?year >= 2023 && ?year <= 2024` |
| `!` | NOT | `!(?year = 2022)` |

### 문자열 필터

```sparql
# 라벨에 "노무" 포함된 회사
FILTER(CONTAINS(?name, "노무"))

# 라벨이 "A"로 시작하는 회사
FILTER(STRSTARTS(?name, "A"))

# 정규표현식 매칭
FILTER(REGEX(?name, "^A.*법인$"))
```

---

## 집계 함수

### COUNT, SUM, AVG

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

# 총 재무제표 수
SELECT (COUNT(?fs) AS ?count)
WHERE {
  ?fs a fin:FinancialStatement .
}

# 3개년 평균 매출
SELECT (AVG(?revenue) AS ?avgRevenue)
WHERE {
  ?is a fin:IncomeStatement ;
      acc:revenue ?revenue .
}

# 3개년 매출 합계
SELECT (SUM(?revenue) AS ?totalRevenue)
WHERE {
  ?is a fin:IncomeStatement ;
      acc:revenue ?revenue .
}
```

### GROUP BY

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

# 연도별 총 비용 (판관비) 집계
SELECT ?year (SUM(?expense) AS ?totalExpense)
WHERE {
  ?is a fin:IncomeStatement ;
      fin:fiscalYear ?year ;
      acc:sellingAndAdminExpenses ?expense .
}
GROUP BY ?year
ORDER BY ?year
```

### HAVING (그룹 필터)

```sparql
# 평균 영업이익이 5억 이상인 회사
SELECT ?company (AVG(?opIncome) AS ?avgOpIncome)
WHERE {
  ?is a fin:IncomeStatement ;
      fin:belongsTo ?company ;
      acc:operatingIncome ?opIncome .
}
GROUP BY ?company
HAVING(AVG(?opIncome) > 500000000)
```

---

## 계산식 활용

### BIND로 계산 결과 저장

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

# 영업이익률 계산
SELECT ?year ?revenue ?opIncome ?opMargin
WHERE {
  ?is a fin:IncomeStatement ;
      fin:fiscalYear ?year ;
      acc:revenue ?revenue ;
      acc:operatingIncome ?opIncome .

  BIND(?opIncome * 100.0 / ?revenue AS ?opMargin)
}
ORDER BY DESC(?year)
```

**결과**:
```
2024  5297933879  420100720   7.93%
2023  4733703948  741223433   15.66%
2022  3012297097  472096906   15.67%
```

### 부채비율 계산

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

# 부채비율 = 부채총계 / 자본총계 × 100
SELECT ?year ?liabilities ?equity ?debtRatio
WHERE {
  ?bs a fin:BalanceSheet ;
      fin:fiscalYear ?year ;
      acc:totalLiabilities ?liabilities ;
      acc:totalEquity ?equity .

  BIND(?liabilities * 100.0 / ?equity AS ?debtRatio)
}
ORDER BY DESC(?year)
```

**결과**:
```
2024  486333117   1708100054  28.47%
2023  1212553609  1366975821  88.71%
```

---

## OPTIONAL: 선택적 패턴

누락된 데이터가 있어도 결과를 반환:

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

SELECT ?year ?revenue ?costOfSales
WHERE {
  ?is a fin:IncomeStatement ;
      fin:fiscalYear ?year ;
      acc:revenue ?revenue .

  # 매출원가는 없을 수도 있음 (서비스업)
  OPTIONAL { ?is acc:costOfSales ?costOfSales }
}
```

**결과** (A노무법인은 서비스업이라 매출원가 없음):
```
2024  5297933879  (null)
2023  4733703948  (null)
2022  3012297097  (null)
```

---

## 실전 분석 쿼리

### 쿼리 1: 연도별 성장률 분석

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

SELECT ?year ?revenue
       ((?revenue - ?prevRevenue) * 100.0 / ?prevRevenue AS ?growthRate)
WHERE {
  ?is a fin:IncomeStatement ;
      fin:fiscalYear ?year ;
      acc:revenue ?revenue .

  # 전년도 데이터
  ?prevIs a fin:IncomeStatement ;
          fin:fiscalYear ?prevYear ;
          acc:revenue ?prevRevenue .

  FILTER(?prevYear = ?year - 1)
}
ORDER BY DESC(?year)
```

**결과**:
```
2024  5297933879  11.92%   (전년 대비 성장률)
2023  4733703948  57.15%
```

### 쿼리 2: 비용 구조 분석

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>

SELECT ?year
       ?execSalary ?empSalary ?welfare ?advertising
       ((?execSalary + ?empSalary) * 100.0 / ?totalExpense AS ?salaryRatio)
WHERE {
  ?is a fin:IncomeStatement ;
      fin:fiscalYear ?year ;
      acc:sellingAndAdminExpenses ?totalExpense ;
      acc:executiveSalaries ?execSalary ;
      acc:employeeSalaries ?empSalary ;
      acc:welfareExpenses ?welfare ;
      acc:advertisingExpenses ?advertising .
}
ORDER BY DESC(?year)
```

**결과**:
```
2024: 인건비 비중 68.0% (임원 12억 + 직원 21.1억 / 판관비 48.7억)
2023: 인건비 비중 70.9%
2022: 인건비 비중 68.9%
```

### 쿼리 3: 재무 건전성 종합 분석

```sparql
PREFIX fin: <http://example.org/financial/>
PREFIX acc: <http://example.org/account/>
PREFIX tax: <http://example.org/tax/>

SELECT ?company ?year
       ?debtRatio ?currentRatio ?opMargin
       (IF(?debtRatio < 100, "양호",
          IF(?debtRatio < 200, "주의", "위험")) AS ?debtStatus)
WHERE {
  # 재무상태표
  ?bs a fin:BalanceSheet ;
      fin:belongsTo ?company ;
      fin:fiscalYear ?year ;
      acc:totalLiabilities ?liabilities ;
      acc:totalEquity ?equity ;
      acc:currentAssets ?curAssets ;
      acc:currentLiabilities ?curLiab .

  # 손익계산서
  ?is a fin:IncomeStatement ;
      fin:belongsTo ?company ;
      fin:fiscalYear ?year ;
      acc:revenue ?revenue ;
      acc:operatingIncome ?opIncome .

  # 재무비율 계산
  BIND(?liabilities * 100.0 / ?equity AS ?debtRatio)
  BIND(?curAssets * 100.0 / ?curLiab AS ?currentRatio)
  BIND(?opIncome * 100.0 / ?revenue AS ?opMargin)
}
ORDER BY ?company DESC(?year)
```

---

## Python에서 SPARQL 실행

### 완전한 예제

```python
from rdflib import Graph, Namespace

# 네임스페이스 정의
FIN = Namespace("http://example.org/financial/")
ACC = Namespace("http://example.org/account/")
TAX = Namespace("http://example.org/tax/")

def analyze_financial_health(graph, year=2024):
    """재무 건전성 분석 쿼리 실행"""

    query = """
    PREFIX fin: <http://example.org/financial/>
    PREFIX acc: <http://example.org/account/>
    PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

    SELECT ?companyName ?year
           ?totalAssets ?totalLiabilities ?totalEquity
           ?revenue ?operatingIncome ?netIncome
    WHERE {
      ?bs a fin:BalanceSheet ;
          fin:belongsTo ?company ;
          fin:fiscalYear ?year ;
          acc:totalAssets ?totalAssets ;
          acc:totalLiabilities ?totalLiabilities ;
          acc:totalEquity ?totalEquity .

      ?is a fin:IncomeStatement ;
          fin:belongsTo ?company ;
          fin:fiscalYear ?year ;
          acc:revenue ?revenue ;
          acc:operatingIncome ?operatingIncome ;
          acc:netIncome ?netIncome .

      ?company rdfs:label ?companyName .

      FILTER(?year = %d)
    }
    """ % year

    results = graph.query(query)

    for row in results:
        company = row.companyName
        assets = int(row.totalAssets)
        liabilities = int(row.totalLiabilities)
        equity = int(row.totalEquity)
        revenue = int(row.revenue)
        op_income = int(row.operatingIncome)
        net_income = int(row.netIncome)

        # 재무비율 계산
        debt_ratio = liabilities / equity * 100
        op_margin = op_income / revenue * 100
        roe = net_income / equity * 100

        print(f"\n=== {company} {year}년 재무 분석 ===")
        print(f"자산총계: {assets:,}원")
        print(f"부채총계: {liabilities:,}원")
        print(f"자본총계: {equity:,}원")
        print(f"매출액: {revenue:,}원")
        print(f"영업이익: {op_income:,}원")
        print(f"당기순이익: {net_income:,}원")
        print(f"\n--- 재무비율 ---")
        print(f"부채비율: {debt_ratio:.2f}%")
        print(f"영업이익률: {op_margin:.2f}%")
        print(f"ROE: {roe:.2f}%")

        # 건전성 판단
        if debt_ratio < 100:
            print("→ 부채비율 양호")
        elif debt_ratio < 200:
            print("→ 부채비율 주의")
        else:
            print("→ 부채비율 위험")

# 실행
g = Graph()
g.parse("tax_tbox.ttl", format="turtle")
g.parse("tax_knowledge_graph.ttl", format="turtle")

analyze_financial_health(g, 2024)
```

**출력**:
```
=== A노무법인 2024년 재무 분석 ===
자산총계: 2,194,433,171원
부채총계: 486,333,117원
자본총계: 1,708,100,054원
매출액: 5,297,933,879원
영업이익: 420,100,720원
당기순이익: 341,124,233원

--- 재무비율 ---
부채비율: 28.47%
영업이익률: 7.93%
ROE: 19.97%
→ 부채비율 양호
```

---

## 핵심 정리

| 구문 | 설명 | 예시 |
|------|------|------|
| `SELECT` | 조회할 변수 | `SELECT ?company ?revenue` |
| `WHERE` | 트리플 패턴 | `?is acc:revenue ?revenue` |
| `FILTER` | 조건 필터 | `FILTER(?year = 2024)` |
| `BIND` | 계산 결과 저장 | `BIND(?a / ?b AS ?ratio)` |
| `OPTIONAL` | 선택적 패턴 | 누락 데이터 허용 |
| `ORDER BY` | 정렬 | `ORDER BY DESC(?year)` |
| `GROUP BY` | 그룹화 | `GROUP BY ?company` |
| `HAVING` | 그룹 필터 | `HAVING(SUM(?x) > 100)` |

---

## 다음 단계 미리보기

**8부: SHACL 규칙으로 데이터 검증하기**

SPARQL로 데이터를 조회하고 분석할 수 있게 되었다. 하지만 "부채비율 200% 초과 시 경고"와 같은 **비즈니스 규칙**을 자동으로 검증하려면 SHACL이 필요하다. 8부에서는:

- SHACL 기본 문법
- 데이터 타입 검증
- 비즈니스 규칙 정의
- 위반 사항 자동 탐지

를 다룬다.

---

## 참고 자료

### SPARQL 표준
- [W3C SPARQL 1.1 Query Language](https://www.w3.org/TR/sparql11-query/)
- [SPARQL 1.1 Cheat Sheet](https://www.iro.umontreal.ca/~laMDFs/courses/IFT6281/pdf/sparql-cheat-sheet.pdf)

### 관련 시리즈
- [이전: OWL로 세무 용어 정의하기](https://blog.imprun.dev/126)
- [다음: SHACL 규칙으로 데이터 검증하기](https://blog.imprun.dev/128)
- [시리즈 목차](https://blog.imprun.dev/120)

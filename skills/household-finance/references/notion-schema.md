# Notion 가계부 DB 구조 레퍼런스

## DB ID

각 DB의 data_source_id는 플러그인 settings.json 환경변수에서 가져온다. SKILL.md의 "시작 전" 섹션 참조.

## 월별 예산 DB

```sql
("월 예산" TITLE, "담당" SELECT('남편','아내'),
 "월급" NUMBER FORMAT 'won', "월" DATE,
 "고정지출 항목" RELATION(고정지출항목),
 "고정지출비" ROLLUP(고정지출항목.예산금액.sum),
 "지출내역" RELATION(수입/지출) -- DUAL,
 "실제지출" ROLLUP(지출내역.금액.sum),
 "상환내역" RELATION(상환기록) -- DUAL,
 "상환금액" ROLLUP(상환내역.금액.sum),
 "저축내역" RELATION(저축기록) -- DUAL,
 "저축금액" ROLLUP(저축내역.금액.sum),
 "잔여예산" FORMULA(월급 - 고정지출비 - 실제지출 - 상환금액 - 저축금액),
 "메모" RICH_TEXT)
```

잔여예산 = 월급에서 모든 출금 경로를 뺀 **진짜 남은 돈**.
DUAL 릴레이션이므로 수입/지출·상환기록·저축기록 DB에 "월별예산" 컬럼 자동 생성됨.

## 고정지출 항목 DB

```sql
("항목명" TITLE, "담당" SELECT('남편','아내','공동'),
 "유형" SELECT('완전고정','변동고정'),
 "예산금액" NUMBER FORMAT 'won',
 "카테고리" SELECT('주거','통신','보험','빚상환','저축','교통','구독','교육','반려동물','기타'),
 "적용시작" DATE, "적용종료" DATE, "활성" CHECKBOX, "메모" RICH_TEXT)
```

- 완전고정: 매달 동일 | 변동고정: 매달 나가지만 금액 변동
- 적용종료 비어있음 = 영구 적용 | 활성 해제 = 예산 미연결

자동 연결 로직 (N월 예산 생성 시):
- 활성=✅ AND (담당=해당담당 OR 공동)
- 적용시작 ≤ N월 AND (적용종료 ≥ N월 OR 적용종료 비어있음)

## 수입/지출 DB

```sql
("내역" TITLE, "날짜" DATE, "구분" SELECT('수입','지출','이체'),
 "카테고리" SELECT('급여','부수입','식비','교통','주거','통신','보험','빚상환','저축','투자','의료','여가','쇼핑','기타'),
 "금액" NUMBER, "결제수단" SELECT('현금','카드','이체'),
 "담당" SELECT('남편','아내','공동'), "메모" RICH_TEXT,
 "월별예산" RELATION(월별예산))
```

카드 일시불 → 즉시 지출 | 할부 구매 → 빚정리+고정지출 | 카드대금 납부 → 기록 안 함

## 빚 정리 DB

```sql
("빚 이름" TITLE, "종류" SELECT('신용카드','주택대출','전세대출','마이너스통장','기타'),
 "총 빚 금액" NUMBER FORMAT 'won', "한도액" NUMBER FORMAT 'won',
 "월 상환액" NUMBER, "이자율(%)" NUMBER FORMAT 'percent',
 "상환 시작일" DATE, "상환 완료 예정일" DATE, "상환 완료" CHECKBOX,
 "상환기록" RELATION(상환기록),
 "총 상환금액" ROLLUP, "남은 잔액" FORMULA, "상환진행률(%)" FORMULA, "예상완료(개월)" FORMULA,
 "메모" RICH_TEXT)
```

## 상환 기록 DB

```sql
("내역" TITLE, "대상 빚" RELATION(빚정리), "날짜" DATE,
 "유형" SELECT('상환','추가인출','이자납부'),
 "금액" NUMBER, "메모" RICH_TEXT, "정산금액" FORMULA,
 "월별예산" RELATION(월별예산))
```

이자 처리: 마이너스통장→"이자납부" | 전세/주택→수입지출DB | 신용카드→"상환"

## 저축 목표 DB

```sql
("목표명" TITLE, "목표금액" NUMBER, "현재금액" ROLLUP,
 "진행률" FORMULA, "메모" RICH_TEXT)
```

## 저축 기록 DB

```sql
("내역" TITLE, "대상 목표" RELATION(저축목표), "날짜" DATE,
 "유형" SELECT('정기적금','자유적금','일시입금','이자','인출'),
 "금액" NUMBER, "메모" RICH_TEXT, "정산금액" FORMULA,
 "월별예산" RELATION(월별예산))
```

## 투자 관리 DB

```sql
("종목명" TITLE, "유형" SELECT('한국주식','ETF','펀드'),
 "매매유형" SELECT('매수','매도','배당'), "매매일" DATE,
 "단가" NUMBER, "보유 수량" NUMBER, "금액" NUMBER,
 "담당" SELECT('남편','아내','공동'), "메모" RICH_TEXT)
```

## 릴레이션 연결

Notion 검색으로 대상 이름을 찾아 URL을 릴레이션에 넣는다.
복수 연결: JSON 배열 `["url1", "url2"]`

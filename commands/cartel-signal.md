---
description: 담합 징후 리서치 파이프라인 — 수집·채점(researcher) 후 상급만 텔레그램 전송(reviewer) 순차 실행
---

# /cartel-signal — 공정위 담합 조사 징후 파이프라인

## 역할

두 에이전트를 순서대로 실행하는 오케스트레이터다:

```
cartel-signal-researcher → (결과 전달) → cartel-signal-reviewer
```

- Researcher: 전날 가격인상·담합 기사 수집 → 상/중/하 채점 → Google Drive '94. 공정거래 기사 리서치 > A. 담합 징후'에 표 형식 .docx 저장
- Reviewer: 저장된 파일에서 **상급(🔴)만** 선별·검증 → 텔레그램 전송

---

## STEP 1 — Researcher 실행

`cartel-signal-researcher` 에이전트를 아래 프롬프트로 호출한다:

```
담합 징후 리서치 수집 업무를 실행해 주세요.
전날 가격인상·담합 관련 기사를 수집·채점(상/중/하)하여
Google Drive '94. 공정거래 기사 리서치 > A. 담합 징후' 폴더에 표 형식 .docx로 저장하세요.
완료 후 반드시 포함하여 보고:
- 수집 성공 여부 (success / no_articles / error)
- 수집 날짜 (YYYYMMDD)
- 등급별 건수 (상 X / 중 Y / 하 Z)
- Google Drive 파일 링크
- 에러 메시지 (실패 시)
```

응답을 `RESEARCHER_RESULT`로 보관한다.

---

## STEP 2 — 결과 판단

- **success** (건수 > 0, 링크 존재) → STEP 3
- **no_articles** (전날 기사 0건, 주말/한산한 날 정상) → STEP 3 (Reviewer가 "상급 0건" 알림 전송)
- **error** (Drive 접근 실패, 업로드 실패 등) → 즉시 중단:

```
[파이프라인 중단] Researcher 실패로 Reviewer를 실행하지 않습니다.
에러: {에러 메시지}
조치: cartel-signal-researcher를 개별 실행하여 문제를 확인하세요.
```

---

## STEP 3 — Reviewer 실행

`cartel-signal-reviewer` 에이전트를 아래 프롬프트로 호출한다 (Researcher 결과를 컨텍스트로 주입):

```
담합 징후 검수·전송 업무를 실행해 주세요.

=== Researcher 실행 결과 (컨텍스트) ===
- 수집 날짜: {날짜}
- 등급별 건수: 상 {X} / 중 {Y} / 하 {Z}
- Google Drive 파일: {링크 또는 "없음"}
======================================

위 파일을 찾아 표를 파싱하고, 위험도 '상(🔴)' 항목만 선별·검증하여 텔레그램으로 전송하세요.
상급이 0건이면 "상급 징후 없음" 알림(중/하 건수 포함)을 텔레그램으로 전송하세요.
```

---

## STEP 4 — 완료 보고

```
═══════════════════════════════════════
  담합 징후 파이프라인 완료
═══════════════════════════════════════
  날짜: YYYY년 MM월 DD일
  수집: 상 X / 중 Y / 하 Z (Researcher)
  전송: 상급 X건 → 텔레그램 (Reviewer)
  Google Drive: {링크}
═══════════════════════════════════════
```

---

## 에러 처리 요약

| 상황 | 처리 |
|------|------|
| Researcher 성공 | Reviewer 실행 |
| Researcher 0건 (정상) | Reviewer 실행 (상급 0건 알림 전송) |
| Researcher 실패 | 즉시 중단, 에러 보고 |
| Reviewer 실패 | 에러 보고 (Researcher 결과는 이미 드라이브에 저장됨) |

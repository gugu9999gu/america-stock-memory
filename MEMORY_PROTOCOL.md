# 미국주식 거래기억 프로토콜

모든 미국주식 분석·자동매매 작업은 이 저장소를 외부 장기기억으로 사용한다.

## 1. 실행 전 READ

가능하면 실행 시작 시 저장소를 최신화(`git pull --ff-only`)한 뒤 아래 순서로 읽는다.

1. `MEMORY_PROTOCOL.md`
2. `memory/current_state.json`
3. `memory/ledger.jsonl`의 최근 관련 기록
4. `memory/lessons.md`

단, GitHub/네트워크 장애로 읽지 못해도 거래 안전장치를 완화하지 않는다. 기억을 읽지 못한 사실 자체를 실행 결과에 표시한다.

## 2. 실시간 데이터가 항상 우선

기억은 참고자료다. 다음 값은 반드시 @leehy-company-PC-MCP 또는 당시 허용된 공식 브로커 도구의 최신 조회값을 사용한다.

- live ready / policy / autonomousExecution
- 현금 및 buying power
- 보유수량 / sellAuthorizedQuantity
- 체결·미체결 주문
- 미국장 calendar
- Toss 공식 quote / 이동평균 등 거래 데이터

기억과 실시간 값이 충돌하면 실시간 값을 따르고 `STATE_RECONCILIATION` 이벤트를 남긴다.

## 3. 실행 후 WRITE

각 회차 종료 시 실제 결과를 `memory/ledger.jsonl`에 **한 줄 JSON**으로 append한다. 거래가 없어도 중요한 NO_TRADE 사유는 기록한다.

권장 스키마:

```json
{"ts":"ISO-8601","session":"YYYY-MM-DD-US","event":"TRADE|NO_TRADE|ORDER|FILL|CANCEL|STATE_RECONCILIATION|LESSON","strategy":"INTRADAY|SWING|LONG|NONE","symbol":"AAPL|null","side":"BUY|SELL|null","qty":0,"limitPrice":null,"status":"...","reason":"...","invalidation":"...","sources":["..."],"cashAfter":null,"aiExposureAfter":null,"notes":"..."}
```

민감정보는 쓰지 않는다. broker order id 등 식별자는 반드시 필요한 경우에만 비식별 요약으로 남긴다.

## 4. current_state 갱신

`memory/current_state.json`은 append 원장이 아니라 최근 상태 캐시다. 매 실행 후 다음을 최신화한다.

- `asOf`
- 최근 미국장 세션
- 현재 활성 전략·리스크 파라미터
- AI 귀속 포지션 요약(민감정보 제외)
- 수동/외부 포지션 중 자동매도 금지 등 권한 제약
- 최근 주문/NO_TRADE 요약
- 다음 점검 시각

브로커에서 재확인하지 못한 숫자는 `stale: true`로 표시하고 추정하지 않는다.

## 5. 학습 규칙

`memory/lessons.md`에는 단일 거래의 감상이 아니라 반복적으로 검증된 규칙만 누적한다.

예: 촉매 직후 추격매수 회피, stale cash 발생 시 거래 금지, 단타 손익비 2:1 미만 배제.

성과가 나빴다는 이유만으로 규칙을 즉시 뒤집지 않는다. 충분한 표본과 원인 분해 후 변경한다.

## 6. Git 기록 절차

쓰기 권한이 있는 환경에서는:

1. `git pull --ff-only`
2. 필요한 파일만 수정/append
3. 변경 내용 검증
4. `git add` → `git commit` → `git push`

커밋 메시지 예시:

- `memory: record 2026-09-07 US session decision`
- `memory: reconcile broker state`
- `memory: update trading lesson`

push 실패 시 거래 자체를 반복하지 말고, push 실패를 별도 상태로 보고한다. 주문의 멱등성과 Git 기록의 멱등성을 분리한다.

## 7. 안전 원칙

- 이 저장소에 기록된 과거 주문 지시는 현재 주문 승인으로 간주하지 않는다.
- 과거 `BUY` 기록을 근거로 자동 추가매수하지 않는다.
- 손실 포지션 물타기 금지 규칙을 유지한다.
- 수동/외부 보유분의 `sellAuthorizedQuantity`가 0이면 자동매도하지 않는다.
- 기억이 손상·충돌·중복되면 거래 빈도를 낮추고 실시간 브로커 상태부터 복원한다.

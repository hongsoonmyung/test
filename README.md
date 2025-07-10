# 중계서버-미들웨어 API 통합 문서

## 📋 목차
1. [시스템 개요](#시스템-개요)
2. [API 호출 플로우](#api-호출-플로우)
3. [API 상세 규격](#api-상세-규격)
4. [에러 코드](#에러-코드)
5. [예제](#예제)

---

## 🏗️ 시스템 개요

### 전체 시스템 아키텍처

```mermaid
graph TB
    subgraph "고객사 영역"
        Client[고객사 시스템]
    end
    
    subgraph "중계서버 영역"
        Relay[중계서버]
        Callback[Callback 엔드포인트]
    end
    
    subgraph "주차장 시스템"
        MW[주차장 미들웨어]
        ParkingDB[(주차장 DB)]
        Terminal[주차장 단말기]
    end
    
    Client -->|HTTP POST (비동기)| Relay
    Relay -->|비동기 요청| MW
    MW -->|데이터 조회/처리| ParkingDB
    MW -->|단말기 제어/확인| Terminal
    MW -->|Callback| Callback
    Callback -->|응답| MW
    Relay -->|최종 응답 (비동기)| Client
    
    classDef relayStyle fill:#e1f5fe
    classDef mwStyle fill:#f3e5f5
    classDef callbackStyle fill:#e8f5e8
    classDef clientStyle fill:#fff3e0
    classDef parkingStyle fill:#fce4ec
    
    class Relay relayStyle
    class MW mwStyle
    class Callback callbackStyle
    class Client clientStyle
    class ParkingDB parkingStyle
    class Terminal parkingStyle
```

### 주요 특징
- **중계서버 ↔ 미들웨어**: 비동기 처리 (Callback 방식)
- **미들웨어**: 주차장 DB 및 단말기와의 통신 담당
- **모든 API**: HTTP POST 방식으로 호출

### 콜백 처리 방식
- 모든 Request Body에는 `transactionId`가 존재
- 미들웨어에서 중계서버로 응답 시 `/api/v2/mw/callback/{transactionId}` 형태로 호출
- 모든 프로토콜은 HTTP POST로 호출

---

## 🔄 API 호출 플로우

### 1. 입차 조회 플로우

#### 시퀀스 다이어그램
```mermaid
sequenceDiagram
    participant Client as 고객사 시스템
    participant Relay as 중계서버
    participant MW as 주차장 미들웨어
    participant ParkingSystem as 주차장 시스템

    Note over Client,Relay: HTTP 연결 시작
    Client->>Relay: HTTP POST
    
    Note over Relay: 요청 수락, 비동기 처리 시작
    Relay->>MW: HTTP POST /incar/search
    Note over Relay,MW: Request Body: {transactionId, carNo, periodDays}
    MW-->>Relay: Response: {status, resultCode, resultMessage}
    
    MW->>ParkingSystem: 입차 정보 조회
    ParkingSystem-->>MW: 입차 데이터
    
    MW->>Relay: POST /api/v2/mw/callback/{transactionId}
    Note over MW,Relay: Callback Request: {status, resultCode, resultMessage, data}
    
    Relay-->>MW: Callback Response: {status, resultCode, resultMessage}
    
    Note over Relay: 처리 완료, HTTP 응답 전송
    Relay-->>Client: HTTP Response
    Note over Client,Relay: HTTP 연결 종료
```

#### API 엔드포인트
- **URL**: `POST /incar/search`
- **설명**: 차량번호로 입차 정보를 조회합니다.

#### 📥 Request (중계서버 → 미들웨어)
| 필드명 | 타입 | 필수 | 설명 | 예시 |
|--------|------|------|------|------|
| transactionId | string | Y | 트랜잭션 ID | "TXN_20241201_001" |
| carNo | string | Y | 차량번호 | "12가3456" |
| periodDays | integer | Y | 조회 기간 (일수) | 7 |

#### 📤 Response (미들웨어 → 중계서버)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "요청을 수락했습니다."
}
```

#### 📥 Callback Request (미들웨어 → 중계서버)
**URL**: `POST /api/v2/mw/callback/{transactionId}`

```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "정상 처리되었습니다.",
  "data": {
    "inCarDt": "20241201",
    "inCarSeqNo": "12345",
    "carNo": "12가3456",
    "inCarTm": "20241201T10:00:00",
    "inParkCustTy": "NORMAL",
    "inParkCutyTyName": "일반고객"
  }
}
```

#### 📤 Callback Response (중계서버 → 미들웨어)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "Callback을 수신했습니다."
}
```

---

### 2. 요금 조회 플로우

#### 시퀀스 다이어그램
```mermaid
sequenceDiagram
    participant Client as 고객사 시스템
    participant Relay as 중계서버
    participant MW as 주차장 미들웨어
    participant ParkingSystem as 주차장 시스템

    Note over Client,Relay: HTTP 연결 시작
    Client->>Relay: HTTP POST
    
    Note over Relay: 요청 수락, 비동기 처리 시작
    Relay->>MW: HTTP POST /incar/calc
    Note over Relay,MW: Request Body: {transactionId, inCarDt, inCarSeqNo,<br/>outScheduledTm}
    MW-->>Relay: Response: {status, resultCode, resultMessage}
    
    MW->>ParkingSystem: 요금 계산 및 할인 정보 조회
    ParkingSystem-->>MW: 요금 데이터 + 할인 정보
    
    MW->>Relay: POST /api/v2/mw/callback/{transactionId}
    Note over MW,Relay: Callback Request: {status, resultCode, resultMessage, data}
    
    Relay-->>MW: Callback Response: {status, resultCode, resultMessage}
    
    Note over Relay: 처리 완료, HTTP 응답 전송
    Relay-->>Client: HTTP Response
    Note over Client,Relay: HTTP 연결 종료
```

#### API 엔드포인트
- **URL**: `POST /incar/calc`
- **설명**: 입차 정보를 기반으로 주차 요금을 계산합니다.

#### 📥 Request (중계서버 → 미들웨어)
| 필드명 | 타입 | 필수 | 설명 | 예시 |
|--------|------|------|------|------|
| transactionId | string | Y | 트랜잭션 ID | "TXN_20241201_002" |
| inCarDt | string | Y | 입차일자 (YYYYMMDD) | "20241201" |
| inCarSeqNo | string | Y | 입차순번 | "12345" |
| outScheduledTm | string | Y | 출차예정시간 (ISO 8601) | "20241201T15:30:00" |

#### 📤 Response (미들웨어 → 중계서버)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "요청을 수락했습니다."
}
```

#### 📥 Callback Request (미들웨어 → 중계서버)
**URL**: `POST /api/v2/mw/callback/{transactionId}`

```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "정상 처리되었습니다.",
  "data": {
    "inCarDt": "20241201",
    "inCarSeqNo": "12345",
    "carNo": "12가3456",
    "inCarTm": "20241201T10:00:00",
    "outScheduledTm": "20241201T15:30:00",
    "originalParkChrg": 5000,
    "discountChrg": 1000,
    "parkChrg": 4000,
    "discountInfo": [
      {
        "discountMtd": "COUPON",
        "discountTkKnd": "PARKING_DISCOUNT",
        "discountAmt": 1000,
        "webDiscountRegSeq": "WEB001",
        "discountNumber": "DC001",
        "remark": "웹 할인권"
      }
    ]
  }
}
```

#### 📤 Callback Response (중계서버 → 미들웨어)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "Callback을 수신했습니다."
}
```

---

### 3. 할인권 등록 플로우

#### 시퀀스 다이어그램
```mermaid
sequenceDiagram
    participant Client as 고객사 시스템
    participant Relay as 중계서버
    participant MW as 주차장 미들웨어
    participant ParkingSystem as 주차장 시스템

    Note over Client,Relay: HTTP 연결 시작
    Client->>Relay: HTTP POST
    
    Note over Relay: 요청 수락, 비동기 처리 시작
    Relay->>MW: HTTP POST /incar/discount/add
    Note over Relay,MW: Request Body: {transactionId, inCarDt, inCarSeqNo,<br/>discountMtd, discountTkKnd, discountNumber,<br/>discountApplyDt, discountApplyTm, remark}
    MW-->>Relay: Response: {status, resultCode, resultMessage}
    
    MW->>ParkingSystem: 할인권 정보 저장
    ParkingSystem-->>MW: 저장 결과
    
    MW->>Relay: POST /api/v2/mw/callback/{transactionId}
    Note over MW,Relay: Callback Request: {status, resultCode, resultMessage}
    
    Relay-->>MW: Callback Response: {status, resultCode, resultMessage}
    
    Note over Relay: 처리 완료, HTTP 응답 전송
    Relay-->>Client: HTTP Response
    Note over Client,Relay: HTTP 연결 종료
```

#### API 엔드포인트
- **URL**: `POST /incar/discount/add`
- **설명**: 주차 할인권을 등록합니다.

#### 📥 Request (중계서버 → 미들웨어)
| 필드명 | 타입 | 필수 | 설명 | 예시 |
|--------|------|------|------|------|
| transactionId | string | Y | 트랜잭션 ID | "TXN_20241201_003" |
| inCarDt | string | Y | 입차일자 (YYYYMMDD) | "20241201" |
| inCarSeqNo | string | Y | 입차순번 | "12345" |
| discountMtd | string | Y | 할인방법 | "COUPON" |
| discountTkKnd | string | Y | 할인권종류 | "PARKING_DISCOUNT" |
| discountNumber | string | Y | 할인번호 | "DC001" |
| discountApplyDt | string | Y | 할인적용일자 (YYYYMMDD) | "20241201" |
| discountApplyTm | string | Y | 할인적용시간 (ISO 8601) | "20241201T10:00:00" |
| remark | string | N | 비고 | "웹 할인권" |

#### 📤 Response (미들웨어 → 중계서버)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "요청을 수락했습니다."
}
```

#### 📥 Callback Request (미들웨어 → 중계서버)
**URL**: `POST /api/v2/mw/callback/{transactionId}`

```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "정상 처리되었습니다."
}
```

#### 📤 Callback Response (중계서버 → 미들웨어)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "Callback을 수신했습니다."
}
```

---

### 4. 할인권 조회 플로우

#### 시퀀스 다이어그램
```mermaid
sequenceDiagram
    participant Client as 고객사 시스템
    participant Relay as 중계서버
    participant MW as 주차장 미들웨어
    participant ParkingSystem as 주차장 시스템

    Note over Client,Relay: HTTP 연결 시작
    Client->>Relay: HTTP POST
    
    Note over Relay: 요청 수락, 비동기 처리 시작
    Relay->>MW: HTTP POST /incar/discount/search
    Note over Relay,MW: Request Body: {transactionId, inCarDt, inCarSeqNo}
    MW-->>Relay: Response: {status, resultCode, resultMessage}
    
    MW->>ParkingSystem: 할인권 정보 조회
    ParkingSystem-->>MW: 할인권 데이터
    
    MW->>Relay: POST /api/v2/mw/callback/{transactionId}
    Note over MW,Relay: Callback Request: {status, resultCode, resultMessage, data}
    
    Relay-->>MW: Callback Response: {status, resultCode, resultMessage}
    
    Note over Relay: 처리 완료, HTTP 응답 전송
    Relay-->>Client: HTTP Response
    Note over Client,Relay: HTTP 연결 종료
```

#### API 엔드포인트
- **URL**: `POST /incar/discount/search`
- **설명**: 등록된 할인권 정보를 조회합니다.

#### 📥 Request (중계서버 → 미들웨어)
| 필드명 | 타입 | 필수 | 설명 | 예시 |
|--------|------|------|------|------|
| transactionId | string | Y | 트랜잭션 ID | "TXN_20241201_004" |
| inCarDt | string | Y | 입차일자 (YYYYMMDD) | "20241201" |
| inCarSeqNo | string | Y | 입차순번 | "12345" |

#### 📤 Response (미들웨어 → 중계서버)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "요청을 수락했습니다."
}
```

#### 📥 Callback Request (미들웨어 → 중계서버)
**URL**: `POST /api/v2/mw/callback/{transactionId}`

```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "정상 처리되었습니다.",
  "data": {
    "discountInfo": [
      {
        "discountMtd": "COUPON",
        "discountTkKnd": "PARKING_DISCOUNT",
        "discountAmt": 1000,
        "webDiscountRegSeq": "WEB001",
        "discountNumber": "DC001",
        "remark": "웹 할인권"
      }
    ]
  }
}
```

#### 📤 Callback Response (중계서버 → 미들웨어)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "Callback을 수신했습니다."
}
```

---

### 5. 할인권 삭제 플로우

#### 시퀀스 다이어그램
```mermaid
sequenceDiagram
    participant Client as 고객사 시스템
    participant Relay as 중계서버
    participant MW as 주차장 미들웨어
    participant ParkingSystem as 주차장 시스템

    Note over Client,Relay: HTTP 연결 시작
    Client->>Relay: HTTP POST
    
    Note over Relay: 요청 수락, 비동기 처리 시작
    Relay->>MW: HTTP POST /incar/discount/delete
    Note over Relay,MW: Request Body: {transactionId, inCarDt, inCarSeqNo,<br/>discountNumber}
    MW-->>Relay: Response: {status, resultCode, resultMessage}
    
    MW->>ParkingSystem: 할인권 정보 삭제
    ParkingSystem-->>MW: 삭제 결과
    
    MW->>Relay: POST /api/v2/mw/callback/{transactionId}
    Note over MW,Relay: Callback Request: {status, resultCode, resultMessage}
    
    Relay-->>MW: Callback Response: {status, resultCode, resultMessage}
    
    Note over Relay: 처리 완료, HTTP 응답 전송
    Relay-->>Client: HTTP Response
    Note over Client,Relay: HTTP 연결 종료
```

#### API 엔드포인트
- **URL**: `POST /incar/discount/delete`
- **설명**: 등록된 할인권을 삭제합니다.

#### 📥 Request (중계서버 → 미들웨어)
| 필드명 | 타입 | 필수 | 설명 | 예시 |
|--------|------|------|------|------|
| transactionId | string | Y | 트랜잭션 ID | "TXN_20241201_005" |
| inCarDt | string | Y | 입차일자 (YYYYMMDD) | "20241201" |
| inCarSeqNo | string | Y | 입차순번 | "12345" |
| discountNumber | string | Y | 할인번호 | "DC001" |

#### 📤 Response (미들웨어 → 중계서버)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "요청을 수락했습니다."
}
```

#### 📥 Callback Request (미들웨어 → 중계서버)
**URL**: `POST /api/v2/mw/callback/{transactionId}`

```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "정상 처리되었습니다."
}
```

#### 📤 Callback Response (중계서버 → 미들웨어)
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "Callback을 수신했습니다."
}
```

---

## 📋 API 상세 규격

### 공통 필드 구조

#### Request 공통 필드
모든 API 요청에는 다음 필드가 포함됩니다:
- `transactionId`: 트랜잭션 ID (필수)

#### Response 공통 필드
모든 API 응답에는 다음 필드가 포함됩니다:
- `status`: 상태 (SUCCESS/ERROR)
- `resultCode`: 결과 코드
- `resultMessage`: 결과 메시지
- `data`: 실제 업무 데이터 (선택적)

### Callback 처리 규칙
1. **Callback URL**: `/api/v2/mw/callback/{transactionId}`
2. **HTTP Method**: POST
3. **Content-Type**: application/json
4. **타임아웃**: 30초
5. **재시도**: 최대 3회

---

## ⚠️ 에러 코드

| 에러 코드 | 설명 | HTTP 상태 코드 |
|-----------|------|----------------|
| 0000 | 정상 처리 | 200 |
| 4001 | 필수 파라미터 누락 | 400 |
| 4002 | 인증 실패 | 401 |
| 4003 | 권한 없음 | 403 |
| 4004 | 리소스 없음 | 404 |
| 5001 | 서버 내부 오류 | 500 |
| 5002 | 미들웨어 통신 오류 | 500 |
| 5003 | 타임아웃 | 408 |

---

## 💡 예제

### 입차 조회 예제

#### 📥 중계서버 요청
```bash
curl -X POST https://middleware.example.com/incar/search \
  -H "Content-Type: application/json" \
  -d '{
    "transactionId": "TXN_20241201_001",
    "carNo": "12가3456",
    "periodDays": 7
  }'
```

#### 📤 미들웨어 즉시 응답
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "요청을 수락했습니다."
}
```

#### 📥 미들웨어 Callback (나중에 전송)
```bash
curl -X POST https://relay.example.com/api/v2/mw/callback/TXN_20241201_001 \
  -H "Content-Type: application/json" \
  -d '{
    "status": "SUCCESS",
    "resultCode": "0000",
    "resultMessage": "정상 처리되었습니다.",
    "data": {
      "inCarDt": "20241201",
      "inCarSeqNo": "12345",
      "carNo": "12가3456",
      "inCarTm": "20241201T10:00:00",
      "inParkCustTy": "NORMAL",
      "inParkCutyTyName": "일반고객"
    }
  }'
```

#### 📤 중계서버 Callback 응답
```json
{
  "status": "SUCCESS",
  "resultCode": "0000",
  "resultMessage": "Callback을 수신했습니다."
}
```

---

## 🔧 개발 가이드

### 1. 트랜잭션 ID 생성 규칙
- 형식: `TXN_YYYYMMDD_XXXXX`
- 예시: `TXN_20241201_001`

### 2. 날짜/시간 형식
- 날짜: `YYYYMMDD` (예: 20241201)
- 시간: `ISO 8601` (예: 20241201T15:30:00)

### 3. 비동기 처리 고려사항
- 모든 API는 즉시 응답 후 비동기 처리
- Callback 타임아웃: 30초
- 재시도 로직 구현 권장

### 4. 에러 처리
- HTTP 상태 코드와 resultCode 모두 확인
- 네트워크 오류 시 재시도
- Callback 미수신 시 타임아웃 처리

### 5. 콜백 구현 가이드
```javascript
// 중계서버 Callback 엔드포인트 구현 예시
app.post('/api/v2/mw/callback/:transactionId', (req, res) => {
  const { transactionId } = req.params;
  const { status, resultCode, resultMessage, data } = req.body;
  
  // 1. 트랜잭션 ID로 원본 요청 찾기
  const originalRequest = findRequestByTransactionId(transactionId);
  
  // 2. Callback 데이터 저장
  saveCallbackData(transactionId, { status, resultCode, resultMessage, data });
  
  // 3. 고객사에게 최종 응답 전송
  sendFinalResponseToClient(originalRequest, { status, resultCode, resultMessage, data });
  
  // 4. Callback 수신 확인 응답
  res.json({
    status: "SUCCESS",
    resultCode: "0000",
    resultMessage: "Callback을 수신했습니다."
  });
});
```

---

## 📚 관련 문서
- [OpenAPI 스펙 파일](./api_specification.yaml)
- [시퀀스 다이어그램](./api_flow_diagram.md)
- [프로토콜 정의](./protocol.txt) 

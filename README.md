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
        Relay[중계서버<br/>Request 세션 관리]
        Callback[Callback 엔드포인트]
    end
    
    subgraph "주차장 시스템"
        MW[주차장 미들웨어]
        ParkingDB[(주차장 DB)]
        Terminal[주차장 단말기]
    end
    
    Client -->|HTTP POST 요청| Relay
    Relay -->|비동기 요청| MW
    MW -->|데이터 조회/처리| ParkingDB
    MW -->|단말기 제어/확인| Terminal
    MW -->|비동기 응답 callback| Callback
    Callback -->|세션 기반 최종 응답| Relay
    Relay -->|최종 응답 전송| Client
    
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

### 대표적인 API 호출 시퀀스

```mermaid
sequenceDiagram
    participant Client as 고객사 시스템
    participant Relay as 중계서버
    participant MW as 주차장 미들웨어
    participant ParkingSystem as 주차장 시스템

    Note over Client,Relay: HTTP 연결 시작
    Client->>Relay: HTTP POST 요청
    
    Note over Relay: 요청 수락, 세션 생성, 비동기 처리 시작
    Relay->>MW: HTTP POST API 호출
    Note over Relay,MW: Request Body: {transactionId, ...}
    MW-->>Relay: Response: {status, resultCode, resultMessage}
    
    MW->>ParkingSystem: 주차장 시스템 처리
    ParkingSystem-->>MW: 처리 결과
    
    MW->>Relay: POST /api/v2/mw/callback/{transactionId}
    Note over MW,Relay: Callback Request: {status, resultCode, resultMessage, data}
    
    Relay-->>MW: Callback Response: {status, resultCode, resultMessage}
    
    Note over Relay: 세션 기반 최종 응답 전송
    Relay-->>Client: HTTP Response
    Note over Client,Relay: HTTP 연결 종료
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
| success | 정상 처리 | 200 |
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
    "transactionId": "550e8400-e29b-41d4-a716-446655440000",
    "carNo": "11가1234",
    "periodDays": 7
  }'
```

#### 📤 미들웨어 즉시 응답
```json
{
  "status": "200",
  "resultCode": "success",
  "resultMessage": "요청을 수락했습니다."
}
```

#### 📥 미들웨어 Callback (나중에 전송)
```bash
curl -X POST https://relay.example.com/api/v2/mw/callback/550e8400-e29b-41d4-a716-446655440000 \
  -H "Content-Type: application/json" \
  -d '{
    "status": "200",
    "resultCode": "success",
    "resultMessage": "정상 처리되었습니다.",
    "data": {
      "inCarDt": "20150710",
      "inCarSeqNo": "000001",
      "carNo": "11가1234",
      "inCarTm": "090000",
      "inParkCustTy": "1",
      "inParkCutyTyName": "일반고객"
    }
  }'
```

#### 📤 중계서버 Callback 응답
```json
{
  "status": "200",
  "resultCode": "success",
  "resultMessage": "Callback을 수신했습니다."
}
```

---

## 🔧 개발 가이드

### 1. 트랜잭션 ID 생성 규칙
- 형식: `UUID v4`
- 예시: `550e8400-e29b-41d4-a716-446655440000`
- 생성 방법: 표준 UUID 라이브러리 사용

### 2. 날짜/시간 형식
- 날짜: `YYYYMMDD` (예: 20241201)
- 시간: `HHMMSS` (예: 153000)

### 3. 비동기 처리 고려사항
- 모든 API는 즉시 응답 후 비동기 처리
- Callback 타임아웃: 30초
- 재시도 로직 구현 권장

### 4. 에러 처리
- HTTP 상태 코드와 resultCode 모두 확인
- 네트워크 오류 시 재시도
- Callback 미수신 시 타임아웃 처리

---

## 📚 관련 문서
- [OpenAPI 스펙 파일](./api_specification.yaml)
- [시퀀스 다이어그램](./api_flow_diagram.md)
- [프로토콜 정의](./protocol.txt) 

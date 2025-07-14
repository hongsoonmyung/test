중계서버와 미들웨어 간 프로토콜 정의

 
### 공통 부분 ###

1. 통신 방식 
    - HTTP Rest API
    - 미들웨어 특성 상 응답값을 HTTP Callback 형태로 처리

2. Callback 방식
    - 모든 Request Body 에는 transactionId 가 존재함. 
    - 미들웨어에서 중계서버로 응답 시 /api/v2/mw/callback/{transactionId} 형태로 호출


3. HTTP Method
    - 모든 프토토콜은 HTTP POST 로 호출
    - Common Request Field 
        - transactionId
        - 기타 필요 필드...

    - Common Response Field 
        - status
        - resultCode
        - resultMessage
        - data 

        * data 부분에 실제 업무 데이터 적재

    
### API 목록 ###

1. 입차 조회

    /incar/search

        Request 
            - transactionId
            - carNo
            - periodDays

        Response
            - status
            - resultCode
            - resultMessage

        Callback Request

            - status
            - resultCode
            - resultMessage
            - data 
                - inCarDt
                - inCarSeqNo
                - carNo
                - inCarDt
                - inCarTm
                - inParkCustTy
                - inParkCutyTyName

        Callback Response

            - status
            - resultCode
            - resultMessage

2. 요금 조회

    /incar/calc

        Request
            - transactionId
            - inCarDt
            - inCarSeqNo
            - outScheduledTm

        Response
            - status
            - resultCode
            - resultMessage

        Callback Request

            - status
            - resultCode
            - resultMessage
            - data 
                - inCarDt
                - inCarSeqNo
                - carNo
                - inCarDt
                - inCarTm
                - outScheduledTm
                - originalParkChrg
                - discountChrg
                - parkChrg
                - discountInfo
                    - discountMtd
                    - discountTkKnd
                    - discountAmt
                    - webDiscountRegSeq
                    - discountNumber
                    - remark

        Callback Response

            - status
            - resultCode
            - resultMessage              

3. 할인권 등록

    /incar/discount/add

        Request
            - transactionId
            - inCarDt
            - inCarSeqNo
            - discountMtd
            - discountTkKnd
            - discountNumber
            - discountApplyDt
            - discountApplyTm
            - remark

        Response
            - status
            - resultCode
            - resultMessage

        Callback Request

            - status
            - resultCode
            - resultMessage

        Callback Response

            - status
            - resultCode
            - resultMessage            

3. 할인권 조회

    /incar/discount/search

        Request
            - transactionId
            - inCarDt
            - inCarSeqNo

        Response
            - status
            - resultCode
            - resultMessage
            - data
                - discountInfo
                    - discountMtd
                    - discountTkKnd
                    - discountAmt
                    - webDiscountRegSeq
                    - discountNumber
                    - remark
        Callback Request

            - status
            - resultCode
            - resultMessage

        Callback Response

            - status
            - resultCode
            - resultMessage            



4. 할인권 삭제

    /incar/discount/delete

        Request
            - transactionId
            - inCarDt
            - inCarSeqNo
            - discountNumber

        Response
            - status
            - resultCode
            - resultMessage

        Callback Request

            - status
            - resultCode
            - resultMessage

        Callback Response

            - status
            - resultCode
            - resultMessage            



* 
변수 목록 (중복 제거)
🔑 공통 필드
transactionId - 트랜잭션 ID
status - 상태
resultCode - 결과 코드
resultMessage - 결과 메시지
data - 실제 업무 데이터
🚗 차량 관련
carNo - 차량번호
inCarDt - 입차일자
inCarSeqNo - 입차순번
inCarTm - 입차시간
inParkCustTy - 입차고객유형
inParkCutyTyName - 입차고객유형명
💰 요금 관련
outScheduledTm - 출차예정시간
originalParkChrg - 원래 주차요금
discountChrg - 할인요금
parkChrg - 주차요금
 할인권 관련
discountMtd - 할인방식
discountTkKnd - 할인권종류
discountAmt - 할인금액
webDiscountRegSeq - 웹할인등록순번
discountNumber - 할인번호
discountApplyDt - 할인적용일자
discountApplyTm - 할인적용시간
remark - 비고



* 
셈플데이터 (중복 제거)
🔑 공통 필드
transactionId - uuidv4 형태
status - 200
resultCode - success
resultMessage - 결과 메시지
🚗 차량 관련
carNo - 11가1234
inCarDt - 20150710
inCarSeqNo - 000001
inCarTm - 090000
inParkCustTy - 1
inParkCutyTyName - 일반고객
💰 요금 관련
outScheduledTm - 20250714170000
originalParkChrg - 5000
discountChrg - 3000
parkChrg - 2000
 할인권 관련
discountMtd - C
discountTkKnd - 10000010
discountAmt - 1000
webDiscountRegSeq - 0001
discountNumber - discount123456
discountApplyDt - 20250704
discountApplyTm - 090000
remark - 비고


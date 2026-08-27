# 5. 기술 구현 계획

> **작성 상태:** 세부 내용 작성 완료
>
> **핵심 결론:** 현재 핵심 제품은 양계장 계분벨트의 1개 Sampling Point에서 분변을 반복 소량채취하는 Sampling Unit, 시료를 전처리하고 상용 AIV 신속항원키트를 자동 검사하는 Analysis Unit, 검사결과를 축적·분석하는 AI Early Warning Platform으로 구성한다. 다지점 감시와 Epidemiological Risk Graph는 현재 제품과 분리된 단계적 고도화 기술로 구현한다.

5장은 4장에서 제시한 솔루션을 실제 장치·소프트웨어·데이터 구조로 구현하는 방법을 정의한다. 본 시스템은 AIV를 확진하지 않으며, 반복 가능한 현장 검사데이터를 생성하고 공식검사가 필요한 농장의 우선순위를 제시하는 예찰보조 시스템으로 구현한다.

## 5.1 전체 시스템 구성

### 1. 전체 아키텍처

시스템은 Farm Edge, AI Early Warning Platform, 향후 Movement Risk Layer의 세 계층으로 구성한다.

~~~text
[Farm Edge]

계분벨트 Sampling Point
        ↓
Sampling Unit
반복 소량채취 / Composite Sample
        ↓
Analysis Unit
혼합 / 침전 / 상층액 채취 / 자동분주
        ↓
AIV Rapid Cassette
        ↓
Camera AI Reader
정성 결과 / C·T Signal / T·C 반정량 신호
        ↓

[AI Early Warning Platform]

검사이력 DB
        ↓
동일 Sampling Point 시계열 분석
        +
농장 환경·생산 보조데이터
        ↓
Risk Score / Risk Level / Reason Codes
        ↓
자동 재검 / 관리자 확인 / 공식검사 요청 지원

        + 향후

[Movement Risk Layer]

Farm ↔ Vehicle ↔ Facility ↔ Farm
        ↓
Movement Risk / Network Risk
        ↓
관련 농장·이동관계 확인 우선순위
        ↓
전문기관 공식 역학조사 지원
~~~

### 2. 구성요소별 역할

| 구성요소 | 주요 역할 | 현재·향후 구분 |
|---|---|---|
| Sampling Unit | 계분벨트 운전 중 분변 반복 소량채취, Composite Sample 생성 | 현재 MVP |
| Analysis Unit | 혼합, 침전, 상층액 채취, AIV 키트 자동분주 | 현재 MVP |
| Camera AI Reader | C/T선 판독, 정성 결과와 반정량 신호 생성, 영상 QC | 현재 MVP |
| Edge Controller | 센서·모터 제어, 검사 State Machine, 시간기록 | 현재 MVP |
| AI Early Warning Engine | 시계열 특징값, Risk Score, Reason Codes, 재검 우선순위 | 현재 MVP |
| Dashboard | 장비상태, 검사이력, 위험도, 알림과 조치이력 표시 | 현재 MVP |
| Multi-point Monitoring | 동일 축사 내 2~3개 지점의 독립검사와 결과통합 | 향후 고도화 |
| Epidemiological Risk Graph | 농장·차량·시설 이동관계 분석 | 기관 협력 후 고도화 |
| Public System Integration | 공식검사·방역시스템과의 제한적 데이터 연계 | 권한·법적근거 확보 후 |

### 3. 하위제어와 상위연산 분리

ESP32 또는 STM32 계열 MCU는 실시간 장치제어를 담당한다.

- Belt Run Signal 또는 Encoder Monitoring
- Sampling Sequence
- 모터와 액추에이터 제어
- Home·Limit Sensor
- Cartridge·Cassette 장착 확인
- 누액감지와 Door Interlock
- 검사 타이머
- 상태 LED와 Buzzer

Raspberry Pi 또는 Mini PC는 영상분석과 데이터처리를 담당한다.

- Camera Capture
- OpenCV 기반 C/T선 판독
- T/C 반정량 신호 계산
- 검사 QC
- 검사이력 DB
- 시계열 분석
- Risk Score와 Reason Codes
- Dashboard·Server 통신

하위제어와 상위연산 사이에는 명확한 상태·명령 인터페이스를 둔다. 상위연산 오류가 발생하더라도 하위제어가 모터를 정지시키고 장치를 안전상태로 전환할 수 있도록 설계한다.

## 5.2 분변 통합시료 시스템

### 1. Sampling Point 기준 위치

산란계 케이지 농장의 계분벨트 구조를 기준으로 Sampling Unit의 1차 설치 위치는 다음과 같다.

> **계분벨트 끝단 스크레이퍼 이후, 횡방향 계분 컨베이어에 떨어지기 이전의 낙하구간**

이 위치는 기존 계분벨트를 직접 긁지 않고, 스크레이퍼가 벨트에서 분리한 계분 일부를 낙하 중에 채취할 수 있다. Sampling Cup은 평상시 낙하경로 밖에서 대기하여 정상적인 계분 배출을 방해하지 않는다.

현재 Sampling Point는 샘플러 1대가 직접 계분을 받는 물리적 지점을 의미한다. 해당 지점에서 얻은 시료를 축사 전체의 완전한 대표시료라고 표현하지 않는다.

### 2. 반복 소량채취

한 시점의 분변 한 덩어리에 의존하지 않도록 계분벨트 운전구간 중 여러 시점에 Sampling Cup을 짧게 진입시킨다.

~~~text
계분벨트 운전
0% ------------------------------------------------ 100%
       ●          ●          ●          ●          ●
     채취 1      채취 2      채취 3      채취 4      채취 N
                         ↓
                Composite Cartridge
~~~

여러 Sampling Event에서 얻은 소량 시료를 하나의 일회용 Composite Cartridge에 모아 해당 Sampling Point의 통합시료를 만든다.

실제 채취횟수, 간격, 1회 채취량과 총 시료량은 현재 확정하지 않는다. 계분벨트 운전조건, 기계 반복성 시험과 농장 실증을 통해 결정한다.

### 3. Sampling Unit 기계구조

MVP의 기준 메커니즘은 Swing 또는 Pivot Sampling Cup으로 둔다.

다회용 기계부:

- 고정 Bracket
- Swing Arm 또는 짧은 Linear Arm
- Servo·Stepper·Geared Motor 후보
- Home·Limit Sensor
- Cartridge Presence Sensor
- 필요 시 Load Cell
- 보호커버
- MCU 제어부

일회용 검체접촉부:

- Sampling Cup 또는 Disposable Insert
- Composite Cartridge
- 후속 전처리 카트리지와 연결되는 검체접촉부

계분벨트 표면을 다회용 Scraper가 지속적으로 긁는 방식, 다회용 배관으로 분변을 흡입하는 방식, 농장 전체를 이동하는 로봇형 채취기는 1차 기준안으로 채택하지 않는다. 기존 설비 간섭, 막힘, 교차오염과 세척부담을 줄이기 위해서다.

### 4. Sampling Timing

Sampling Trigger는 다음 우선순위로 검토한다.

1. 기존 계분벨트 제어반의 Motor ON·OFF 신호
2. 구동축 Encoder 또는 Hall Sensor를 이용한 이동량 감지
3. 전류·진동·광학센서를 이용한 비침습적 운전감지

농장 설비에서 안전하게 신호를 받을 수 있다면 제어반 신호 또는 Encoder를 우선 적용한다.

### 5. 다지점 확장

현재 제품은 1 Sampling Point / 1 Analysis Unit의 End-to-End 동작검증을 목표로 한다.

향후 동일 축사 내 2~3개 지점에 독립 Sampling Node를 배치할 경우 각 지점의 시료를 먼저 서로 섞지 않고 독립 검사한다. 특정 지점의 이상신호가 평균값에 의해 희석되지 않도록 지점별 원자료, T/C 추세와 재검결과를 유지한 뒤 소프트웨어에서 축사 위험도를 통합한다.

다지점 배치가 대표성이나 검출 가능성을 얼마나 개선하는지는 비교실증 전까지 확정적으로 주장하지 않는다.

## 5.3 AIV 신속항원 자동검사 장치

### 1. Wet Zone과 Dry Zone

장치의 핵심 설계원칙은 다음과 같다.

> **검체에 닿는 것은 일회용, 움직이고 읽고 계산하는 것은 다회용으로 구성한다.**

일회용 Wet Zone:

| 구성 | 기본방향 |
|---|---|
| Sampling Cup·Insert | 일회용 또는 교체형 |
| Composite Sample Container | 일회용 |
| 전처리·침전 카트리지 | 일회용 |
| 완충액 | 일회용 소모품 |
| Dropper·Pipette Tip | 일회용 |
| AIV Rapid Cassette | 일회용 |
| 폐기용 라이너·밀폐백 | 일회용 |

다회용 Dry Zone:

| 구성 | 역할 |
|---|---|
| Sampling Bracket·Arm | Cup 위치제어 |
| Sampling Actuator | Cup 진입·복귀 |
| Cartridge Holder | 카트리지 고정 |
| Mixing Motor | 카트리지 외부 혼합 |
| Z-axis | 상층액 채취·분주 위치제어 |
| Dropper Actuator | 흡입·분주 동작 |
| Cassette Stage | 키트 고정 |
| Camera·LED | 비접촉 판독 |
| MCU | 모터·센서 제어 |
| Raspberry Pi·Mini PC | 영상분석·AI·통신 |

검체가 다회용 펌프·튜브·밸브를 통과하지 않게 하여 교차오염과 세척부담을 줄인다.

### 2. 전처리·혼합·침전

Composite Sample은 일회용 전처리 카트리지 안에서 완충액과 밀폐 혼합한다. 카트리지 외부를 다회용 모터가 흔들어 내부 시료를 혼합하며 Vortex, 편심진동 또는 Rocking 방식을 후보로 둔다.

혼합이 끝나면 일정 시간 정지시켜 큰 고형물을 아래쪽으로 침전시킨다. MVP에서는 복잡한 필터구조를 필수로 두지 않고 상층액을 사용하는 방식을 우선 검증한다. 필터 필요성은 실제 분변상태와 키트 적용성 시험 이후 결정한다.

### 3. 상층액 자동채취와 분주

Z-axis가 일회용 Dropper 또는 Tip을 카트리지의 상층액 영역으로 이동시킨다. 침전물을 과도하게 흡입하지 않도록 카트리지 형상과 흡입위치를 반복 가능하게 고정한다.

~~~text
일회용 전처리 카트리지
        ↓
상층액 영역에 Tip 위치
        ↓
상층액 흡입
        ↓
AIV Cassette Sample Well로 이동
        ↓
지정량 자동분주
~~~

MVP에서는 카트리지 액면과 상층액 위치를 고정해 Z축 위치를 제어한다. 향후 필요하면 광학식 액면센서나 카메라 기반 액면감지를 검토한다.

### 4. AIV Cassette Stage

현재 MVP는 한 번에 1개의 Cassette를 검사하는 고정 Holder 구조로 구현한다.

- Cassette 장착 여부 확인
- 상층액 자동분주
- 검사 시작시각 기록
- 키트별 지정 반응시간 관리
- 판독시각 자동기록
- Camera Reader 촬영

향후에는 Cassette Magazine, Rotary Carousel, 자동배출, 순차검사와 Kit Lot·QR·Barcode 관리를 검토한다.

### 5. Kit Profile

상용 AIV 신속항원키트마다 검체 종류, 완충액량, 분주량과 판독시간이 다를 수 있다. 이를 코드에 고정하지 않고 Kit Profile로 분리한다.

Profile 값은 제조사의 사용설명서와 전문기관 검증결과를 기준으로 설정한다. 검증 전 임의의 시료량, 희석비와 판독 임계값을 확정하지 않는다.

### 6. 장치 State Machine

~~~text
IDLE
  ↓
BELT_DETECTED
  ↓
SAMPLING
  ↓
CARTRIDGE_READY
  ↓
MIXING
  ↓
SETTLING
  ↓
SUPERNATANT_PICKUP
  ↓
DISPENSING
  ↓
REACTION_WAIT
  ↓
IMAGING
  ↓
ANALYSIS
  ↓
SAVE_RESULT
  ↓
RETEST / COMPLETE / ERROR
~~~

Cartridge 또는 Cassette 미장착, Home Sensor 오류, 누액, Door 개방, 촬영실패와 C선 미검출은 별도 오류상태로 기록한다. 오류 발생 시 모터를 정지하고 장치를 안전상태로 전환한다.

## 5.4 카메라 기반 AI 판독 모듈

### 1. 촬영환경

판독 재현성을 위해 다음 조건을 최대한 일정하게 유지한다.

- 고정 카메라와 거리
- 고정 각도
- 고정 White LED와 Diffuser
- 외부광 차단
- 고정 Exposure·White Balance
- 제품별 지정 판독시간

### 2. 영상처리 파이프라인

~~~text
Cassette Image
      ↓
Cassette ROI 탐지
      ↓
Perspective·Position 보정
      ↓
C-line ROI / T-line ROI 추출
      ↓
배경보정과 신호 강도 계산
      ↓
C-line 유효성 확인
      ↓
정성 결과 + T/C 반정량 신호 + QC
~~~

초기 MVP에서는 데이터가 부족하므로 OpenCV 기반의 설명 가능한 영상처리를 우선 적용한다. 충분한 데이터가 확보되기 전에는 복잡한 딥러닝 모델을 핵심기술로 고정하지 않는다.

### 3. 판독 출력

- 정성 결과: Negative, Positive, Invalid, Uncertain
- 반정량 신호: C-line signal, T-line signal, T/C ratio 또는 동등 지표
- 품질정보: 초점, 노출, 키트 위치, 판독시간, C선 유효성
- 추적정보: 이미지 URI, 판독 알고리즘 버전, Kit Type·Lot

T/C 신호는 정성판독을 보조하고 동일 조건의 변화추세를 확인하는 optical signal이다. 전문기관 검증 전에는 바이러스 절대농도, RNA copy 수 또는 정량 진단값으로 해석하지 않는다.

### 4. 판독 품질관리

다음 조건은 Invalid 또는 Retake Required로 처리한다.

- C선 미검출
- 영상 초점불량
- 과노출·저노출
- Cassette 위치 이탈
- 판독시간 범위 이탈
- Kit Type 불일치
- 번짐 등 판독불가 패턴

애매한 결과를 무리하게 양성·음성으로 결정하지 않도록 Uncertain 상태를 유지한다.

## 5.5 시료 전처리 및 분석 연계

### 1. 시료 처리 인터페이스

~~~text
Sampling Point의 Composite Sample
        ↓
완충액 밀폐 혼합
        ↓
고형물 침전
        ↓
상층액 채취
        ↓
AIV 신속항원키트 자동검사
        ↓
정성 결과 / T/C 신호 / QC
~~~

이 흐름은 현장 신속항원검사를 위한 전처리다. 장치 내부에 RT-qPCR이나 공식 확진기능을 포함하지 않는다.

### 2. 검사 추적성

Sampling Point, Sampling Cycle, Composite Sample, RapidTest, TestImage와 OpticalMeasurement를 식별자로 연결한다.

- Sampling Point와 Belt Run
- Sampling Event 수와 Trigger 방식
- 시료와 키트 ID·Lot
- 전처리 시작시각
- 분주시각과 판독시각
- 이미지와 C/T 신호
- 정성 결과와 QC
- 재검 관계
- 위험도와 권고조치

모든 시간은 동일한 timestamp 정책으로 기록해 검사순서와 재검 관계를 재현할 수 있게 한다.

### 3. 전문기관 검사 연계

위험신호가 반복되면 전문기관 확인검사를 요청할 수 있도록 지원한다. 향후 OfficialTestReference를 통해 공식검사 진행상태와 결과를 연결하고 장치 경보와 기준검사를 비교할 수 있도록 설계한다.

실제 감염성 바이러스 취급, RT-qPCR, 민감도·특이도 검증, 최종 확진과 법정 방역판정은 전문기관의 영역으로 유지한다.

## 5.6 데이터 흐름

### 1. 검사 데이터 흐름

~~~text
Sensor / Actuator Event
        ↓
Device State
        ↓
Sampling Cycle / Sample
        ↓
RapidTest / TestImage
        ↓
OpticalMeasurement
        ↓
Time-series Feature
        ↓
RiskAssessment
        ↓
RetestRequest / Alert
        ↓
OfficialTestReference
~~~

### 2. 핵심 식별자와 데이터

| 구분 | 핵심 데이터 |
|---|---|
| 농장구조 | farm_id, barn_id, sampling_point_id, belt_id·row_id |
| 장비 | device_id, equipment_status, reader_version |
| 채취 | sampling_cycle_id, belt_run_id, sampling_event_count, trigger_type |
| 시료·검사 | sample_id, test_id, kit_type, kit_lot |
| 시간 | sampling_start·end, pretreatment_start, dispense_time, read_time |
| 영상·광학 | image_uri, c_signal, t_signal, t_c_ratio |
| 품질 | qc_status, qc_reason, retest_of |
| 보조정보 | environmental_snapshot, production_snapshot |
| 위험도 | risk_score, risk_level, reason_codes, recommended_action |

핵심 엔티티는 Farm, Barn, Device, SamplingPoint, Sample, RapidTest, TestImage, OpticalMeasurement, EnvironmentReading, RiskAssessment, Alert, RetestRequest, OfficialTestReference와 MaintenanceEvent로 구성한다.

### 3. Edge·Server 데이터 분리

Edge에서는 장치동작에 필요한 최근 데이터, 이미지 판독과 임시 검사이력을 유지한다. 서버 또는 플랫폼에서는 농장별 장기 검사이력, 위험도 변화, 경보와 사용자 조치이력을 관리한다.

통신이 끊겨도 현장 검사와 로컬 저장이 가능하게 하고, 연결이 복구되면 중복 없이 동기화할 수 있는 구조를 목표로 한다.

### 4. Mock·Real 인터페이스

다음 항목은 초기 MVP에서 Mock으로 처리할 수 있다.

- 실제 감염성 분변
- 바이러스 농도와 공식검사 결과
- 실제 농장 생산데이터
- 주변 발생정보 API
- 차량·시설 이동데이터

Mock 데이터도 실제 데이터와 같은 식별자와 스키마를 사용하여 이후 실제 인터페이스로 교체할 수 있게 한다.

### 5. 향후 이동관계 데이터

향후 Farm Movement Data는 vehicle_id, vehicle_type, company_id, farm_id, entry_time과 exit_time을 최소단위로 검토한다.

개인 실명과 연락처보다 차량·업체·시설·시간 중심의 가명 식별자를 우선한다. KAHIS 등 공식 시스템의 데이터는 기관 협력, 제공권한과 법적근거가 확보된 이후에만 연계한다.

## 5.7 AI 활용 구조

### 1. Camera AI

Camera AI는 Cassette 위치와 C/T선을 탐지하고 정성 결과, T/C 반정량 신호와 QC를 생성한다. 이는 현장 영상을 반복 가능한 구조화 데이터로 전환하는 역할이다.

### 2. AI Early Warning Engine

주요 입력:

- AIV 정성 검사결과
- C/T optical signal과 T/C 반정량 신호
- 동일 Sampling Point의 이전 검사결과
- 직전 검사 대비 변화량
- 최근 N회 변화추세와 연속 상승
- 자동 재검 결과
- QC 정보
- 농장별 baseline
- 환경·생산 보조데이터
- 주변 발생정보

주요 출력:

- Risk Score
- Risk Level: NORMAL, WATCH, CAUTION, ALERT
- Risk Reason Codes
- Recommended Action
- 자동 재검 우선순위
- 전문기관 확인검사 필요성에 대한 지원정보

### 3. 단계별 AI 개발

Phase A에서는 Rule-Based Risk Engine을 구현한다. 경보 이유를 사람이 확인할 수 있도록 Reason Code를 함께 제공한다.

Phase B에서는 농장과 Sampling Point별 정상분포가 확보되면 Robust Rolling Baseline, EWMA, Change Point Detection과 같은 통계적 이상탐지를 검토한다.

Phase C에서는 실증데이터와 공식검사 label이 충분히 확보된 이후 Logistic Regression, Tree 기반 모델과 시계열 분류모델을 비교한다. 데이터 분리, 재현성, calibration과 설명가능성을 모델 복잡도보다 우선한다.

AI 모델은 입력·출력 인터페이스를 고정하고 내부 모델을 교체할 수 있도록 모듈화한다.

### 4. 통합 위험모델

~~~text
Biological Risk
분변 AIV 검사 / 반복 T/C 변화

        +

Farm Context Risk
환경·생산·주변 발생정보

        + 향후

Movement / Network Risk
축산차량·시설 방문관계

        ↓

Integrated Risk Assessment
~~~

### 5. Epidemiological Risk Graph

Epidemiological Risk Graph는 현재 MVP가 아니라 단계적 고도화 기술이다. 허가된 Mock 데이터 또는 협력농장 자체 데이터로 Farm, Vehicle, Facility, Visit 모델과 Network Graph를 먼저 검증한다.

향후 공통 방문차량 수, 차량 종류, 방문순서, 시간간격, 반복방문, 연결농장 수와 각 농장의 자체 Risk Score를 Feature로 가공하여 Movement Risk와 확인 우선순위를 제안한다.

AI는 공식 역학관계를 확정하지 않으며 전문기관의 공식 역학조사를 지원한다.

## 5.8 현재 구현 가능범위

### 1. 팀이 현재 구현할 수 있는 범위

Hardware·Control:

- 계분벨트 낙하부 모형
- Swing Sampling Cup
- Belt Signal 또는 Encoder 기반 반복채취
- Composite Cartridge 생성
- 일회용 전처리 카트리지 혼합·침전
- 상층액 채취와 자동분주
- 1개 AIV Cassette Stage
- 고정 Camera·LED 판독챔버
- 센서·모터 State Machine

Software·AI:

- 장비상태와 검사 스케줄러
- Sampling Point·Sample·Kit ID 관리
- Camera Capture와 C/T ROI 판독
- T/C 반정량 신호와 QC
- 검사이력 DB
- Sampling Point별 시계열 분석
- Rule-Based Risk Score와 Reason Codes
- 자동 재검 로직
- Dashboard와 이상알림
- Mock Movement Graph 시각화

### 2. Mock으로 검증하는 범위

- 안전한 분변 모사물질
- 샘플 키트 이미지 또는 안전한 모사신호
- 가상 공식검사 결과
- 가상 환경·생산데이터
- 가상 차량·방문이력

### 3. 전문기관 협력이 필요한 범위

- 실제 감염성 AIV 검체
- 실제 분변 검체의 회수율과 방해요인
- 상용 키트와 자동화장치의 호환성
- 민감도·특이도와 위양성·위음성
- RT-qPCR 등 기준검사 비교
- T/C와 Ct·바이러스량 관계
- 실제 감염농장 또는 생물안전시설 시험
- 공식검사 결과와 시스템 경보의 일치도

### 4. 현재 확정하지 않는 값

- 최적 채취주기·횟수·간격
- 1회 및 Composite Sample의 최적량
- 한 Sampling Point의 대표범위
- 최적 희석비·침전시간·흡입높이
- 키트별 T/C 임계값
- 위험도 가중치
- 민감도·특이도
- 조기탐지 선행시간
- 다지점 배치의 성능개선 폭
- 비용절감률과 살처분 감소율

이 값들은 키트 IFU, 기계 반복성 시험, 전문기관 검증과 농장 실증을 통해 단계적으로 결정한다.

---

## 작성 기준

- [사업 아이템 기준정의](https://github.com/thjm0829/-ai/blob/main/BUSINESS_ITEM.md)
- [AIV 분변 자동검사 장치 개념설계](https://github.com/thjm0829/-ai/blob/main/DEVICE_CONCEPT.md)
- [메인 프로젝트 요구사항](https://github.com/thjm0829/-ai)

---

[이전: 4. 제안 솔루션](https://github.com/thjm0829/avian-ai-04-proposed-solution) · [전체 목록](https://github.com/thjm0829/-ai/blob/main/CHAPTER_REPOSITORIES.md) · [다음: 6. 기술 검증 및 실증](https://github.com/thjm0829/avian-ai-06-validation)


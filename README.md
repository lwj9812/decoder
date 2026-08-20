# 메모리 오류 감지 및 장애 예측 분석(PFA) 도구

## 개요
이 도구는 ARM 하드웨어 오류를 실시간으로 모니터링하여, ECC 메모리 오류를 물리적 DIMM 토폴로지와 매핑함으로써 신속한 진단을 가능하게 합니다. DRAM ECC 이벤트를 가로채고 복잡한 신드롬(Syndrome) 데이터를 분석하여 실행 가능한 **예측적 장애 분석(PFA, Predictive Failure Analysis)** 경고를 생성합니다.

이 프로젝트는 데이터 센터의 대규모 DRAM 장애 연구 논문들에서 깊은 영감을 받았습니다:
- *Predicting Memory Failures in Data Centers (USENIX ATC '19)*
- *DRAM errors in the wild: a large-scale field study*
- *DRAM failure prediction in AIOps: Empirical evaluation challenge*

이 도구는 오류 발생 빈도, 지속성(Persistence), 그리고 영향을 받은 DQ 레인(Lanes)을 분석하여 DRAM 오류를 분류하고, 시스템 다운타임이 발생하기 전에 임박한 **정정 불가능한 오류(UCE, Uncorrectable Error)**나 하드웨어 장애를 예측합니다.

## 주요 기능
- **실시간 로그 파싱**: 하드웨어 로그에서 DRAM ECC 이벤트를 지속적으로 모니터링하고 파싱합니다.
- **물리적 토폴로지 매핑**: 논리적 주소와 오류를 Socket, Channel, SubChannel, DIMM, Rank, CID, Bank Group, Bank, Row, Column을 포함한 물리적 토폴로지에 정확히 매핑합니다.
- **DQ 신호 분석**: `phy_lanes_bitmask`를 사용하여 활성 DQ 오류와 특정 영향을 받은 DQ 레인을 식별합니다.
- **예측적 장애 분석 (PFA)**:
  - **단일 소자/칩 장애(Single Device/Chip Fault)**: 완전한 칩 장애 전에 자주 발생하는 다중 비트 CE(Correctable Error) 폭풍(Storm)을 감지합니다.
  - **Link/RCD 인터페이스 신호 노이즈**: 신호 무결성 문제(PreReplay 오류)를 표시합니다.
  - **하드 고착 장애(Hard Stuck-at Faults)**: 지속성 비율(`persist_cnt` / `correctable_cnt`)을 사용하여 영구적인 셀 손상을 감지하고 페이지 오프라인(Page Offline) 조치를 권장합니다.
  - **CE 폭풍 및 UCE 위험**: 임박한 정정 불가능한 오류(UCE) 위험을 나타내는 높은 CE 발생 수치에 대해 경고합니다.
  - **Patrol Scrubbing/Sparing**: 일반적인 메모리 스크러빙(Scrubbing) 정정 작업과 실제 런타임 오류를 구분합니다.

## 오류 분류 모델
AIOps 및 대규모 데이터 센터 연구의 실증적 결과를 바탕으로, 이 도구는 다음과 같은 장애 시그니처를 구현합니다:
1. `CRITICAL UCE`: 정정 불가능한 오류 감지.
2. `[PFA: Single Device/Chip Fault (Critical)]`: 4개 이상의 활성 DQ 레인이 동시에 영향을 받음.
3. `[PFA: Link/RCD Interface Signal Noise]`: 일시적인 인터페이스 노이즈 감지(`pre_replay`).
4. `[PFA: Hard Stuck-at Fault (Action: Page Offline)]`: 지속성 비율 80% 이상의 높은 정정 가능 오류(CE) 발생.
5. `[PFA: CE Storm / Imminent UCE Risk]`: 단시간 내 정정 가능 오류 급증(1,000개 이상).

## 사용법
ARM 기반 서버에서 직접 도구를 실행합니다. 도구는 DIMM 토폴로지(`--dimm-info`)를 자동으로 가져오고 로그 모니터링을 시작합니다.

```bash
go run main.go
```

기본적으로 이 도구는 내부적으로 `arm_tool`을 호출하여 시스템 로그를 가져오고 추적합니다.

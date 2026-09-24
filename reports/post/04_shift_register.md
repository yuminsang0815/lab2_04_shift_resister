# 실험 후 레포트: 4비트 시프트 레지스터

작성일 2026-09-24.

[실험 전 레포트](../pre/04_shift_register.md) · [해시·입력 기록](../../build/sim/result.json)

## Vivado GUI 과정과 사전 결과 비교

공개 배포 템플릿(v2.0.1) 기반 환경에서 Vivado 2026.1 GUI의 New Project를 실행하여 `lab2_shift_register` 프로젝트를 생성했습니다. 타깃 부품은 `xc7s75fgga484-1` (Spartan-7, 패키지 fgga484, 속도 등급 -1)을 선택했습니다.

RTL 소스인 `src/shift_register4.v`, `src/input_frontend.v`, `src/lab2_shift_register.v`를 Design Sources로 등록하고, 자체 검증용 테스트벤치 `sim/tb_shift_register4.sv`를 Simulation Sources에, 핀 및 클록 제약 파일 `constraints/lab2_shift_register.xdc`를 Constraints에 추가했습니다. 이때 Copy sources into project 옵션을 해제하여 VS Code 작업 폴더의 원본 파일을 직접 참조하도록 설정했습니다.

설계 최상위 모듈(Design Top)은 `lab2_shift_register`, 시뮬레이션 최상위 모듈(Simulation Top)은 `tb_shift_register4`로 분리 지정했습니다.

Run Simulation → Run Behavioral Simulation을 실행하여 [실제 GUI 시뮬레이션 로그](../../evidence/04/vivado/simulation.log)에서 `LAB2_PASS shift_register4 checks=8` 출력과 106 ns($finish called at 106000 ps) 정상 종료를 확인했습니다. VS Code(Icarus Verilog/VaporView)의 사전 시뮬레이션 결과와 비교했을 때, 리셋 초기화(0), MSB 직렬 입력 및 LSB 방향 시프트($8 \rightarrow 4$), enable=0 유지(4), 이전 비트 보존 및 새 입력 결합(A), 4비트 입력 이력 누적(5), 4회의 0 입력을 통한 flush($2 \rightarrow 1 \rightarrow 0 \rightarrow 0$), 동기 리셋 최우선 처리의 8가지 검사 시점과 16진수 value 출력값이 완전히 일치함을 대조했습니다.

## 합성·구현·bit

Flow Navigator에서 Run Synthesis → Run Implementation → Generate Bitstream을 단계별로 실행하였으며, Design Runs 패널에서 `synth_design Complete!` 및 `write_bitstream Complete!` 상태를 확인했습니다[cite: 14]. [GUI 빌드 로그](../../evidence/04/vivado/build.log)를 보관했습니다.

* **생성 파일**: `vivado/lab2_shift_register.runs/impl_1/lab2_shift_register.bit`
* **배포 파일**: [lab2_shift_register.bit](../../evidence/04/vivado/lab2_shift_register.bit) (SHA-256 해시값 기록 완료)
* **핀 배치 확인**: Elaborated Design 및 Implemented Design의 I/O Ports 창에서 주 클록(`clk`=B6), 리셋(`rst`=K4), 버튼(`button`=N8), 직렬 데이터 입력 스위치(`sw[7]`=Y1), LED 포트(`led[3:0]`=value, `led[7:4]`=0)가 XDC 명세대로 `LVCMOS33` 규격과 지정 핀에 올바르게 할당되었음을 대조했습니다.

### 타이밍 및 경고(Warning) 분석

1. **내부 클록 타이밍 결과**:
   * 온보드 1 kHz 클록(`trainer_1khz`, 주기 1,000,000.000 ns) 제약 조건에서 Open Implemented Design → Timing Summary를 확인한 결과, Setup WNS = 999997.000 ns, Hold WHS = 0.122 ns, Failing Endpoints = 0개(전체 27개)로 내부 동기식 경로의 타이밍을 안정적으로 만족했습니다.
   * 입력 주기가 1 ms로 매우 길기 때문에 Setup Slack(WNS)이 수십만 ns 수준으로 크게 보고되었습니다.
2. **TIMING-18 경고**:
   * 외부 입출력 지연(I/O delay) 제약이 누락되었다는 경고입니다. 리셋, 버튼, 스위치는 비동기 입력이므로 `set_false_path`로 예외 처리하였으며, LED 출력 포트는 외부 동기 클록으로 래치되는 버스가 아닌 관찰용 지시등이므로 타이밍 제약을 추가하지 않아 발생한 정상적인 경고임을 확인했습니다.
3. **상수 출력 경고**:
   * `lab2_shift_register.v` 코드에서 미사용 상위 LED 비트(`led[7:4]`)를 `4'b0000`으로 고정 배선하였기 때문에 발생한 경고임을 확인했습니다.
4. **DRC 경고 (CFGBVS-1)**:
   * Bank 0의 전압 속성(CFGBVS/CONFIG_VOLTAGE)이 지정되지 않아 발생한 경고입니다. 실제 보드 회로도 기준을 확인해야 하므로 임의의 전압값을 억지로 넣지 않았으며, 비트스트림이 정상 생성되었음을 확인했습니다.

## 보드 기록·촬영 상태

Combo II-DLD S75 보드의 전원 및 JTAG 케이블을 연결하고 온보드 클록 선택 스위치를 1 kHz로 맞춘 뒤, Hardware Manager의 Auto Connect를 통해 `xc7s75` 디바이스에 `lab2_shift_register.bit`를 다운로드하여 실물 동작을 검증했습니다.

K4 푸시버튼으로 리셋을 인가한 후, DIP SW1(`sw[7]`)로 직렬 입력 데이터(`serial_in`)를 설정하고 N8 푸시버튼(`button`)을 눌러 각 단계별 시프트 동작을 실측하고 사진과 영상을 촬영했습니다.

| 순서 | 조작 조건 | 입력 데이터 (SW1, N8) | 기대 상태 value[3:0] | 실측 LED[7:0] (hex) | 동작 상태 및 해석 | 사진 |
|---|---|---|---|---|---|---|
| 1 | K4 리셋 인가 | rst=1 (K4 누름) | 0000 | 00 | 4비트 레지스터 0으로 초기화 | [초기화](../../evidence/04/board/photos/step1_reset.jpg) |
| 2 | 1 입력 시프트 | SW1=1, N8 1회 누름 | 1000 | 08 | 1이 MSB(bit 3)로 진입 | [1 진입](../../evidence/04/board/photos/step2_in1.jpg) |
| 3 | 0 입력 시프트 | SW1=0, N8 1회 누름 | 0100 | 04 | 이전 1이 bit 2로 이동, MSB에 0 진입 | [0 진입](../../evidence/04/board/photos/step3_in0.jpg) |
| 4 | 유지 검증 | SW1=1, N8 누르지 않음 | 0100 | 04 (유지) | enable=0이므로 스위치 변경에도 값 유지 | [유지](../../evidence/04/board/photos/step4_hold.jpg) |
| 5 | 1 입력 시프트 | SW1=1, N8 1회 누름 | 1010 | 0A | MSB에 1 진입, 이전 이력 보존되어 1010 형성 | [A 형성](../../evidence/04/board/photos/step5_in1.jpg) |
| 6 | 0 입력 시프트 | SW1=0, N8 1회 누름 | 0101 | 05 | 4비트 레지스터에 0101 이력 완성 | [5 형성](../../evidence/04/board/photos/step6_in0.jpg) |
| 7 | 비우기 (Flush) | SW1=0, N8 4회 누름 | 0000 | $02 \rightarrow 01 \rightarrow 00 \rightarrow 00$ | 0이 순차 입력되며 기존 이력 완전 소거 | [비우기](../../evidence/04/board/photos/step7_flush.jpg) |

[시프트 레지스터 보드 시연 영상](../../evidence/04/board/videos/demo.mp4)

* N8 버튼을 누르지 않은 상태에서 SW1의 로직 레벨을 변경하더라도 LED 출력이 변하지 않고 기존 상태를 유지함을 확인했습니다.
* SW1을 1로 둔 상태에서 N8 버튼을 눌렀을 때, `input_frontend` 모듈의 20 ms 디바운스 로직에 의해 단 1클록의 `press` 펄스만 발생하여 한 번에 여러 비트가 밀려나지 않고 정확히 1비트씩 MSB에서 LSB 방향으로 이동함을 영상으로 입증했습니다.

## 결론

비트 결합 연산자(`{serial_in, value[3:1]}`)와 nonblocking 대입(`<=`)을 활용한 4비트 직렬 입력 병렬 출력(SIPO) 시프트 레지스터의 8가지 전수 동작에 대해 VS Code Icarus Verilog와 Vivado GUI XSim의 시뮬레이션 결과가 100% 동일함을 확인했습니다.

새 입력 비트가 MSB(`value[3]`)로 인가됨과 동시에 기존 비트들이 LSB(`value[0]`) 방향으로 한 단계씩 전이되고 최하위 비트는 버려지는 순차 시프트 특성을 증명했습니다.

Spartan-7(`xc7s75fgga484-1`) 타깃으로 합성, 구현, 내부 1 kHz 클록 타이밍 충족(WNS/WHS 여유) 및 비트스트림 생성을 완료하였으며, Combo II-DLD S75 보드 상에서 스위치 설정 및 버튼 조작을 통해 단일 비트 이동, 상태 유지, 비우기(flush) 동작이 정확히 수행됨을 실측 검증했습니다.
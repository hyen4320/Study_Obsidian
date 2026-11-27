> 제어기 HMI SCADA 분석
> 조사일: 2026-11-26 · 소스 읽기만 수행(빌드·실행 없음). 줄번호는 클론 시점 커밋 기준
> 요약: OpenPLC_v3 · FUXA · Rapid SCADA v6의 추상화와 거기서 유추한 요구사항
> 관련: [[보안/ICS SCADA/개발/공통요구사항]] · [[보안/ICS SCADA/연구/오픈소스 프로젝트]]

## 프로젝트별 한 장 요약

아래 그림은 이 노트의 내용을 프로젝트마다 한 장으로 줄인 것이다. 근거(파일 · 줄번호)는 그 아래 본문에 있다.

### OpenPLC v3

![[ics-p01-OpenPLC.png]]

코드 구조 (모듈 · 의존 · 추상화 구현 방식)

![[ics-c01-OpenPLC.png]]

### FUXA

![[ics-p02-FUXA.png]]

코드 구조 (모듈 · 의존 · 추상화 구현 방식)

![[ics-c02-FUXA.png]]

### Rapid SCADA v6

![[ics-p03-RapidSCADA.png]]

코드 구조 (모듈 · 의존 · 추상화 구현 방식)

![[ics-c03-RapidSCADA.png]]

---

## 상세 분석 (원문)

# ICS/SCADA 그룹 A 보고서 — 제어기 · HMI · SCADA 플랫폼

대상: OpenPLC_v3(소프트 PLC), FUXA(웹 SCADA/HMI), Rapid SCADA v6(scada-v6).
방법: 소스 읽기만 수행(빌드·실행·수정·네트워크 없음. 단, OpenPLC의 `openplc.db`는 sqlite 읽기 전용 모드로 스키마/기본 행만 조회).
표기: `파일:줄`은 직접 확인한 줄번호. 확인하지 못한 것은 "미확인". 경로 접두어는 생략했으며 기준 디렉터리는 다음과 같다.

- OpenPLC: `/home/com/Study_agents/oss/ics/OpenPLC_v3/webserver/` (이하 `OP/`)
- FUXA: `/home/com/Study_agents/oss/ics/FUXA/server/` (이하 `FX/`), 클라이언트 모델은 `/home/com/Study_agents/oss/ics/FUXA/client/`
- Rapid SCADA: `/home/com/Study_agents/oss/ics/scada-v6/` (이하 `RS/`)

---

## 1. OpenPLC_v3 (소프트 PLC 런타임)

### 1.1 최상위 구조와 데이터 흐름

프로세스는 2개다.

1. 웹/관리 프로세스(Python Flask): `OP/webserver.py`. HTTP 8080(`webserver.py:2747`, `host='0.0.0.0'`, TLS 없음)과 HTTPS REST 8443(`webserver.py:2695`, 자체 서명 인증서)을 스레드 2개로 띄운다(`webserver.py:2765-2766`).
2. 런타임 프로세스(C/C++ 바이너리 `core/openplc`): 웹 프로세스가 `subprocess.Popen(['./core/openplc'])`로 기동(`OP/openplc.py:86`).

두 프로세스 사이의 제어 채널은 텍스트 RPC다. 런타임이 127.0.0.1:43628에 "interactive server"를 열고(`core/utils.cpp:152-156`, `core/interactive_server.cpp:180`), 웹이 `quit()`, `start_modbus(port)` 같은 문자열을 보낸다(`openplc.py:89-103`, `openplc.py:233-267`).

PLC 프로그램 적재는 "소스 → 재컴파일 → 런타임 바이너리 교체" 방식이다. 업로드된 ST 파일을 `iec2c`로 C로 변환(`scripts/compile_program.sh:20`), `glue_generator`가 `glueVars.cpp` 생성(`compile_program.sh:69,108`), `g++ *.cpp *.o -o openplc`로 런타임 전체를 다시 링크한다(`compile_program.sh:71,111`). 즉 사용자 로직·하드웨어 계층·프로토콜 서버가 하나의 바이너리로 정적 결합된다.

스캔 사이클(`core/main.cpp:160-237`):

```
updateBuffersIn()            // 하드웨어 → 입력 이미지 (main.cpp:182)
lock(bufferLock)             // main.cpp:184
updateBuffersIn_MB()         // Modbus 마스터가 읽어 둔 슬레이브 값 → 입력 이미지 (203)
handleSpecialFunctions()     // 시각/사이클 카운터 (204)
config_run__(__tick++)       // IEC 61131-3 로직 실행 (205)
updateBuffersOut_MB()        // 출력 이미지 → 슬레이브 쓰기 버퍼 (206)
unlock(bufferLock)           // 207
updateBuffersOut()           // 출력 이미지 → 하드웨어 (209)
sleep_until(common_ticktime__) // 주기 유지 (223)
RecordCycletimeLatency()     // 사이클/지연 통계 (236)
```

프로토콜 서버(Modbus/TCP, EtherNet/IP, DNP3, S7(snap7))와 영속 저장 스레드는 같은 이미지 테이블을 `bufferLock`으로 공유한다. 종료 시 `disableOutputs()` → `updateBuffersOut()` → `finalizeHardware()` 순으로 출력을 0으로 내린다(`main.cpp:257-259`, `utils.cpp:164-`).

### 1.2 핵심 추상화

| 추상화 | 위치 | 계약 |
|---|---|---|
| I/O 이미지 테이블 | `core/ladder.h:61-86` | `IEC_BOOL *bool_input[BUFFER_SIZE][8]`, `bool_output`, `byte_*`, `int_*`, `dint_*`, `lint_*`, `int_memory/dint_memory/lint_memory`, `special_functions`. 값이 아니라 **포인터 배열**이며 `glueVars()`가 사용자 변수 주소에 연결한다. `BUFFER_SIZE 1024`(`ladder.h:35`) |
| 버퍼 락 | `ladder.h:89` | `extern pthread_mutex_t bufferLock` — 모든 계층이 지켜야 하는 유일한 동기화 규약 |
| 하드웨어 계층 | `ladder.h:107-110` | `void initializeHardware(); void finalizeHardware(); void updateBuffersIn(); void updateBuffersOut();` 링크 타임 계약(가상 함수·테이블 없음) |
| 로직 계약(컴파일러 생성물) | `ladder.h:99-104` | `config_init__()`, `config_run__(unsigned long tick)`, `glueVars()`, `updateTime()` |
| 프로토콜 서버 | `ladder.h:130,146-147,151,154,164` | `startServer(uint16_t port, int protocol_type)`, `processModbusMessage(buf,size)`, `processEnipMessage`, `processPCCCMessage`, `dnp3StartServer(port)`. 프로토콜 식별은 `#define MODBUS_PROTOCOL 0 / DNP3_PROTOCOL 1 / ENIP_PROTOCOL 2`(`ladder.h:29-31`) |
| Modbus 마스터(원격 I/O) | `ladder.h:157-160`, `core/modbus_master.cpp:62` | `struct MB_device`(TCP/RTU 파라미터 + discrete_inputs/coils/input_registers/holding 주소 범위), `initializeMB()`, `querySlaveDevices()` 스레드(`modbus_master.cpp:354`), `updateBuffersIn_MB/Out_MB`(693/711) |
| 영속 저장 | `ladder.h:167-168`, `core/persistent_storage.cpp:35,42,81,167` | `persistent.file`에 메모리 영역 저장, XOR 체크섬으로 변경 감지 |
| 디버그/강제 | `core/debug.cpp:48,68` | `force_var(idx, forced, val)`, `set_trace(...)` — Modbus 사용자 정의 FC 0x41–0x45로 노출(`core/modbus.cpp:55-59`) |
| 런타임 제어 RPC | `core/interactive_server.cpp:240`(`processCommand`) | `quit()`(252), `start_modbus(`(294), `stop_modbus()`(315), `start_snap7()`(329), `start_dnp3(`(363), `start_enip(`(398), `start_pstorage(`(433), `runtime_logs()`(462), `exec_time()`(470) |
| 웹 측 런타임 래퍼 | `OP/openplc.py:73`(`class runtime`) | `start_runtime`, `stop_runtime`, `compile_program(st_file)`, `start_modbus(port)` 등 |
| REST 콜백 | `OP/restapi.py:29-30,88-96,254-256` | `register_callback_get/post(Callable[[str, dict], dict])`; `/api/<command>`를 `webserver.py:51`의 `restapi_callback_get`으로 위임(start-plc, stop-plc, runtime-logs, compilation-status, status, ping / upload-file) |

구현체:

- 하드웨어 계층(`core/hardware_layers/`): `blank.cpp`, `raspberrypi.cpp`, `raspberrypi_old.cpp`, `pixtend.cpp`, `pixtend2s.cpp`, `pixtend2l.cpp`, `unipi.cpp`, `neuron.cpp`, `fischertechnik.cpp`, `sequent.cpp`, `sl_rp4.cpp`, `PiPLC.cpp`, `synergy.cpp`, `opi_zero2.cpp`, `simulink.cpp`(UDP로 시뮬레이터와 교환, `simulink.cpp:70-185`), `psm.cpp`(Python 하드웨어 모듈 브리지). 예: `blank.cpp:39,47,56,78`, `raspberrypi.cpp:68,100,109,127`.
- 프로토콜 서버: `modbus.cpp`(FC 1,2,3,4,5,6,15,16 — `modbus.cpp:46-54`), `enip.cpp`+`pccc.cpp`, `dnp3.cpp`(opendnp3 `ICommandHandler` 구현 `CommandCallback`, `dnp3.cpp:88`), S7은 snap7 래퍼(`main.cpp:124`의 `initializeSnap7()`; 래퍼 소스는 `utils/snap7_src` — 내용 미확인).
- EtherCAT은 `#ifdef _ethercat_src`로 감싼 콜백 전달 방식(`main.cpp:102-105,187-202`).

### 1.3 확장점

- 새 하드웨어: `core/hardware_layers/<이름>.cpp`에 4개 함수를 구현하고, `scripts/change_hardware_layer.sh`가 선택된 파일을 `hardware_layer.cpp`로 복사(`change_hardware_layer.sh:15-141`)한 뒤 재컴파일. 동적 로딩 없음.
- 코드 없이 추가: PSM(Python SubModule). `core/psm/main.py:34,38,48`의 `hardware_init()/update_inputs()/update_outputs()`를 작성하고 `psm.get_var/set_var`(`core/psm/psm.py:78,99`)로 이미지에 접근. 웹 UI의 `/hardware` 화면이 폼 입력(`custom_layer_code`)을 그대로 `./core/psm/main.py`에 기록한다(`webserver.py:1955-1957` 부근, 라우트 `webserver.py:1879`).
- 새 프로토콜 서버: `ladder.h`에 프로토콜 상수 추가, `server.cpp`의 `processMessage` 분기(`server.cpp:181-`)와 `interactive_server.cpp`의 start/stop 명령, `openplc.py`의 RPC 래퍼, 웹 설정 화면을 모두 수정해야 한다. 등록 테이블이 없으므로 5곳 이상을 손으로 맞춰야 한다.

### 1.4 데이터 모델

- 포인트 = IEC 주소로 인덱싱된 고정 크기 배열 슬롯. 태그 이름·단위·스케일은 런타임에 없다(모니터링용 이름은 웹이 ST 파일을 파싱해 얻음: `OP/monitoring.py:16`).
- Modbus 주소 매핑이 곧 외부 데이터 모델: 코일/이산입력 8192, 홀딩 8192, 입력레지스터 1024(`modbus.cpp:34-37`), 홀딩 1024–2047=%MW, 2048–4095=%MD, 4096–8191=%ML(`modbus.cpp:39-44`).
- 품질(quality): **없음**. 타임스탬프: **없음**(값에 부착되는 시각 없음). 시각은 special function 슬롯으로만 제공(`utils.cpp:200-`).
- 알람/이벤트 모델: **없음**. 명령 모델: **없음**(쓰기는 곧 이미지 테이블 직접 변경). DNP3만 Select/Operate 인터페이스를 갖지만 `Select`는 무조건 `SUCCESS`(`dnp3.cpp:92-94,120-122`).
- 설정/프로그램/사용자/슬레이브 장치는 SQLite `openplc.db`(테이블 `Users`, `Programs`, `Settings`, `Slave_dev` — 스키마 직접 조회).

### 1.5 보안 관련 추상화

| 항목 | 상태 | 근거 |
|---|---|---|
| 웹 UI 인증 | flask_login 세션. 각 라우트 첫머리에서 `flask_login.current_user.is_authenticated` 수동 검사 | `webserver.py:645,660,...` (라우트 642~2582) |
| 비밀번호 저장 | **평문**. `Users.password TEXT`를 `row[1] == password`로 비교 | `webserver.py:621`, `webserver.py:576` |
| 하드코딩 기본 계정 | **있음**: 저장소에 포함된 `openplc.db`의 `Users` 행 `('OpenPLC User','openplc','openplc')` | `OP/openplc.db` 조회 결과 |
| 역할/권한 | 웹 UI: **없음**(로그인한 모든 사용자가 사용자 추가·프로그램 업로드·하드웨어 코드 변경 가능). REST: `role` 컬럼은 있으나 `# TODO implement roles` | `restapi.py:55-59` |
| REST 인증 | JWT(`flask_jwt_extended`), PBKDF2-SHA256 600000회 + PEPPER, 로그아웃 블록리스트는 프로세스 메모리 `set` | `restapi.py:43,62,65,220-236,245-251` |
| REST 최초 사용자 | 사용자가 0명이면 인증 없이 `/api/create-user` 허용, `role`은 요청 본문에서 그대로 수용 | `restapi.py:98-128` (114) |
| REST 사용자 관리 권한 | 인증된 아무 사용자나 타 사용자 삭제 가능, 비밀번호 변경은 대상의 기존 비밀번호 필요 | `restapi.py:173-216` |
| 비밀 관리 | `.env`에 JWT_SECRET_KEY/PEPPER 자동 생성, chmod 600. 기본 `FLASK_ENV=development`. 웹 세션 키는 기동 시 `os.urandom` | `config.py:31-42`, `webserver.py:32` |
| 로그에 민감정보 | 비밀번호 해시를 DEBUG 로그로 출력, 로그 레벨 DEBUG 고정 | `restapi.py:23,68` |
| 세션 수명 | 5분 슬라이딩 | `webserver.py:590` |
| 통신 암호화 | 웹 UI 8080은 평문 HTTP, REST 8443만 TLS(자체 서명). 필드 프로토콜은 전부 평문 | `webserver.py:2695,2747` |
| 필드 프로토콜 접근통제 | **없음**. Modbus/ENIP 리슨 소켓은 `INADDR_ANY`, DNP3는 `"0.0.0.0"`. `processModbusMessage`는 함수코드로만 분기(1107~) | `server.cpp:112`, `dnp3.cpp:431`, `modbus.cpp:1107-` |
| 쓰기(제어) 통제 | **없음**. FC5/6/15/16이 범위 검사 후 이미지에 직접 기록. 변수 강제(FC 0x42)도 같은 포트로 노출 | `modbus.cpp:516,635,668,727,825`, `modbus.cpp:56` |
| 기본 노출 | 기본 설정에서 Modbus 502, ENIP 44818, snap7 활성 | `openplc.db` `Settings` 행 |
| 런타임 제어 RPC | 인증 없음, 루프백 바인드로만 보호 | `interactive_server.cpp:180` |
| 감사 로그 | 전용 구조 **미발견**. 런타임 로그는 1MB 메모리 버퍼(`ladder.h:123`)뿐이며 "누가" 정보 없음 |
| 코드 실행 표면 | 인증된 웹 사용자가 ST 업로드→네이티브 컴파일, Python 하드웨어 코드 직접 편집 가능(설계상 기능) | `openplc.py:113-204`, `webserver.py:1879-` |
| CSRF 방어 | 미확인(관련 코드 grep에서 미발견) |
| 입력 이스케이프 | HTML 이스케이프 유틸 존재 | `webserver.py:2613-2640` |

### 1.6 추상화가 암시하는 요구사항

- 포인터 배열 이미지 테이블 + 단일 락 → "입력 스냅샷 → 로직 → 출력 반영"의 원자적 스캔과, 통신 스레드가 스캔 중간 상태를 보지 못하게 하는 일관성 요구.
- `SCHED_FIFO` 우선순위 30, `mlockall`(`main.cpp:134-146`), 사이클/지연 통계(`main.cpp:214-236`) → 결정적 주기 실행과 지터 계측 요구.
- 4함수 하드웨어 계층 + PSM → 로직을 바꾸지 않고 보드를 교체하는 이식성, 비 C 개발자도 I/O를 붙일 수 있어야 한다는 요구.
- `disableOutputs()` 종료 시퀀스 → 정지 시 출력 안전 상태(fail-safe) 요구.
- `persistent.file` → 재기동 후 유지 변수(retain) 요구.
- interactive server의 프로토콜별 start/stop → 프로토콜 서비스를 런타임 재시작 없이 개별 on/off 해야 한다는 운영 요구.
- Modbus 마스터의 `isConnected`, polling/timeout 설정(`Slave_polling`, `Slave_timeout`) → 원격 I/O 통신 두절 감지·재접속 요구.
- REST가 뒤늦게 PBKDF2/JWT/TLS로 따로 만들어진 점, `role` TODO → 인증·역할 요구가 초기 설계에 없었고 후행 추가 중임을 보여 준다(= 요구사항 문서에는 처음부터 포함해야 할 항목).

---

## 2. FUXA (웹 SCADA/HMI)

### 2.1 최상위 구조와 데이터 흐름

단일 Node.js 프로세스. `FX/main.js`가 Express + socket.io 서버를 만들고(`main.js:267-268,599`), `FX/runtime/index.js`가 하위 매니저(devices, alarms, scripts, daq storage, notificator, scheduler, plugins, users, project)를 초기화한다. REST는 `FX/api/index.js`가 기능별 서브 앱(projects, users, alarms, daq, scripts, command, plugins, diagnose, apikeys 등)을 조립한다(`api/index.js:71-97`).

데이터 흐름:

```
현장 장치 ──(드라이버 polling)──> 드라이버 내부 태그 캐시(value, timestamp, changed)
   └ events.emit('device-value:changed')            (modbus/index.js:738)
        ├ runtime: socket.io 'device-values' 로 브라우저에 푸시 (runtime/index.js:128,640-663)
        ├ addDaq 콜백 → DAQ 저장소(SQLite/InfluxDB/TDengine/QuestDB) (modbus/index.js:721, devices/index.js:188)
        └ 알람 매니저/스크립트가 태그 값 조회
제어: 브라우저 socket 'device-values' {cmd:'set'} (runtime/index.js:204-217)
      또는 REST POST /api/setTagValue (api/command/index.js:120)
        → devices.setDeviceValue → Device.setValue → comm.setValue (devices/index.js:452, device.js:318-325)
```

### 2.2 핵심 추상화

**(a) 장치 래퍼 + 드라이버 덕 타이핑 계약** — `FX/runtime/devices/device.js`

- `function Device(data, runtime)`(`device.js:36`): 상태 기계 INIT/IDLE/POLLING(`device.js:672-676`), 명령 START/STOP(`681-685`), 연결 상태 OFF/WARNING/ON(`690-694`).
- 드라이버 객체 `comm`의 필수 메서드가 주석으로 명시된다(`device.js:51-53`): `connect, disconnect, isConnected, polling, init, load, getValue, getValues, getStatus, setValue, bindAddDaq, getTagProperty`. 정식 인터페이스 정의는 없고 참조 구현이 `FX/runtime/devices/template/index.js`다: `init`(16), `connect`(24), `disconnect`(35), `polling`(44), `load`(52), `getValues`(59), `getValue`(66), `getStatus`(73), `getTagProperty`(80), `setValue(tagid, value)`(87), `isConnected`(94), `bindAddDaq`(101), `lastReadTimestamp`(109), `getTagDaqSettings/setTagDaqSettings`(117/125), 팩토리 `create(data, logger, events, manager)`(133).
- 선택 메서드: `browse`(OPC UA, BACnet, MQTT, ODBC, Redis, SNMP, GenericEthernetIP만 — `device.js:330-374`), `readAttribute`(OPC UA만, 386-398), `bindGetProperty`(자격증명 조회, OPC UA/MQTT/ODBC/SNMP만, 440-444).
- 드라이버 선택은 `data.type` 문자열에 대한 if-else 체인(`device.js:56-142`)이며 레지스트리가 아니다.
- 구현체(`DeviceEnum`, `device.js:646-667`): SiemensS7, OPCUA, ModbusRTU, ModbusTCP, BACnet, WebAPI, MQTTclient, EthernetIP, GenericEthernetIP, OmronEthernetIP, FuxaServer(내부 가상 장치), ODBC, ADSclient, GPIO, internal, WebCam, MELSEC, REDIS, SNMP. 디렉터리: `FX/runtime/devices/{s7,opcua,modbus,bacnet,httprequest,mqtt,ethernetip,genericethernetip,omron-ethernetip,fuxaserver,odbc,adsclient,gpio,webcam,melsec,redis,snmp}`.
- 공통 감독 로직: 5초 주기 상태 점검(`device.js:29,161-163`), 연결 실패 4회 초과 시 강제 disconnect(`212-215`), 마지막 읽기 시각이 폴링 주기의 2배/5배를 넘으면 WARNING/OFF(`227-235`), 재접속 후 DAQ에서 값 복원(`restoreValues`, `485-507`).

**(b) 장치 매니저** — `FX/runtime/devices/index.js`: `loadDevice`(157), `getDevicesValues`(213), `setDeviceValue`(452), `setTagValue`, `enableDevice`(341), `browseDevice`(474) 등(`570-600` exports).

**(c) 이벤트 버스** — Node `EventEmitter`(`FX/runtime/events.js`). 내부 이벤트 `device-value:changed`, `device-status:changed`, `alarms-status:changed`, `project-device:change`, `tag-change:subscription`, `script-console`(`runtime/index.js:127-132`). 브라우저 채널 이름은 `IoEventTypes` 열거(`events.js`): `device-status`, `device-values`, `device-browse`, `daq-query`, `alarms-status`, `heartbeat` 등.

**(d) DAQ 저장소 추상화** — `FX/runtime/storage/daqstorage.js`: `addDaqNode(_id, fncgetprop)`(43)이 설정에 따라 SQLite/InfluxDB/TDengine/QuestDB 백엔드 생성(46-63), `getNodeValues(tagid, fromts, tots)`(70), `checkRetention()`(135), `DaqStoreTypeEnum`(191-196).

**(e) 알람** — `FX/runtime/alarms/index.js`: `AlarmsManager`(14), `Alarm(name, type, subprop, tagprop)`(572), `check(time, dt, value, displayValue)`(590), `setAlarmAck(alarmName, userId, permission)`(190). 저장은 `alarmstorage.js`의 `alarms`(현재)·`chronicle`(이력) 테이블(56-57).

**(f) 스크립트** — `FX/runtime/scripts/msm.js`: 사용자 스크립트를 문자열로 조합해 `Module._compile`로 로드(`msm.js:142-147`), 시스템 함수는 `global`에 주입(`msm.js:30-32`). 권한 판정 `isAuthorised(_script, permission)`(`scripts/index.js:93`).

**(g) 플러그인(드라이버 npm 패키지)** — `FX/runtime/plugins/index.js`: 런타임에 npm 패키지를 설치/제거(`plugins/index.js:154,186`)하고 `device.loadPlugin(type, module)`로 드라이버 모듈 교체(`device.js:580-614`).

### 2.3 확장점

새 프로토콜 드라이버를 추가하려면:

1. `FX/runtime/devices/<이름>/index.js`에 template의 메서드 집합과 `create()` 구현.
2. `device.js`에 `require`, `DeviceEnum` 항목, 생성 분기(56-142), `loadPlugin` 분기(580-614) 추가.
3. browse/보안 속성이 필요하면 `browse`·`bindGetProperty` 분기에도 추가.
4. 클라이언트 `client/src/app/_models/device.ts`의 `DeviceType` 및 설정 UI 추가.

중앙 레지스트리가 없어 한 드라이버 추가에 if-else 4곳 이상을 수정해야 한다. DAQ 백엔드 추가는 `daqstorage.js:46-63` 분기에 추가. 로직 확장은 서버 스크립트(JS)와 Node-RED 연동(`settings.default.js:136-143`).

### 2.4 데이터 모델

- Device(`client/src/app/_models/device.ts:19`): `id, name, enabled, property, type, polling, tags`.
- Tag(`device.ts:60`): `id, name, label, value, type, memaddress, address, divisor, access, options, format`, 그리고 `daq: TagDaq{enabled, interval, changed, restored}`(`device.ts:226`), `deadband`, `scale: TagScale{mode, rawLow, rawHigh, scaledLow, scaledHigh, readExpression, writeExpression}`(`device.ts:1258`).
- 런타임 값: `{id, value, rawValue, timestamp, changed, daq}`(`modbus/index.js:685-726`). **타임스탬프는 서버 수신 시각** `new Date().getTime()`(`modbus/index.js:715`)이며 장치 원천 시각이 아니다(Modbus 드라이버 기준. 다른 드라이버는 미확인).
- 품질(quality): 태그 단위 품질 필드 **미발견**(Modbus 드라이버·device-utils 기준). 장치 단위 연결 상태(OFF/WARNING/ON)만 존재.
- 값 가공: 데드밴드(`device-utils.js:35-36`), 스케일/표현식(`tagValueCompose` 22, `tagRawCalculator` 62), DAQ 저장 조건 "변경 시 또는 주기"(`tagDaqToSave`, `device-utils.js:9-14`).
- 알람: `Alarm{name, type, subproperty, tagproperty, ontime, offtime, acktime, status, userack}`(`alarms/index.js:572-583`), 상태 `'' / N / NF / NA`(`alarms/index.js:712-716`), ack 모드 float/ackactive/ackpassive(719~), 지연 `timedelay`(607).
- 명령: 별도 객체 **없음**. `(deviceId, tagId, value, fnc)` 호출이 전부이며 명령 ID·발행자·결과 기록이 없다.
- 프로젝트 저장: SQLite 테이블 `general, views, devices, devicesSecurity, texts, alarms, recipes, notifications, scripts, reports, locations, arMarkers`, 각 행은 JSON TEXT(`project/prjstorage.js:75-86`).

### 2.5 보안 관련 추상화

| 항목 | 상태 | 근거 |
|---|---|---|
| 인증 기본값 | **꺼짐**. `secureEnabled`가 기본 설정에서 주석 처리 | `settings.default.js:115-119`, `api/jwt-helper.js:6` |
| 꺼져 있을 때 권한 | 모든 요청이 관리자 그룹으로 취급 | `api/index.js:364-379`(else 분기에서 `adminGroups[0]` 반환), `runtime/index.js:42-43,49-50,145` |
| 사용자/비밀번호 | SQLite `users(username, fullname, password, groups, info)`, bcrypt(10) | `users/usrstorage.js:69,165`, `api/auth/index.js:159` |
| 하드코딩 기본 계정 | **있음**: `admin` / `123456`, groups = -1(관리자) | `users/usrstorage.js:119-124` |
| 토큰 | JWT(`x-access-token`), 기본 60분, 비밀키 미설정 시 기동마다 무작위 생성 | `api/jwt-helper.js:8-9,59-85`, `main.js:214-216` |
| 게스트 폴백 | 토큰이 없거나 무효면 오류가 아니라 guest로 진행 | `api/jwt-helper.js:62-72,118-127`, `runtime/index.js:139-141` |
| API 키 | `x-api-key`, enabled/expires 검사. **유효한 API 키는 곧 관리자 권한** | `api/apikeys/verify-api-or-token.js:15-34`, `api/index.js:366-368` |
| 권한 모델 | 두 가지: (1) 비트마스크 그룹(하위 8비트 enabled, 상위 8비트 show), (2) `userRole` 설정 시 역할 이름 목록. 관리자는 -1 또는 255 | `runtime/index.js:774-840`, `api/jwt-helper.js:10,139-147` |
| REST 쓰기 통제 | `/api/setTagValue`는 관리자만 | `api/command/index.js:120-124` |
| socket 쓰기 통제 | `secureEnabled`이면 "인증된(비 guest) 소켓"인지만 검사 후 `setDeviceValue`. 이 경로에서 **태그/뷰 단위 권한 검사는 확인되지 않음**. `secureOnlyEditor` 설정이면 검사 자체를 건너뜀 | `runtime/index.js:41-46,204-217` |
| 관리 기능 통제 | 프로젝트 저장, 장치 설정, 사용자/역할, 플러그인, 로그, API 키는 관리자 검사 | `api/projects/index.js:94-98,198-202`, `api/users/index.js:78-82`, `api/plugins/index.js:61-65`, `api/diagnose/index.js:62-66`, `api/apikeys/index.js:60-64` |
| 알람 ack | 사용자 ID를 기록(`userack`), 권한은 알람의 tagproperty 기준 | `api/alarms/index.js:89-94`, `alarms/index.js:190-199`, `alarmstorage.js:191-192` |
| 속도 제한 | 로그인/리프레시 5분 100회, 일반 API 5분 1000회 | `api/index.js:55-66` |
| 통신 암호화(웹) | `settings.https`가 있을 때만 HTTPS, 기본은 주석 | `main.js:267-268`, `settings.default.js:109-112` |
| CORS | 기본 허용 목록에 `http://192.168.*`, `http://10.*` | `settings.default.js:84` |
| 장치 측 보안 | OPC UA securityMode/Policy·사용자명, MQTT 사용자명/인증서 디렉터리 지원. 자격증명은 `devicesSecurity` 테이블에 분리 저장. **저장 시 암호화 여부는 미확인**(prjstorage에서 암호화 코드 grep 미발견) | `devices/opcua/index.js:55-59,96`, `devices/mqtt/index.js:58`, `device.js:85`, `project/prjstorage.js:78` |
| 스크립트 실행 격리 | **없음**. 사용자 스크립트를 Node 모듈로 컴파일해 서버 프로세스 권한으로 실행. test 모드 실행은 관리자만 | `scripts/msm.js:142-147`, `api/scripts/index.js:54-61` |
| 런타임 패키지 설치 | 관리자 API로 npm 패키지 설치 가능 | `api/plugins/index.js:61`, `plugins/index.js:154` |
| 감사 로그 | 전용 구조 **미발견**. HTTP 접근 로그(morgan, `api/index.js:43-46`), 알람 chronicle의 `userack`, 무권한 쓰기 시도 경고 로그(`runtime/index.js:213`)가 전부. 성공한 쓰기의 "누가/무엇을/언제" 기록은 확인되지 않음 |

### 2.6 추상화가 암시하는 요구사항

- 드라이버 메서드 집합(`connect/polling/getValue/setValue/isConnected/lastReadTimestamp`) → 프로토콜 무관하게 "주기 수집·현재값 캐시·쓰기·연결 감시"를 동일 방식으로 다뤄야 한다는 요구.
- Device 상태 기계 + 재시도·강제 disconnect → 장치 두절에서 자동 복구해야 하고 한 장치의 장애가 다른 장치를 막지 않아야 한다는 요구.
- `lastReadTimestamp` 기반 OFF/WARNING/ON → 값의 신선도(stale)를 운영자에게 보여야 한다는 요구(다만 태그 품질로는 승격되지 않음).
- `bindAddDaq` + 저장소 4종 → 수집과 저장의 분리, 시계열 DB 교체 가능성, 보존기간(retention) 요구.
- 데드밴드·"변경 시/주기" 저장 → 저장량·네트워크 부하 억제 요구.
- `restoreValues` → 재기동/재접속 후 설정값(setpoint) 복원 요구.
- `browse`/`getSupportedProperty` → 엔지니어링 시 태그 자동 탐색 요구.
- 알람 상태/ack 모드/`userack` → 알람 수명주기와 확인자 기록 요구.
- show/enabled 2축 권한 → "보이는 것"과 "조작 가능한 것"을 따로 통제해야 한다는 HMI 요구.
- `secureEnabled` 옵션화, guest 폴백, 기본 계정 → 데모 편의가 기본값을 지배. 요구사항에는 "보안 기본 활성(secure by default)"을 명시해야 함을 시사.

---

## 3. Rapid SCADA v6 (scada-v6)

### 3.1 최상위 구조와 데이터 흐름

서비스 4종 + 관리 도구로 분리된다.

- **ScadaComm(Communicator)**: 현장 통신. `RS/ScadaComm/ScadaComm/ScadaCommEngine/`(엔진), `ScadaCommCommon/`(드라이버 SDK), `RS/ScadaComm/OpenDrivers*/`(드라이버).
- **ScadaServer**: 현재값·계산·이벤트·아카이브·명령 중계·사용자 검증. `RS/ScadaServer/ScadaServer/ScadaServerEngine/`, 모듈 SDK `ScadaServerCommon/`, 모듈 `RS/ScadaServer/OpenModules/`.
- **ScadaWeb(Webstation)**: ASP.NET Core UI. 플러그인 SDK `RS/ScadaWeb/ScadaWeb/ScadaWebCommon/`, 플러그인 `RS/ScadaWeb/OpenPlugins/`.
- **ScadaAgent**: 원격 배포·서비스 제어(`RS/ScadaAgent/ScadaAgent/ScadaAgentEngine/AgentListener.cs:47`).
- 공통: `RS/ScadaCommon/ScadaCommon/`(데이터 모델, 자체 TCP 프로토콜, 클라이언트, 저장소 추상화).

서비스 간 통신은 자체 바이너리 TCP 프로토콜이다(기본 포트 10000, `ScadaCommon/Server/ListenerOptions.cs:44`). 기능 코드: 세션(0x0001–0x0005), 파일(0x0101–0x0104), 데이터(`GetCurrentData 0x0201`, `WriteChannelData 0x0208`), 이벤트(`WriteEvent 0x0303`, `AckEvent 0x0304`), 명령(`SendCommand 0x0401`, `GetCommand 0x0402`), 에이전트(0x0501–0x0504) — `ScadaCommon/Protocol/FunctionID.cs:38-67`.

데이터 흐름:

```
[Communicator] CommLine.LineCycle (CommLine.cs:297)
  channel.BeforeSession → DeviceLogic.Session() → channel.AfterSession   (363-386)
  실패 시 InvalidateData()                                              (373)
  TransferDeviceData: GetCurrentData + DequeueSlice                     (582-591)
    → CoreLogic.EnqueueCurrentData → DataSourceHolder.WriteCurrentData  (Comm CoreLogic.cs:920-925)
    → DataSourceLogic 구현(ScadaServerDSL 등)이 Server로 전송
[Server] CoreLogic.Execute (Server CoreLogic.cs:517)
  WriteCurrentData(1319) → 입력 수식/한계 판정 UpdateCnlStatus(803) → GenerateEvent(949)
  → CurrentArchiveLogic / HistoricalArchiveLogic / EventArchiveLogic
  → 모듈 훅 OnCurrentDataProcessing/Processed, OnEvent
[Web] ScadaClient로 Server 조회 → 플러그인이 화면 구성
명령: Web MainApiController.SendCommand (MainApiController.cs:573)
  → Server CoreLogic.SendCommand (1568): 사용자/채널/권한 검사
  → ProcessCommands (678): 모듈 OnCommand → 이벤트 생성 → listener.EnqueueCommand
  → Communicator가 GetCommand로 수신 → CoreLogic.EnqueueCommand (Comm 953)
  → CommLine.EnqueueCommand (799) → DeviceLogic.SendCommand (DeviceLogic.cs:437)
```

Communicator에는 파일 기반 명령 입력도 있다: 명령 디렉터리의 `*.dat`를 읽어 큐에 넣고 삭제(`ScadaCommEngine/CommandReader.cs:70-99`).

### 3.2 핵심 추상화

**Communicator 드라이버 SDK** (`RS/ScadaComm/ScadaComm/ScadaCommCommon/`)

| 타입 | 위치 | 주요 시그니처 |
|---|---|---|
| `abstract class DriverLogic` | `Drivers/DriverLogic.cs:40` | `abstract string Code`(77); `virtual DataSourceLogic CreateDataSource(ICommContext, DataSourceConfig)`(94); `virtual ChannelLogic CreateChannel(ILineContext, ChannelConfig)`(102); `virtual DeviceLogic CreateDevice(ILineContext, DeviceConfig)`(110); `OnServiceStart()`(118); `OnServiceStop()`(125) |
| `DriverFactory` | `Drivers/DriverFactory.cs:37` | 명명 규약으로 리플렉션 로드: 파일 `<Code>.Logic.dll`(45), 타입 `Scada.Comm.Drivers.<Code>.Logic.<Code>Logic`(46), `Assembly.LoadFrom`(52), `Activator.CreateInstance(type, commContext)`(54) |
| `abstract class DeviceLogic` | `Devices/DeviceLogic.cs:47` | `CanSendCommands`(180), `ConnectionRequired`(185), `DeviceTags/DeviceData/DeviceStats`(232-242); `OnCommLineStart()`(332), `OnConnectionSet()`(346), `CheckBehaviorSupport(ChannelBehavior)`(353), `InitDeviceTags()`(361), `BindDeviceTags(ConfigDataset)`(368), `InitDeviceData()`(397), `InvalidateData()`(405), `GetCurrentData(bool allData)`(413), `Session()`(423), `SendCommand(TeleCommand cmd)`(437), `ReceiveIncomingRequest(Connection, IncomingRequestArgs)`(454), `GetInfo()`(499). 재시도 헬퍼 `RequestNeeded(ref int tryNum)`(259), `FinishRequest()`(267) |
| `abstract class ChannelLogic` | `Channels/ChannelLogic.cs:42` | `MakeReady()`(232), `Start()`(240), `Stop()`(247), `BeforeSession(DeviceLogic)`(254), `AfterSession(DeviceLogic)`(261) |
| `enum ChannelBehavior` | `Channels/ChannelBehavior.cs:32` | `Master, Slave, Mixed` |
| `abstract class Connection` | `Channels/Connection.cs:39` | `abstract bool Connected`(101); `Read(buffer, offset, count, timeout, ...)`(118); 정지 조건 기반 `Read`(134); `ReadLines`(152); `Write`(187); `WriteLine`(201). 프레임 로그 문자열 생성 포함(215-260) |
| `abstract class DataSourceLogic` | `DataSources/DataSourceLogic.cs:41` | `MakeReady/Start/Close/Refresh`(110-132), `ReadConfigDatabase(out ConfigDatabase)`(139), `WriteCurrentData(DeviceSlice)`(148), `WriteHistoricalData(DeviceSlice)`(155), `WriteEvent(DeviceEvent)`(162) |
| `interface ICommContext` | `Drivers/ICommContext.cs:39` | `AppConfig, AppDirs, Storage, Log, ConfigDatabase, SharedData, GetCommLines(), GetDevice(...), SendCommand(TeleCommand, string source)` |
| `interface ILineContext` | `Drivers/ILineContext.cs:40` | `SelectDevices()`(76), `GetDeviceByAddress(...)`(91,96), `EnqueueCommand(TeleCommand)`(101) |

구현체(전수 grep 결과):

- 장치(DeviceLogic): `DevModbusLogic`(DrvModbus), `DevOpcUaLogic`, `DevMqttClientLogic`, `DevMqttPublisherLogic`, `DevDbImportLogic`, `DevSimulatorLogic`, `DevTesterLogic`, OpenDrivers2의 `DevSnmpLogic`, `DevCsvReaderLogic`, `DevRsClientLogic`, `DevEmailLogic`, `DevSmsLogic`, `DevSmsParserLogic`, `DevHttpNotifLogic`. (DrvEnronModbus는 DriverLogic만 확인, 장치 클래스 위치 미확인.)
- 채널(ChannelLogic) — DrvCnlBasic: `SerialPortChannelLogic`, `TcpClientChannelLogic`, `TcpServerChannelLogic`, `UdpChannelLogic`; DrvCnlMqtt: `MqttClientChannelLogic`. 연결: `SerialPortConnection`, `TcpConnection`, `UdpConnection`.
- 데이터 소스(DataSourceLogic): `ScadaServerDSL`(DrvDsScadaServer), `MqttDSL`(DrvDsMqtt), `OpcUaServerDSL`(DrvDsOpcUaServer).

핵심 설계: **물리 채널(직렬/TCP/UDP/MQTT)과 장치 프로토콜(Modbus 등)이 직교**한다. 통신 라인 하나가 채널 1개와 장치 N개를 가진다. 알림 수단(Email/SMS/HTTP)도 "명령을 받는 장치"로 모델링된다.

**Server 모듈 SDK** (`RS/ScadaServer/ScadaServer/ScadaServerCommon/`)

| 타입 | 위치 | 주요 시그니처 |
|---|---|---|
| `abstract class ModuleLogic` | `Modules/ModuleLogic.cs:40` | `abstract string Code`(77); `ModulePurposes`(93); `CreateArchive(IArchiveContext, ArchiveConfig, ...)`(105); `OnServiceStart/Ready/Stop`(114-129); `OnIteration()`(136); `OnCurrentDataProcessing(Slice)`(144); `OnCurrentDataProcessed(Slice)`(152); `OnHistoricalDataProcessing/Processed(int archiveMask, Slice)`(159,166); `OnEvent(int archiveMask, Event)`(173); `OnEventAck(EventAck)`(180); `OnCommand(TeleCommand, CommandResult)`(187); `ValidateUser(string username, string password)`(194); `FindUser(int userID)`(202) |
| `enum ModulePurposes` | `Modules/ModulePurposes.cs:35` | `Logic = 1, Archive = 2, Auth = 4` |
| `interface IServerContext` | `Modules/IServerContext.cs:43` | `GetCurrentData(int cnlNum)`(94), `GetTrends(...)`(114), `GetEvents(...)`(139), `WriteCurrentData(...)`(144,149), `WriteHistoricalData(...)`(154), `WriteEvent(...)`(159), `SendCommand(TeleCommand)`(164), `CommandResult SendCommand(TeleCommand, WriteCommandFlags)`(169) |
| `abstract class ArchiveLogic` | `Archives/ArchiveLogic.cs:42` | `MakeReady()`(130), `Close()`(137), `DeleteOutdatedData()`(144), `GetLastWriteTime()`(151) |
| `HistoricalArchiveLogic` | `Archives/HistoricalArchiveLogic.cs:39` | `GetTrends`(134), `GetTrend`(139), `GetTimestamps`(144), `GetSlice`(149), `GetCnlData`(154), `ProcessData(ICurrentData)`(160), `AcceptData(ref DateTime)`(166), `BeginUpdate/UpdateData/EndUpdate`(181-191) |
| `CurrentArchiveLogic` | `Archives/CurrentArchiveLogic.cs:35` | `ReadData(ICurrentData, out bool completed)`(55), `WriteData`(60), `ProcessData`(66) |
| `EventArchiveLogic` | `Archives/EventArchiveLogic.cs:39` | `GetEventByID`(59), `GetEvents(TimeRange, DataFilter)`(64), `WriteEvent(Event)`(69), `AckEvent(EventAck)`(74) |

구현체: `ModArcBasicLogic`(BasicCAL/BasicHAL/BasicEAL, 파일), `ModArcPostgreSqlLogic`(PostgreCAL/HAL/EAL), `ModArcInfluxDbLogic`(InfluxHAL), `ModDbExportLogic`, `ModDiffCalculatorLogic`, `ModActiveDirectoryLogic`(인증 모듈).

**Web 플러그인 SDK**: `abstract class PluginLogic`(`RS/ScadaWeb/ScadaWeb/ScadaWebCommon/Plugins/PluginLogic.cs:41`) — `ViewSpecs`(109), `AddFilters(FilterCollection)`(156), `AddServices(IServiceCollection)`(163), `OnUserLogin(UserLoginArgs)`(184), `OnUserLogout`(191), `GetUserMenuItems(User, UserRights)`(198), `GetUserReports`(206). 구현체: `PlgMainLogic`, `PlgChartLogic`, `PlgSchemeLogic`, `PlgSchBasicCompLogic`, `PlgWebPageLogic`, `PlgMimicLogic`, `PlgMimicEditorLogic`, `PlgMimBasicCompLogic`.

**저장소 추상화**: `IStorage` / `abstract class StorageLogic`(`RS/ScadaCommon/ScadaCommon/Storages/StorageLogic.cs:39`), 구현 `FileStorageLogic`(`RS/ScadaCommon/FileStorage/FileStorageLogic.cs:18`), `PostgreSqlStorageLogic`(`RS/ScadaCommon/PostgreSqlStorage/PostgreSqlStorageLogic.cs:19`).

**리스너 베이스**: `ListenerBase`(`RS/ScadaCommon/ScadaCommon/Server/ListenerBase.cs`) — 세션·로그인·파일 전송을 공통 구현하고 `ValidateUser`(1020), `ProcessCustomRequest`(1058), `AcceptUpload`(1095) 등을 가상 메서드로 노출. 파생: `ServerListener`(`ScadaServerEngine/ServerListener.cs:48`), `AgentListener`.

### 3.3 확장점

- 새 프로토콜: `Drv<X>.Logic.dll`에 `Drv<X>Logic : DriverLogic`(Code 반환, `CreateDevice` 재정의)와 `Dev<X>Logic : DeviceLogic`(`InitDeviceTags`, `Session`, 필요 시 `SendCommand`, 슬레이브면 `ReceiveIncomingRequest`). 네임스페이스·파일명 규약 필수(`DriverFactory.cs:45-46`). 설정 UI는 별도 `.View` 어셈블리.
- 새 물리 채널: `ChannelLogic` + `Connection` 파생, 드라이버의 `CreateChannel` 재정의.
- 새 상위 연계(데이터 반출): `DataSourceLogic` 파생, `CreateDataSource` 재정의.
- 새 아카이브 백엔드: `ModuleLogic.CreateArchive`에서 Current/Historical/Event ArchiveLogic 파생 반환.
- 새 인증원: `ModulePurposes.Auth` 모듈이 `ValidateUser`/`FindUser` 재정의(기본 DB 검증 실패 시 모듈로 위임 — `Server CoreLogic.cs:1175-1178`).
- 계산 로직: 채널 수식과 스크립트를 Roslyn으로 컴파일(`ScadaServerEngine/Calculator.cs:266-270`).
- 웹 화면·감사: `PluginLogic` 파생, 감사 기록은 `IAuditPlugin.WriteToAuditLog(AuditLogEntry)`(`ScadaWebCommon/Audit/IAuditPlugin.cs:32,42`).

### 3.4 데이터 모델

- 채널(포인트) 엔티티 `Cnl`(`RS/ScadaCommon/ScadaCommon/Data/Entities/Cnl.cs:39-79`): `CnlNum, Active, Name, Code, DataTypeID, DataLen, CnlTypeID, ObjNum, DeviceNum, TagNum, TagCode, FormulaEnabled, InFormula, OutFormula, FormatID, OutFormatID, QuantityID, UnitID, LimID, ArchiveMask, EventMask`.
- 채널 유형(`Data/Const/CnlTypeID.cs:37-57`): `Input=1, InputOutput=2, Calculated=3, CalculatedOutput=4, Output=5`. 명령은 출력 유형 채널에만 허용.
- 값 `struct CnlData { double Val; int Stat; }`(`Data/Models/CnlData.cs:36,78,83`). **품질은 `Stat`**(`Data/Const/CnlStatusID.cs:36-48`): `Undefined=0, Defined=1, Archival=2, FormulaError=3, Unreliable=4, LoLo=11, Low=12, Normal=13, High=14, HiHi=15, Good=21, Warning=22, Error=23`. 즉 품질과 한계 상태가 한 필드에 합쳐져 있다.
- **타임스탬프는 슬라이스 단위**: `DeviceSlice{Timestamp, DeviceTags[], CnlData[], DeviceNum, ArchiveMask, DataSentCallback, FailedToSendCallback}`(`ScadaCommCommon/Devices/DeviceSlice.cs:35,76-114`). 개별 값에는 시각이 없다. 드라이버가 과거 시각 슬라이스를 큐에 넣어 이력 보충 가능(`DeviceData.EnqueueSlice`, `DeviceData.cs:869`).
- 드라이버 측 태그 `DeviceTag{Index, Code, Name, DataType, DataLen, Format, Cnl}`(`Devices/DeviceTag.cs:35-133`), 값 버퍼 `DeviceData`(`Set(tagIndex, val, stat)` 466, `Invalidate()` 684, `SetStatusTag(DeviceStatus)` 655, `GetModifiedData()` 815).
- 통신 실패 시 품질 처리: 세션 실패 → `InvalidateData()`(`CommLine.cs:373`) → `DeviceData.Invalidate()`(`DeviceLogic.cs:405-408`).
- 이벤트 `Event`(`Data/Models/Event.cs:35,67-157`): `EventID, Timestamp, Hidden, CnlNum, ObjNum, DeviceNum, PrevCnlVal, PrevCnlStat, CnlVal, CnlStat, Severity, AckRequired, Ack, AckTimestamp, AckUserID, TextFormat, Text, Data`. 확인 `EventAck{EventID, Timestamp, UserID}`(`EventAck.cs:35`).
- 명령 `TeleCommand`(`Data/Models/TeleCommand.cs:40,89-155`): `CommandID, CreationTime, ClientName, UserID, CnlNum, ObjNum, DeviceNum, CmdNum, CmdCode, InitialCmdVal, InitialCmdData, CmdVal, CmdData, RecursionLevel`. 결과 `CommandResult{IsSuccessful, TransmitToClients, ErrorMessage}`(`CommandResult.cs:32`). 수명 제한: 오래된 명령 폐기(`CommLine.cs:455`, `ScadaUtils.CommandLifetime`).
- 아카이브는 비트마스크(`ArchiveMask`)로 채널별 다중 저장소 지정, 이벤트 발생 조건은 `EventMask`.

### 3.5 보안 관련 추상화

| 항목 | 상태 | 근거 |
|---|---|---|
| 사용자 검증 | 서버가 중앙에서 수행: 설정 DB의 `User` 조회 → 해시 비교 → `Enabled && RoleID > Disabled` → 실패 시 인증 모듈 위임 | `ScadaServerEngine/CoreLogic.cs:1138-1178` |
| 비밀번호 해시 | **MD5 3회 조합 + 소스에 하드코딩된 고정 솔트**: `ComputeHash(MD5(password) + MD5(userID) + PasswordSalt)` | `ScadaCommon/ScadaUtils.Crypto.cs:28,76-78,193-200` |
| 역할 | 내장 `Disabled=0, Administrator=1, Dispatcher=2, Guest=3, Application=4` + 사용자 정의 역할, 역할 상속 | `Data/Const/RoleID.cs:36-40`, `Data/Models/RightMatrix.cs:73` |
| 권한 모델 | 역할 × 객체(Obj, 계층) 매트릭스, 권한은 `List/View/Control` 3종 | `Data/Models/Right.cs:34,70-80`, `RightMatrix.cs:39,219` |
| **제어 명령 통제(서버)** | `SendCommand`에서 사용자 존재 → 사용자 활성 → 출력 채널 여부 → `rightMatrix.GetRight(user.RoleID, objNum).Control` 순으로 검사, 실패 사유 로그 | `ScadaServerEngine/CoreLogic.cs:1568-1615` |
| 제어 명령 통제(웹) | `EnableCommands` 전역 스위치, 출력 채널 여부, `userContext.Rights.GetRightByObj(cnl.ObjNum).Control` 검사 후 감사 기록 | `ScadaWeb/OpenPlugins/PlgMain/Controllers/MainApiController.cs:573-600,627` |
| 제어 명령 통제(통신 라인) | 라인별 `CmdEnabled`가 꺼져 있으면 명령 거부·로그 | `ScadaCommEngine/CommLine.cs:799-815` |
| 명령 이력 | 명령 성공 시 사용자명을 포함한 이벤트 생성(채널 EventMask에서 Command 허용 시) | `ScadaServerEngine/CoreLogic.cs:994-1015` |
| 서비스 간 인증 | 세션 ID + 로그인. 로그인 전에는 0x0004 이하 기능만 허용. 서버 리스너는 **Application 역할만** 로그인 허용 | `Protocol/FunctionID.cs:110-113`, `Server/ListenerBase.cs:404-405,460`, `ScadaServerEngine/ServerListener.cs:482-514`(495) |
| 서비스 간 암호화 | **전송 구간 암호화 없음**(`ListenerBase.cs`에 SslStream/TLS 미발견). 비밀번호 필드만 세션 ID와 `SecretKey`로 암호화 | `ListenerBase.cs:566-610`, `Protocol/ProtocolUtils.cs:76-96` |
| 하드코딩 키 | `DefaultSecretKey`, `DefaultIV`가 소스에 상수로 존재하고 인자 없는 `Encrypt(s)/Decrypt(s)`가 이를 사용. 또한 동일한 `SecretKey` 값이 여러 배포 기본 설정 파일에 반복 | `ScadaUtils.Crypto.cs:17,23,100-102,152-154`; 동일 키가 `ScadaAgent/.../ScadaAgentConfig.xml:6,16`, `ScadaWeb/.../ScadaWebConfig.xml`, `ScadaComm/OpenDrivers/DrvDsScadaServer.Shared/Config/DrvDsScadaServer.xml`, `Projects/HelloWorld/.../ScadaServerConfig.xml` 등에 존재 |
| 기본 계정 | 빈 프로젝트 템플릿에 `ScadaWeb`·`ScadaComm`(Application), `admin`(Administrator), `guest`(Guest)가 해시와 함께 포함. 에이전트 설정에 `ScadaAgent`/`ScadaAdmin` 계정. **평문 값은 미확인** | `ScadaAdmin/ScadaAdmin/ScadaAdmin/Templates/EmptyProject.en-GB/BaseXML/User.xml`, `ScadaAgentConfig.xml:12-35` |
| 무차별 대입 방어 | `BruteForceProtector`(분당 실패 한도·차단 시간), 오답 지연 상수, 최대 세션 100, 비활성 60초 후 세션 제거 | `Security/BruteForceProtector.cs:36,51,165,195`, `ListenerBase.cs:59,83,105,149` |
| 웹 인증 | 쿠키 인증, DataProtection 키 저장소 커스텀, **기본 정책이 "인증 필수"**, 정책 `Administrators`/`RequireViewAll`/`Restricted`, 운영 모드 HSTS | `ScadaWeb/ScadaWeb/ScadaWeb/Program.cs:114-140,207` |
| 웹 로그인 옵션 | `RequireCaptcha`, `AllowRememberMe`, `RememberMeExpires`, **`AutoLoginUsername/AutoLoginPassword`**(설정 파일에 계정 저장하는 자동 로그인) | `ScadaWebCommon/Config/LoginOptions.cs:52-72` |
| 감사 로그 | 계약 존재: `IAuditLog`, `IAuditPlugin.WriteToAuditLog`, `AuditLogEntry{ActionTime, Username, ActionType, ActionArgs, ActionResult, Severity, Message}`, 유형 `Login, Logout, OpenView, OpenChart, GenerateReport, SendCommand`. **단 이 저장소에서 `IAuditPlugin` 구현체는 미발견**(디스패치만 존재) | `ScadaWebCommon/Audit/AuditLogEntry.cs:34,64-94`, `AuditActionType.cs:60-65`, `Plugins/PluginHolder.cs:112,566`, `ScadaWeb/Code/AuditLog.cs:35` |
| 원격 배포 통제 | 에이전트가 역할(Administrator/Agent)로 기능 구분, 업로드 경로 접두어 검사, 서비스 제어 | `ScadaAgentEngine/AgentListener.cs:232-255,394,542,738-743` |
| OPC UA 서버 반출 | 사용자명 토큰 검증, 익명 허용 여부 설정, 미신뢰 인증서 자동 수락 옵션 | `DrvDsOpcUaServer.Logic/CustomServer.cs:81,107-119`, `OpcUaServerDSL.cs:74` |
| 파일 기반 명령 | Communicator가 명령 디렉터리의 `*.dat`를 인증 절차 없이 큐에 넣음(파일시스템 권한에 의존) | `ScadaCommEngine/CommandReader.cs:70-99` |
| 동적 코드 | 설정 DB의 수식/스크립트를 서버가 컴파일·실행, 드라이버/모듈 DLL은 명명 규약으로 로드(서명 검증 미확인) | `Calculator.cs:266`, `DriverFactory.cs:52` |

### 3.6 추상화가 암시하는 요구사항

- Driver/Channel/Device 3분할 → 같은 프로토콜을 직렬·TCP·UDP에서 재사용하고, 한 라인에 여러 장치를 순차 폴링해야 한다는 요구(멀티드롭 RS-485 등).
- `ChannelBehavior{Master, Slave, Mixed}` + `ReceiveIncomingRequest` → 폴링뿐 아니라 장치가 먼저 보내는 통신(슬레이브/비요청 보고)도 지원해야 한다는 요구.
- `RequestNeeded/FinishRequest`, `ReqRetries`, `InvalidateData` → 재시도 정책과 "통신 실패 시 값을 미정의로 표시" 요구.
- `CnlData.Stat` → 모든 값은 품질을 수반해야 하며, 한계 초과 상태를 값과 함께 운반해야 한다는 요구.
- `DeviceSlice.Timestamp` + `EnqueueSlice` + `DataSentCallback/FailedToSendCallback` → 장치 시각 기준 이력 보충(backfill)과 전송 실패 인지 요구.
- `DataSourceLogic` 복수 등록 → 수집 계층이 여러 상위 시스템에 동시에 데이터를 공급해야 한다는 요구.
- Current/Historical/Event 아카이브 분리 + `ArchiveMask` + `DeleteOutdatedData` → 저장 목적별 백엔드 선택, 채널별 저장 정책, 보존기간 요구.
- `ModuleLogic`의 `...Processing/...Processed` 훅 쌍, `OnCommand(cmd, result)` → 데이터·명령 파이프라인에 가로채기(검증·차단·변환)가 필요하다는 요구. `CommandResult.TransmitToClients`는 모듈이 명령 전달을 취소할 수 있음을 뜻한다.
- `TeleCommand`의 `UserID/CreationTime/ClientName/CommandID` + 명령 이벤트 → 명령은 식별·추적 가능한 일급 객체여야 하고 유효기간이 있어야 한다는 요구.
- `Right{List, View, Control}` × 객체 계층 → 설비 영역별로 조회와 제어 권한을 분리해야 한다는 요구.
- `ModulePurposes.Auth`, `ModActiveDirectory` → 외부 디렉터리 연동 인증 요구.
- `IStorage`(파일/PostgreSQL) → 설정 저장 위치를 배포 형태에 따라 바꿔야 한다는 요구.
- Agent 분리 → 원격지 인스턴스에 설정 배포·서비스 재시작을 중앙에서 해야 한다는 운영 요구.

---

## 4. 세 프로젝트를 가로지르는 공통 패턴과 차이

### 4.1 공통으로 나타나는 추상화

| 패턴 | OpenPLC | FUXA | Rapid SCADA |
|---|---|---|---|
| 현장 I/O 어댑터 계약 | 4함수 하드웨어 계층(`ladder.h:107-110`) | 드라이버 메서드 집합(`template/index.js`) | `DeviceLogic`/`ChannelLogic`/`Connection` |
| 중앙 값 저장소(프로세스 이미지) | 포인터 배열 + `bufferLock` | 드라이버별 태그 캐시 + 이벤트 버스 | 서버 `CurrentData` + `Slice` |
| 주기 실행 루프 | 스캔 사이클(`main.cpp:160`) | 장치별 `setInterval` 폴링(`device.js:271`) | `CommLine.LineCycle`, 서버 `Execute` |
| 통신 두절 감지·재접속 | `MB_device.isConnected` | 상태 기계 + `lastReadTimestamp` | 세션 실패 → `InvalidateData`, `DeviceStatus` |
| 수집과 저장의 분리 | `persistent.file`(유지 변수만) | `bindAddDaq` + DAQ 백엔드 4종 | `DataSourceLogic` + Archive 3종 |
| 상위 연계용 서버 역할 | Modbus/DNP3/ENIP/S7 서버 | REST/socket.io | 자체 TCP 프로토콜, OPC UA 서버, MQTT |
| 사용자 로직 주입 | IEC 61131-3 컴파일 | JS 스크립트, Node-RED | C# 수식/스크립트(Roslyn), 모듈 |
| 관리 평면과 실행 평면의 분리 | 웹 프로세스 ↔ 런타임 RPC | 없음(단일 프로세스) | Admin/Agent ↔ 서비스 |
| 설정을 데이터로 저장 | SQLite | SQLite(JSON) | XML 테이블/PostgreSQL |

### 4.2 차이

- **확장 방식**: OpenPLC는 컴파일 타임 파일 교체, FUXA는 if-else 체인 + npm 런타임 설치, Rapid SCADA는 명명 규약 기반 DLL 리플렉션. 확장 계약의 명시성은 Rapid SCADA > FUXA(주석·템플릿) > OpenPLC(헤더 프로토타입) 순.
- **품질/타임스탬프**: OpenPLC 둘 다 없음. FUXA는 서버 수신 시각만 있고 태그 품질 미발견. Rapid SCADA는 값마다 상태, 슬라이스마다 시각.
- **명령 모델**: OpenPLC는 메모리 쓰기와 동일. FUXA는 함수 호출(객체 없음). Rapid SCADA는 `TeleCommand` 객체 + 결과 + 이벤트 + 3단계 통제(웹·서버·라인).
- **알람/이벤트**: OpenPLC 없음. FUXA는 알람 상태 기계 + chronicle. Rapid SCADA는 한계 상태가 값 품질에 포함되고 Event/EventAck로 이력화.
- **권한**: OpenPLC는 로그인 여부만. FUXA는 비트마스크/역할의 show·enabled. Rapid SCADA는 역할 × 객체 계층의 List/View/Control.
- **보안 기본값**: OpenPLC는 평문 기본 계정 + 무인증 필드 프로토콜 개방. FUXA는 인증 자체가 기본 꺼짐. Rapid SCADA는 웹 인증 필수가 기본이나 약한 해시와 공유 기본 키.
- **프로세스 격리**: Rapid SCADA만 수집·서버·웹이 분리되어 구간별 계정(Application 역할)을 둔다.

### 4.3 세 프로젝트 공통의 보안 약점(= 요구사항으로 뒤집어야 할 것)

1. 저장소에 포함된 기본 계정 또는 기본 비밀값(OpenPLC `openplc/openplc`, FUXA `admin/123456`, Rapid SCADA 템플릿 계정·공유 `SecretKey`).
2. 필드 프로토콜/서비스 간 통신의 전송 암호화 부재 또는 선택 사항.
3. 사용자 제공 코드를 서버 권한으로 실행하는 기능이 격리 없이 존재.
4. 감사 로그가 없거나(OpenPLC, FUXA) 계약만 있고 구현이 공개 저장소에 없음(Rapid SCADA).

---

## 5. 유추한 요구사항 목록

표기: [O]=OpenPLC, [F]=FUXA, [R]=Rapid SCADA. "근거"는 해당 요구가 존재했음을 보여 주는 추상화, "반례"는 그 요구가 빠져 생긴 약점.

### 5.1 기능 요구사항

| ID | 요구사항 | 근거 |
|---|---|---|
| F-01 | 현장 I/O·프로토콜 접근은 공통 계약(초기화/연결, 주기 읽기, 쓰기, 종료, 연결 상태) 뒤에 숨겨야 하며 로직/상위 계층은 프로토콜을 몰라야 한다 | [O] `core/ladder.h:107-110`; [F] `runtime/devices/template/index.js:16-125`; [R] `ScadaCommCommon/Devices/DeviceLogic.cs:361-454` |
| F-02 | 물리 전송(직렬/TCP/UDP 등)과 장치 프로토콜을 분리해 조합 가능해야 한다 | [R] `Channels/ChannelLogic.cs:42`, `Channels/Connection.cs:39`, DrvCnlBasic 구현체. 반례 [F]: 드라이버가 전송을 내장(Modbus RTU/TCP를 `init` 인자로 구분, `device.js:256-260`) |
| F-03 | 마스터(폴링)와 슬레이브(수신 대기)·비요청 수신 양쪽을 지원해야 한다 | [R] `Channels/ChannelBehavior.cs:32`, `DeviceLogic.cs:454`; [O] 서버(`server.cpp`)와 마스터(`modbus_master.cpp`) 병존 |
| F-04 | 모든 현재값은 단일 공유 저장소(프로세스 이미지/태그 테이블)를 거쳐야 하며 동시 접근 규약이 정의되어야 한다 | [O] `ladder.h:61-89`, `main.cpp:184-207`; [R] `IServerContext.cs:94-109`; [F] `runtime/devices/index.js:213-229` |
| F-05 | 포인트는 주소, 데이터형, 접근 방향(입력/출력/계산), 스케일·포맷, 단위를 메타데이터로 가져야 한다 | [R] `Data/Entities/Cnl.cs:39-79`, `Data/Const/CnlTypeID.cs:37-57`; [F] `client/src/app/_models/device.ts:60-82,1258-1266`. 반례 [O]: 주소 슬롯뿐 |
| F-06 | 값에는 품질 상태가 따라야 하고 통신 실패 시 품질을 무효로 바꿔야 한다 | [R] `CnlData.cs:78-83`, `CnlStatusID.cs:36-48`, `CommLine.cs:373`. 반례 [O] 품질 없음, [F] 태그 품질 미발견 |
| F-07 | 값(또는 값 묶음)에는 타임스탬프가 있어야 하고, 가능하면 원천 시각과 과거 데이터 보충을 지원해야 한다 | [R] `Devices/DeviceSlice.cs:76`, `DeviceData.cs:869`; [F] `devices/modbus/index.js:715-720`(수신 시각만). 반례 [O] 없음 |
| F-08 | 장치 연결 상태를 감시하고 자동 재접속하며, 신선도 저하(stale)를 표시해야 한다 | [F] `device.js:196-239`; [R] `DeviceLogic.cs:405`, `DeviceData.cs:655`; [O] `modbus_master.cpp:62-81`(`isConnected`) |
| F-09 | 제어 명령은 식별자·발행자·생성 시각·대상·값을 가진 객체로 표현하고 결과를 반환하며 유효기간을 가져야 한다 | [R] `TeleCommand.cs:89-155`, `CommandResult.cs:32`, `CommLine.cs:455`. 반례 [F] `devices/index.js:452`(호출만), [O] `modbus.cpp:516`(직접 쓰기) |
| F-10 | 한계 초과·상태 변화로부터 알람/이벤트를 생성하고 발생·해제·확인의 수명주기와 확인자를 기록해야 한다 | [F] `alarms/index.js:572-635,712-716`, `alarmstorage.js:56-57`; [R] `Event.cs:67-157`, `EventAck.cs:35`, `Server CoreLogic.cs:803,949` |
| F-11 | 이력 저장소는 수집과 분리된 교체 가능한 백엔드여야 하고 현재값/이력/이벤트를 구분해야 한다 | [R] `Archives/*ArchiveLogic.cs`, 구현 3종; [F] `storage/daqstorage.js:43-63,191-196` |
| F-12 | 저장 정책(변경 시/주기, 데드밴드, 포인트별 대상 저장소, 보존기간)을 설정할 수 있어야 한다 | [F] `devices/device-utils.js:9-14,35-36`, `daqstorage.js:135`; [R] `Cnl.ArchiveMask`(`Cnl.cs:77`), `ArchiveLogic.cs:144` |
| F-13 | 사용자 정의 로직(제어 프로그램, 수식, 스크립트)을 실행하는 엔진을 제공해야 한다 | [O] `ladder.h:99-100`, `scripts/compile_program.sh:20`; [F] `scripts/msm.js`; [R] `Calculator.cs:266` |
| F-14 | 데이터·이벤트·명령 처리 경로에 가로채기 훅(전/후)을 제공해 확장 모듈이 검증·변환·차단할 수 있어야 한다 | [R] `Modules/ModuleLogic.cs:144-187`, `Server CoreLogic.cs:678-700` |
| F-15 | 수집한 데이터를 여러 상위 시스템/프로토콜로 동시에 반출할 수 있어야 한다 | [R] `DataSources/DataSourceLogic.cs:148-162`, 구현 3종; [O] Modbus/DNP3/ENIP/S7 서버 동시 제공(`interactive_server.cpp:294-398`) |
| F-16 | 엔지니어링 편의를 위한 태그 탐색(browse)과 장치 템플릿을 제공해야 한다 | [F] `device.js:330-374`; [R] `DeviceLogic.cs:361`(`InitDeviceTags`) |
| F-17 | 재기동 후 유지해야 할 값(retain/setpoint)을 저장·복원해야 한다 | [O] `persistent_storage.cpp:81,167`; [F] `device.js:485-507`; [R] `CurrentArchiveLogic.cs:55-60` |
| F-18 | 실행 평면과 분리된 관리 인터페이스(프로그램/설정 배포, 서비스 시작·정지, 상태·로그 조회)를 제공해야 한다 | [O] `interactive_server.cpp:240-470`, `openplc.py:73-267`; [R] `AgentListener.cs:394,738`, `FunctionID.cs:64-67` |
| F-19 | HMI는 서버 푸시로 실시간 값을 받고, 화면 요소별로 표시/조작 가능 여부를 제어해야 한다 | [F] `runtime/index.js:640-663,793-840`; [R] `PluginLogic.cs:109,198` |
| F-20 | 시뮬레이션/테스트용 가상 장치를 제공해야 한다 | [O] `hardware_layers/simulink.cpp`, `blank.cpp`; [R] `DrvSimulator`, `DrvTester`; [F] `fuxaserver`(내부 장치) |

### 5.2 비기능 요구사항

| ID | 요구사항 | 근거 |
|---|---|---|
| N-01 | 제어 로직은 결정적 주기로 실행되어야 하고 사이클 시간·지연을 계측해야 한다 | [O] `main.cpp:134-146,214-236` |
| N-02 | 정지·종료·이상 시 출력을 안전 상태로 내려야 한다 | [O] `main.cpp:257-259`, `utils.cpp:164` |
| N-03 | 한 장치·라인·모듈의 장애가 다른 것에 전파되지 않아야 한다(격리, 예외 포획) | [R] `ScadaCommEngine/DeviceWrapper.cs:156-190`(래퍼가 예외 포획), 라인별 스레드; [F] `device.js:207-218`; [O] 클라이언트별 스레드(`server.cpp:305`) |
| N-04 | 새 프로토콜/하드웨어/저장소/화면을 코어 수정 없이 추가할 수 있어야 한다(명시적 플러그인 계약·등록 메커니즘) | [R] `DriverFactory.cs:45-54`, `ModuleFactory.cs:42`. 반례 [F] `device.js:56-142,580-614`(다중 분기 수정 필요), [O] 재컴파일 필요 |
| N-05 | 통신 재시도 횟수·타임아웃·폴링 주기는 장치/라인 단위로 설정 가능해야 한다 | [R] `DeviceLogic.cs:145,259`, `Config/PollingOptions.cs`; [O] `Settings`의 `Slave_polling/Slave_timeout`; [F] `device.js:29-32,289` |
| N-06 | 저장소·네트워크 부하를 줄이는 수단(변경분 전송, 데드밴드, 구독 기반 푸시)을 가져야 한다 | [F] `device-utils.js:9-36`, `runtime/index.js:425-447`; [R] `DeviceData.cs:815`(`GetModifiedData`) |
| N-07 | 전송 실패를 감지하고 재전송/버퍼링할 수 있어야 한다 | [R] `DeviceSlice.cs:109-114` |
| N-08 | 설정·프로젝트는 데이터로 저장되어 백업·이관·원격 배포가 가능해야 한다 | [F] `project/prjstorage.js:75-86`; [R] `Storages/StorageLogic.cs:39`, Agent; [O] `openplc.db` |
| N-09 | 진단을 위한 통신 프레임 로그와 장치/서비스 상태 정보를 제공해야 한다 | [R] `Connection.cs:215-260`, `DeviceLogic.cs:499`; [O] `ladder.h:123`, `interactive_server.cpp:462`; [F] `api/diagnose/index.js:62` |
| N-10 | 다양한 하드웨어·OS로 이식 가능해야 한다 | [O] `hardware_layers/` 16종, `compile_program.sh`의 플랫폼 분기; [R] 저장소 추상화, .NET |
| N-11 | 다국어 지원 | [R] `Locale.IsRussian` 분기와 `Lang/*.xml`; [F] `runtime/localization.js` |
| N-12 | 동시 세션 수·요청 크기·요청률에 상한을 두어야 한다 | [R] `ListenerBase.cs:59,83`; [F] `api/index.js:49,55-66`. 반례 [O]: 연결마다 스레드 생성, 상한 미확인(`server.cpp:271-305`) |

### 5.3 보안 요구사항

| ID | 요구사항 | 근거(존재) / 반례(부재) |
|---|---|---|
| S-01 | 기본 계정·기본 비밀번호·기본 키를 제품에 포함하지 말고 최초 기동 시 설정을 강제해야 한다 | 반례: [O] `openplc.db` `Users` 행(`openplc/openplc`); [F] `users/usrstorage.js:119-124`(`admin/123456`); [R] 템플릿 `User.xml`, 다수 설정 파일의 동일 `SecretKey`, `ScadaUtils.Crypto.cs:17,23`. 부분 존재: [O] REST는 최초 사용자 생성 방식(`restapi.py:98-110`), `.env` 자동 생성(`config.py:31-42`) |
| S-02 | 인증은 기본 활성이어야 하며, 인증 실패/부재 시 익명 권한으로 폴백하지 않아야 한다 | 존재: [R] `Program.cs:138-140`(fallback policy). 반례: [F] `settings.default.js:115`, `api/index.js:377-379`, `api/jwt-helper.js:62-72` |
| S-03 | 비밀번호는 적응형 해시(솔트 포함)로 저장해야 한다 | 존재: [F] bcrypt(`usrstorage.js:165`), [O] REST PBKDF2(`restapi.py:62`). 반례: [O] 웹 UI 평문(`webserver.py:621`), [R] MD5 + 고정 솔트(`ScadaUtils.Crypto.cs:28,193-200`) |
| S-04 | 역할 기반 권한을 두고, 조회 권한과 제어 권한을 분리하며, 설비 영역(객체) 단위로 부여할 수 있어야 한다 | 존재: [R] `Right.cs:70-80`, `RightMatrix.cs:219`; [F] `runtime/index.js:793-840`. 반례: [O] 역할 없음(`restapi.py:55-59` TODO) |
| S-05 | 제어 명령(쓰기)은 서버 측에서, 모든 진입 경로에 대해, 대상 포인트 단위로 권한을 검사해야 한다 | 존재: [R] `Server CoreLogic.cs:1578-1615` + `MainApiController.cs:573-600` + `CommLine.cs:799-801`. 부분: [F] REST는 관리자만(`api/command/index.js:120-124`)이나 socket 경로는 인증 여부만 확인(`runtime/index.js:41-46,204-217`). 반례: [O] `modbus.cpp:516-727`, `dnp3.cpp:92-197` |
| S-06 | 쓰기 기능은 전역·라인·장치 단위로 끌 수 있어야 한다(읽기 전용 운전) | 존재: [R] `EnableCommands`(`MainApiController.cs:580` 부근), `CmdEnabled`(`CommLine.cs:801`), `DeviceLogic.CanSendCommands`(`DeviceLogic.cs:180`). 반례: [O] 프로토콜 서버 단위 on/off만 가능 |
| S-07 | 디버그·강제(force)·진단 기능은 운영 프로토콜과 분리하고 별도 인증을 요구해야 한다 | 반례: [O] `modbus.cpp:55-59,825`(FC 0x41–0x45가 Modbus 포트에 공존) |
| S-08 | 로그인·로그아웃·제어 명령·설정 변경·프로그램 다운로드에 대해 "누가·언제·무엇을·결과"를 남기는 감사 로그가 있어야 하고 변조에 대비해야 한다 | 계약 존재: [R] `AuditLogEntry.cs:64-94`, `AuditActionType.cs:60-65`, 명령 이벤트(`Server CoreLogic.cs:994-1015`). 반례: [R] `IAuditPlugin` 구현체 미발견; [F] 전용 감사 로그 미발견(`userack`만); [O] 미발견 |
| S-09 | 관리·HMI·서비스 간 통신은 전송 암호화와 상호 인증을 지원해야 하고 기본 활성이어야 한다 | 부분: [O] REST만 TLS(`webserver.py:2695`), UI는 HTTP(`2747`); [F] 선택(`main.js:267`); [R] 웹 HSTS(`Program.cs:207`), 서비스 간은 비밀번호 필드만 암호화(`ListenerBase.cs:605-610`) |
| S-10 | 필드 프로토콜 서버는 바인드 주소·허용 클라이언트를 제한할 수 있어야 하고 불필요한 프로토콜은 기본 비활성이어야 한다 | 반례: [O] `server.cpp:112`, `dnp3.cpp:431`, 기본 `Settings`(Modbus 502·ENIP·snap7 활성). 존재: [O] 관리 RPC는 루프백(`interactive_server.cpp:180`); [F] `uiHost` 설정(`settings.default.js:69-72`) |
| S-11 | 장치 접속용 자격증명·키는 프로젝트 데이터와 분리해 보호 저장해야 한다 | 부분: [F] `prjstorage.js:78`(분리 저장, 암호화 미확인), `device.js:85`(인증서 디렉터리); [R] 설정 XML의 암호화 문자열(기본 키 사용 가능성 — `ScadaUtils.Crypto.cs:100-102`) |
| S-12 | 세션에는 만료·비활성 타임아웃·폐기 수단이 있어야 한다 | 존재: [O] `webserver.py:590`(5분), `restapi.py:43-48`(메모리 블록리스트); [F] `api/jwt-helper.js:9`; [R] `ListenerBase.cs:83` |
| S-13 | 로그인 시도 제한(무차별 대입 방어)과 API 요청률 제한이 있어야 한다 | 존재: [R] `Security/BruteForceProtector.cs:36-208`; [F] `api/index.js:55-66`. [O] 미발견 |
| S-14 | 사용자 제공 코드(제어 프로그램, 스크립트, 수식, 하드웨어 모듈) 실행은 권한 분리된 역할로 제한하고 격리·무결성 검증을 적용해야 한다 | 반례: [O] `webserver.py:1879-`(로그인한 누구나 Python 하드웨어 코드 편집), `openplc.py:113-204`; [F] `scripts/msm.js:142-147`(격리 없음), 부분 통제 `api/scripts/index.js:54-61`; [R] `Calculator.cs:266` |
| S-15 | 플러그인/드라이버 로드와 런타임 패키지 설치는 관리자 권한과 출처·무결성 검증을 요구해야 한다 | 부분: [F] 관리자 검사(`api/plugins/index.js:61-65`), 출처 검증 미확인; [R] `DriverFactory.cs:52`(검증 미확인) |
| S-16 | 기계 간 인증(API 키, 서비스 계정)은 최소 권한이어야 하고 사람 계정과 구분되어야 한다 | 존재: [R] Application 역할 전용 로그인(`ServerListener.cs:495`), Agent 역할 구분(`AgentListener.cs:232-255`). 반례: [F] API 키 = 관리자(`api/index.js:366-368`) |
| S-17 | 외부 인증원(디렉터리 서비스) 연동이 가능해야 한다 | 존재: [R] `ModuleLogic.cs:194-202`, `ModActiveDirectoryLogic`. [O][F] 미발견 |
| S-18 | 로그에 비밀번호·해시·토큰 등 민감정보를 남기지 않아야 하고 운영 기본 로그 레벨은 디버그가 아니어야 한다 | 반례: [O] `restapi.py:23,68`, `config.py:37`(기본 development) |
| S-19 | 자동 로그인·비밀번호 저장 같은 편의 기능은 기본 비활성이고 사용 시 위험이 명시되어야 한다 | 근거: [R] `LoginOptions.cs:67-72`(`AutoLoginUsername/Password`), `AllowRememberMe`(57) |
| S-20 | 관리자 간 권한 경계: 사용자 관리(생성·삭제·역할 지정)는 관리자 역할에만 허용해야 한다 | 존재: [F] `api/users/index.js:78-82`; [R] `Program.cs:132-133`. 반례: [O] `restapi.py:114,201-216`(역할 자가 지정, 아무 인증 사용자나 삭제), 웹 UI `/add-user`(`webserver.py:2046`) |

### 5.4 미확인 사항(후속 확인 필요)

- OpenPLC: snap7 래퍼(`utils/snap7_src`)의 접근통제, EtherNet/IP·PCCC 쓰기 경로 상세, CSRF 방어 유무, 업로드 파일명 검증 상세.
- FUXA: Modbus 외 드라이버의 타임스탬프/품질 처리, `devicesSecurity` 저장 시 암호화 여부, 클라이언트(Angular) 측에서만 수행되는 권한 검사 범위, Node-RED 연동의 실제 인증 동작.
- Rapid SCADA: 템플릿 기본 계정의 평문 비밀번호, DLL 로드 시 서명 검증 유무, `ListenerBase` 이외 경로(웹 ↔ 브라우저 외)의 TLS 설정 문서화 여부, `BruteForceProtector` 상수의 실제 적용 범위(웹 로그인 포함 여부), `IAuditPlugin`의 상용/비공개 구현 존재 여부.

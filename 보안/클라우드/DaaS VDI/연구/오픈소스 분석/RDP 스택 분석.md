> RDP 스택 분석
> 조사일: 2026-11-26 · 소스 읽기만 수행(빌드·실행 없음). 줄번호는 클론 시점 커밋 기준
> 요약: xrdp · FreeRDP · PyRDP의 추상화와 거기서 유추한 요구사항
> 관련: [[보안/클라우드/DaaS VDI/개발/공통요구사항]] · [[보안/클라우드/DaaS VDI/연구/오픈소스 프로젝트]]

## 프로젝트별 한 장 요약

아래 그림은 이 노트의 내용을 프로젝트마다 한 장으로 줄인 것이다. 근거(파일 · 줄번호)는 그 아래 본문에 있다.

### xrdp

![[daas-p07-xrdp.png]]

코드 구조 (모듈 · 의존 · 추상화 구현 방식)

![[daas-c07-xrdp.png]]

### FreeRDP

![[daas-p08-FreeRDP.png]]

코드 구조 (모듈 · 의존 · 추상화 구현 방식)

![[daas-c08-FreeRDP.png]]

### PyRDP

![[daas-p09-PyRDP.png]]

코드 구조 (모듈 · 의존 · 추상화 구현 방식)

![[daas-c09-PyRDP.png]]

---

## 상세 분석 (원문)

# DaaS/VDI 디스플레이 프로토콜(RDP) 스택 분석 — C조 (xrdp / FreeRDP / pyrdp)

> 목적: 오픈소스 RDP 스택 3종의 소스코드를 읽어 "추상화 구조"를 파악하고, 거기서 DaaS/VDI 제품을 만들 때 필요한 공통 요구사항(기능/비기능/보안)을 역으로 유추한다. 방어·학습 목적의 코드 분석이며, 수정·빌드·실행·네트워크 접근은 하지 않았다.
>
> 분석 대상 커밋:
> - xrdp: `7a7a112` (2026-11-20)
> - FreeRDP: `258ffd3` (2026-11-25)
> - pyrdp: `b610809` (2026-05-12)
>
> 규칙 준수: 코드에서 실제로 확인한 것만 기재. 줄번호는 실제 확인한 것. 확인 못 한 것은 "미확인"으로 표기.

---

## 0. 세 프로젝트의 역할 요약

| 프로젝트 | 역할 | RDP 측 지위 |
|---|---|---|
| **xrdp** | 리눅스용 RDP **서버**. 클라이언트(mstsc 등)를 받아 백엔드 디스플레이(Xorg/Xvnc/원격 RDP)에 연결 | 서버 (libxrdp가 프로토콜, xrdp가 WM, sesman이 인증·세션, chansrv가 채널) |
| **FreeRDP** | RDP **라이브러리 + 클라이언트 + 서버(shadow, proxy)**. winpr 플랫폼 추상화 포함 | 클라이언트/서버/프록시 모두 |
| **pyrdp** | RDP **MITM(Monster-in-the-Middle) + 세션 녹화·재생** 라이브러리 | 중간자 (클라이언트에겐 서버, 서버에겐 클라이언트) |

세 프로젝트 모두 동일한 RDP 계층 스택([MS-RDPBCGR] 기반)을 구현하되 관점이 다르다. 이 관점 차이가 요구사항 도출의 핵심이다 — xrdp는 "서버가 무엇을 통제해야 하는가", FreeRDP는 "프로토콜 추상화의 완전한 형태", pyrdp는 "통제가 없을 때 무엇이 노출되는가"를 보여준다.

---

## 1. 계층 구조와 계층 경계 인터페이스

### 1.1 xrdp

libxrdp가 프로토콜 계층을 소유하고, 각 계층은 상위 계층을 `owner`로 역참조하는 포인터 체인으로 연결된다.

- 계층 구조체 (`libxrdp/libxrdp.h`):
  - `struct xrdp_iso` (:40) — X.224/TPKT. 멤버: `requestedProtocol`, `selectedProtocol`, `failureCode`, `struct trans *trans`
  - `struct xrdp_mcs` (:61) — MCS. `channel_list`, `userid`, `chanid`
  - `struct xrdp_sec` (:101) — 보안 계층. `crypt_method`, `crypt_level`, RC4/FIPS 키 재료(`decrypt_key`, `encrypt_key`, `sign_key`, `pub_mod`, `pri_exp` 등), `rsa_key_bytes`
  - `struct xrdp_fastpath` (:76) — fast-path 입출력
  - `struct xrdp_channel` (:158) + `struct xrdp_drdynvc` (:140) — 정적/동적 가상 채널
  - `struct xrdp_rdp` (:169) — RDP 세션 상태, `client_info`, mppc/rfx 인코더
- 세션 경계: `struct xrdp_session` (`libxrdp/libxrdpinc.h:70`) — `trans`, `callback`(상위로 이벤트 전달), `rdp`, `orders`, `client_info`. libxrdp 공개 API 경계.
- 트랜스포트 추상화: `struct trans` (`common/trans.h:94`), `struct source_info` (:88), `enum xrdp_source` (:53). TCP/TLS/UDS/vsock를 동일 인터페이스로 추상화.
- TLS는 `struct xrdp_tls` (common/trans.h:41 전방선언, 구현 `common/ssl_calls.c`)로 trans 밑에 끼워진다.

계층 경계 흐름: `trans`(TCP/TLS/UDS) ↔ `xrdp_iso`(X.224) ↔ `xrdp_mcs`(MCS) ↔ `xrdp_sec`(보안) ↔ `xrdp_channel`/`xrdp_fastpath`(채널·그래픽) ↔ `xrdp_rdp`(세션) ↔ `xrdp_session.callback`(상위 WM/mm).

### 1.2 FreeRDP

계층이 `rdpRdp` 하위 서브오브젝트로 구성되며, 공개 API는 `rdpContext`/`freerdp` 인스턴스를 통해 노출된다.

- `libfreerdp/core/` 파일이 계층별로 분리: `tcp.c`, `transport.c`, `tpkt.c`, `tpdu.c`, `x224`(`nego.c`에서 X.224 nego), `mcs.c`, `gcc.c`, `security.c`, `nla.c`(CredSSP), `rdstls.c`, `aad.c`(Azure AD), `license.c`, `channels.c`, `fastpath.c`, `rdp.c`, `connection.c`(상태머신), `activation.c`
- 트랜스포트 I/O 계층 경계가 명시적 함수 포인터 테이블로 추상화됨: `struct rdp_transport_io` (`include/freerdp/transport_io.h:110`):
  - `TCPConnect`, `TLSConnect`, `TLSAccept`, `TransportAttach`, `TransportDisconnect`, `ReadPdu`, `WritePdu`, `ReadBytes`, `GetPublicKey`, `SetBlockingMode`, `ConnectLayer`, `AttachLayer`
  - 기본 구현: `transport_default_*` (`libfreerdp/core/transport.c:1834~`), 설치 지점 `transport.c:1833~`
- 연결 상태머신: `enum` `CONNECTION_STATE_INITIAL` 등 (`include/freerdp/types.h:42~`), 전이는 `libfreerdp/core/connection.c`.

### 1.3 pyrdp

레이어 체인을 명시적 객체 그래프로 조립한다. 가장 "교과서적"으로 계층 경계가 드러난다.

- `pyrdp/layer/`: `tcp.py`, `segmentation.py`, `tpkt.py`, `x224.py`, `mcs.py`, `rdp/security.py`, `rdp/fastpath.py`, `rdp/slowpath.py`
- 조립: `RDPLayerSet` (`mitm/layerset.py:12`):
  ```
  tcp → segmentation → (TPKT) → tpkt → x224 → mcs
  security(SecurityLayer, TLS 여부에 따라 동적) / slowPath / fastPath
  ```
  `LayerChainItem.chain(self.tpkt, self.x224, self.mcs)` (:30)로 체인 구성.
- 계층 경계 추상화의 핵심: `BaseLayer`/`Layer`/`IntermediateLayer` (`layer/layer.py:85,126,188`). 각 계층은 `recv(data)` → `mainParser.parse` → `pduReceived(pdu)` → 상위 전달, `sendBytes`로 하위 전달. `shouldForward(pdu)` (:208)로 MITM 통과 제어.

**공통 패턴**: 세 프로젝트 모두 TCP/TLS → X.224/TPKT → MCS → 보안 → (채널/fastpath) → RDP 세션의 동일 계층 스택을 구현하되, **계층 경계 추상화 방식**이 다르다: xrdp=owner 포인터 체인 구조체, FreeRDP=함수 포인터 테이블(transport_io) + 서브오브젝트, pyrdp=Observer 기반 레이어 체인 객체.

---

## 2. 핵심 추상화 (인터페이스·구조체+함수포인터·플러그인 계약)

### 2.1 xrdp — 백엔드 모듈 인터페이스 `struct xrdp_mod`

파일: `xrdp/xrdp_types.h:51`. 백엔드(세션 타입)를 공유 라이브러리 플러그인으로 추상화한 **함수 포인터 테이블**. VDI 제품의 "세션 타입 플러그인 계약"의 교과서적 예.

클라이언트(=백엔드 제어) 함수 포인터 (`:57~78`):
```c
int (*mod_start)(struct xrdp_mod *v, int w, int h, int bpp);
int (*mod_connect)(struct xrdp_mod *v, int fd);
int (*mod_event)(struct xrdp_mod *v, int msg, long p1..p4);
int (*mod_signal)(struct xrdp_mod *v);
int (*mod_end)(struct xrdp_mod *v);
int (*mod_set_param)(struct xrdp_mod *v, const char *name, const char *value);
int (*mod_session_change)(...);
int (*mod_get_wait_objs)(...); int (*mod_check_wait_objs)(...);
int (*mod_frame_ack)(...); int (*mod_suppress_output)(...);
int (*mod_server_monitor_resize)(...); ...
```
서버(=xrdp 코어가 제공) 콜백 (`:82~206`): `server_begin_update`, `server_end_update`, `server_paint_rect`, `server_set_pointer`, `server_palette`, `server_set_clip`, `server_draw_line`, `server_draw_text`, `server_get_channel_count`, `server_query_channel`, `server_send_to_channel`, `server_bell_trigger`, `server_chansrv_in_use`, RAIL 관련 `server_window_new_update`/`server_window_delete`/`server_notify_*`/`server_monitored_desktop`, GFX `server_egfx_cmd`, `server_paint_rects_ex`(shmem 공유), `server_session_info` 등. `tintptr mod_dumby[...]`/`server_dumby[...]`로 ABI 패딩(바이너리 호환성 확보).

구현체 목록(각 파일의 `mod_init()`에서 포인터 설정):
- **xup** (Xorg/xorgxrdp 백엔드): `xup/xup.c` — `mod_init()` (:2499), `lib_mod_start`(:58), `lib_mod_connect`(:264, UDS 또는 TCP 연결), `lib_mod_event`(:433), `lib_mod_set_param`(:2391)
- **vnc** (Xvnc 백엔드): `vnc/vnc.c` — `mod_init()` (:2840)
- **neutrinordp** (원격 RDP 프록시 백엔드, FreeRDP 기반): `neutrinordp/xrdp-neutrinordp.c` — `mod_init()` (:2528)

로더: `xrdp_mm.c` — `g_load_library(libname)` (:141), `g_get_proc_address(..., "mod_init")`/`"_mod_init"` (:5368~5372), `"mod_exit"` (:5383). 서버 콜백 설치는 `xrdp_mm.c:5426~` (`self->mod->server_* = server_*`).

동적 채널 프로시저 추상화: `struct xrdp_drdynvc_procs` (`libxrdp/libxrdpinc.h:89`) — `open_response`/`close_response`/`data_first`/`data` 콜백.

### 2.2 FreeRDP — `rdpContext` / `freerdp` 인스턴스 콜백

파일: `include/freerdp/freerdp.h`. 인스턴스 콜백이 **ALIGN64 + 오프셋 주석 + padding**으로 ABI 안정성을 확보한 함수 포인터 테이블. (각 콜백 offset 주석을 코드가 명시)

- `struct rdp_context` (`:357`) — `instance`, `peer`(서버 모드), `ServerMode`, `rdp`, `gdi`, `rail`, `cache`, `channels`, `graphics`, `input`, `update`, `settings`, `codecs`, `autodetect`, `dump`(streamdump) 등 서브오브젝트 집합.
- `struct rdp_freerdp` (`:461`) 주요 콜백 (typedef는 `:101~348`):
  - `pContextNew ContextNew` (offset 33), `pContextFree` (34)
  - `pConnectCallback PreConnect` (48), `PostConnect` (49)
  - `pAuthenticate Authenticate` (50, deprecated) / `pAuthenticateEx AuthenticateEx` (69, `rdp_auth_reason` 전달)
  - `pVerifyCertificateEx VerifyCertificateEx` (66), `pVerifyChangedCertificateEx` (67), `pVerifyX509Certificate` (53)
  - `pLogonErrorInfo LogonErrorInfo` (54)
  - `pPostDisconnect` (55), `PostFinalDisconnect` (60)
  - `pAuthenticate GatewayAuthenticate` (56), `pPresentGatewayMessage` (57), `Redirect` (58), `LoadChannels` (59)
  - `pSendChannelData` (64), `pReceiveChannelData` (65), `pSendChannelPacket` (68)
  - `pChooseSmartcard ChooseSmartcard` (70), `pGetAccessToken GetAccessToken` (71, AAD 토큰), `pRetryDialog` (72)
- 인증 사유 enum `rdp_auth_reason` (`:88`): `AUTH_NLA, AUTH_TLS, AUTH_RDP, GW_AUTH_HTTP, GW_AUTH_RDG, GW_AUTH_RPC, AUTH_SMARTCARD_PIN, AUTH_RDSTLS` — **인증 방식을 1급 추상화로 분리**.
- 인증서 검증 플래그 `RDP_VERIFY_CERT_FLAG` (`:72~82`): `VERIFY_CERT_FLAG_MISMATCH`(0x80), `_CHANGED`(0x40), `_GATEWAY`(0x20), `_FP_IS_PEM`(0x200) 등.
- 서버 측 peer 콜백: `include/freerdp/peer.h` — `struct rdp_freerdp_peer`, typedef `psPeerCapabilities`(:71), `psPeerPostConnect`(:72), `psPeerActivate`(:73), `psPeerLogon`(:86), `psPeerVirtualChannel*`(:102~109), `psPeerReceiveChannelData`(:99), `psPeerSendServerRedirection`(:89) 등.
- 클라이언트 진입점 계약: `struct rdp_client_entry_points_v1` (`include/freerdp/client.h:73`) — `ClientNew`/`ClientFree`/`ClientStart`/`ClientStop`, 심볼명 `RDP_CLIENT_ENTRY_POINT_NAME = "RdpClientEntry"` (:92). 기본 콜백 설치는 `client/common/client.c:158~160` (`AuthenticateEx = client_cli_authenticate_ex`, `VerifyCertificateEx = client_cli_verify_certificate_ex`).
- 설정 추상화: `rdpSettings` — getter/setter API `freerdp_settings_get_bool/set_bool/get_uint32` 등으로 캡슐화 (`libfreerdp/core/settings.c`). 수백 개의 `FreeRDP_*` 키로 모든 동작을 설정화.
- 그래픽/입력 파이프라인: `struct rdp_update` (`include/freerdp/update.h:200`) — `BeginPaint`/`EndPaint`/`BitmapUpdate`/`DesktopResize`/`SurfaceBits`/`SurfaceCommand`/`SurfaceFrameAcknowledge`/`RefreshRect`/`SuppressOutput`/`RemoteMonitors`, 하위 `pointer`/`primary`/`secondary`/`altsec`/`window` 업데이트. `struct rdp_input` (`input.h:87`) — `SynchronizeEvent`/`KeyboardEvent`/`UnicodeKeyboardEvent`/`MouseEvent`/`ExtendedMouseEvent`/`RelMouseEvent`/`QoEEvent`.

### 2.3 FreeRDP — 채널 애드인 계약 (SVC/DVC)

- **정적 가상 채널(SVC)**: 심볼명 `CHANNEL_EXPORT_FUNC_NAME = "VirtualChannelEntry"` / `"VirtualChannelEntryEx"` (`include/freerdp/svc.h:32`). 진입점 구조체 `CHANNEL_ENTRY_POINTS_FREERDP` (:42) — `pVirtualChannelInit`/`Open`/`Close`/`Write` + `MagicNumber`/`context`.
- **동적 가상 채널(DVC)**: `include/freerdp/dvc.h`. 플러그인 인터페이스 `s_IWTSPlugin` (:107) — `Initialize`/`Connected`/`Disconnected`/`Terminated`/`Attached`/`Detached`. 채널 콜백 `s_IWTSVirtualChannelCallback` (:144) — `OnDataReceived`/`OnOpen`/`OnClose`. 리스너 `s_IWTSListenerCallback.OnNewChannelConnection` (:137). 진입점 `PDVC_PLUGIN_ENTRY = "DVCPluginEntry"` (:174), 진입점 서비스 `S_IDRDYNVC_ENTRY_POINTS` (:163) — `RegisterPlugin`/`GetPlugin`/`GetRdpSettings`/`GetRdpContext`.
- 애드인 로더: `include/freerdp/addin.h:54` — `FREERDP_LOAD_CHANNEL_ADDIN_ENTRY_FN`, `freerdp_load_dynamic_addin` (:74). 보안상 full path 로딩 불가, 설치 경로만 조회(프록시 모듈도 동일 정책).
- 채널 목록(구현체): `channels/` 아래 각각 `client`/`server` 하위로 분리 — `cliprdr`(양방향), `rdpdr`(드라이브/프린터/스마트카드/시리얼), `rdpsnd`(오디오출력), `audin`(오디오입력), `rail`(원격앱), `rdpgfx`(GFX), `disp`(동적해상도), `drdynvc`(DVC 매니저), `urbdrc`(USB), `rdpecam`(카메라), `video`/`tsmf`, `smartcard`, `encomsp`/`remdesk`(원격지원), `echo`, `location`, `geometry`, `telemetry` 등. (`channels/CMakeLists.txt` 및 디렉터리 확인)

### 2.4 FreeRDP — 프록시 모듈 API

파일: `include/freerdp/server/proxy/proxy_modules_api.h`. 외부 모듈이 훅/필터를 등록하는 플러그인 계약.

- 플러그인 구조체 `struct proxy_plugin` (:~) — `name`/`description`/`PluginUnload` + **훅**(`proxyHookFn`)과 **필터**(`proxyFilterFn`)의 함수 포인터 집합. reserved 필드로 ABI 확장 여지.
  - 클라이언트 훅(custom=`rdpContext*`): `ClientInitConnect`(66), `ClientPreConnect`(68), `ClientPostConnect`(69), `ClientX509Certificate`(71), `ClientLoginFailure`(72), `ClientEndPaint`(73), `ClientRedirect`(74), `ClientLoadChannels`(75)
  - 서버 훅(custom=`freerdp_peer*`): `ServerPostConnect`(96), `ServerPeerActivate`(97), `ServerChannelsInit`(98), `ServerSessionStarted`(102), `ServerSessionEnd`(100)
  - 필터(통과 여부 반환): `KeyboardEvent`(128), `MouseEvent`(129), `ClientChannelData`(130), `ServerChannelData`(131), `DynamicChannelCreate`(132), `ServerFetchTargetAddr`(133), `ServerPeerLogon`(134), `UnicodeEvent`(136), `DynChannelToIntercept`(138)/`DynChannelIntercept`(139)/`StaticChannelToIntercept`(140)
- 매니저 `struct proxy_plugins_manager` — `RegisterPlugin`/`SetPluginData`/`GetPluginData`/`AbortConnect`. 모듈 진입점 `proxyModuleEntryPoint`.
- 필터 반환 규약: `proxyFilterFn`은 "프록시해야 하면 TRUE, 무시하면 FALSE" (파일 헤더 주석). 훅 실행: `pf_modules_run_hook`/`pf_modules_run_filter` (`server/proxy/pf_modules.c`).
- 예제 모듈: `server/proxy/modules/demo/demo.cpp`(모든 훅 등록 예, :444~), `bitmap-filter`(비트맵 캐시 필터, 설명 :44 "deactivates and filters persistent bitmap cache"), `dyn-channel-dump`(동적 채널 데이터를 파일로 덤프, :57). 모듈 로딩 규약: `proxy-<name>-plugin.<ext>`, `<install>/lib/freerdp3/proxy`에서만 조회 (`modules/README.md`).

### 2.5 pyrdp — Layer / Parser / Observer / Recorder / MITM

- **Observer/Subject** 패턴이 전체 뼈대: `Observer` (`core/observer.py:10`) — `onPDUReceived`, `setPeer`(:27); `CompositeObserver` (:35)로 다중 관찰자 fan-out; `Subject` (`core/subject.py:10`) — `addObserver`/`removeObserver`.
- **Layer**: `core/observer.py`+`layer/layer.py`. `LayerObserver`(:16), `LayerRoutedObserver`(핸들러 dict 라우팅, :30), `LayerStrictRoutedObserver`(:57).
- **Parser**: `parser/parser.py` — `BaseParser.parse/doParse/write` (:13), `Parser`(:44, bytes), `StreamParser`(:56). PDU 정의는 `pdu/`, 각 프로토콜 파서는 `parser/rdp/`, `parser/mcs.py`, `parser/x224.py` 등.
- **Recorder**: `recording/recorder.py:19` — `addTransport`/`setParser`/`record(pdu, messageType)`(:65). 녹화는 `FileLayer`(:94, `.pyrdp` replay 파일로 저장)로 흘려보냄. 녹화 관찰자 `RecordingFastPathObserver`/`RecordingSlowPathObserver` (`recording/observer.py:14,25`).
- **MITM 컴포넌트**(모두 `mitm/`): 조립의 중심 `RDPMITM` (`mitm/RDPMITM.py:47`). 계층별 MITM: `TCPMITM`, `X224MITM`, `MCSMITM`, `SecurityMITM`, `SlowPathMITM`, `FastPathMITM`, 채널별 `ClipboardMITM`, `DeviceRedirectionMITM`, `VirtualChannelMITM`, 공격자 주입 `AttackerMITM`. 공유 상태 `RDPMITMState` (`mitm/state.py:19`).
- MITM의 기본 구조: 각 MITM은 `client` 레이어와 `server` 레이어를 받고 각 측에서 수신한 PDU를 변형/기록 후 반대편으로 전달(예: `onClientPDUReceived`/`onServerPDUReceived`). `RDPMITM.buildChannel` (`RDPMITM.py:262`)이 MCS 채널 ID→이름 매핑으로 적절한 채널 MITM을 생성(IO/CLIPBOARD/DEVICE_REDIRECTION/가상채널).

---

## 3. 확장점 — 무엇을 구현해야 새 기능이 붙는가

### 3.1 새 백엔드(세션 타입)
- **xrdp**: `struct xrdp_mod`의 `mod_*` 함수 포인터를 구현한 공유 라이브러리를 작성하고 `mod_init()`/`mod_exit()` 심볼을 export. `xrdp.ini`의 세션 섹션에 `lib=libXXX.so`로 등록(예: `[Xorg] lib=libxup.so`, `[Xvnc] lib=libvnc.so`, `[neutrinordp-any] lib=libxrdpneutrinordp.so` — `xrdp/xrdp.ini.in:265,278,347`). 로더가 런타임에 `g_load_library`로 적재.
- **FreeRDP 클라이언트**: `RdpClientEntry` 진입점에서 `rdp_client_entry_points_v1`을 채운다(`ClientNew`/`Start` 등, `client.h:73`).
- **FreeRDP shadow 서브시스템**: `RDP_SHADOW_ENTRY_POINTS`(`include/freerdp/server/shadow.h:275~`)의 `New`/`Init`/`Start`/`EnumMonitors`/`SynchronizeEvent`/`KeyboardEvent`/`MouseEvent`/`Authenticate`/`ClientConnect`/`ClipboardOffer` 등 `pfnShadow*`를 구현. 진입점 `pfnShadowSubsystemEntry`(:69).

### 3.2 새 가상 채널
- **FreeRDP SVC**: `VirtualChannelEntry`(Ex) export + `CHANNEL_ENTRY_POINTS_FREERDP` 사용(`svc.h`).
- **FreeRDP DVC**: `DVCPluginEntry` export + `IWTSPlugin`/`IWTSVirtualChannelCallback` 구현(`dvc.h`). 애드인은 `channels/<name>/client|server`로 추가.
- **xrdp(서버 측 채널 처리)**: chansrv에서 정적 채널은 이름 비교로 하드코딩 디스패치(`sesman/chansrv/chansrv.c:433~` `CLIPRDR_SVC_CHANNEL_NAME`/`RDPSND`/`RDPDR`/`RAIL`). 동적 채널은 `chansrv_drdynvc_open(name, flags, struct chansrv_drdynvc_procs*, ...)`(`chansrv.h:72`)로 `open_response`/`close_response`/`data_first`/`data` 콜백 등록. libxrdp 레벨은 `libxrdp_drdynvc_open`(`libxrdpinc.h:271`) + `xrdp_drdynvc_procs`.

### 3.3 새 인증 방식
- **xrdp(sesman)**: 인증은 `struct auth_info` opaque 핸들 + `auth_userpass`/`auth_uds`/`auth_start_session`/`auth_end`/`auth_set_env`/`auth_put_env` 인터페이스(`sesman/libsesman/sesman_auth.h`)로 추상화. 구현체를 컴파일타임 선택: `verify_user.c`(crypt, 기본), `verify_user_pam.c`(PAM), `verify_user_bsd.c`(BSD) — `configure.ac:404~418`에서 `AUTHMOD_OBJ`/`AUTHMOD_LIB` 선택. 새 방식은 이 6개 함수를 구현하면 됨.
- **FreeRDP**: 인증 방식 자체가 `rdp_auth_reason` enum + nego 프로토콜 비트(`PROTOCOL_RDP/SSL/HYBRID/HYBRID_EX/RDSTLS/RDSAAD`)로 추상화. 클라이언트는 `AuthenticateEx`/`GetAccessToken`(AAD)/`ChooseSmartcard` 콜백 구현. 서버는 `psPeerLogon`.

### 3.4 새 프록시 필터/훅
- **FreeRDP proxy**: `proxyModuleEntryPoint`에서 `RegisterPlugin(plugin)` 호출, `proxy_plugin`의 원하는 훅/필터 함수 포인터만 설정(`proxy_modules_api.h`). 설정 `[Plugins] Modules=`/`Required=`에 이름 등록(`modules/README.md`). 필터는 TRUE/FALSE로 이벤트 통과/차단, 훅은 FALSE 반환 시 연결 중단 가능.
- **pyrdp**: 새 채널 MITM 클래스를 만들고 `RDPMITM.buildChannel` 디스패치에 연결, Observer로 로깅/녹화 등록. 공격 주입은 `AttackerMITM`.

---

## 4. 리디렉션 채널별 통제 지점 — 설정 키와 기본값

### 4.1 xrdp

**채널 전역 on/off**:
- `allow_channels=true` (xrdp.ini.in:106 / 기본 true). 코드: `channels_allowed` 파싱 `libxrdp/xrdp_rdp.c:112`, 적용 `libxrdp/xrdp_sec.c:2042` (0이면 MCS 채널 join 거부 — "All channels are disabled by configuration").
- `[Channels]` 섹션(xrdp.ini.in:238~250): `rdpdr=true`, `rdpsnd=true`, `drdynvc=true`, `cliprdr=true`, `rail=true`, `xrdpvr=true`. **중요**: 로드 로직(`xrdp/xrdp_wm.c:36~` `xrdp_wm_load_channel_config`)은 `[Channels]`에 명시되지 않은 클라이언트 채널을 **기본 차단**(`int disabled = 1; /* Channels disabled if not found */`, 약 :55). 즉 화이트리스트 방식.
- 세션별 override: `channel.<name>=` (xrdp.ini.in:388~394 주석, Xorg/Xvnc/neutrinordp 섹션). 적용: `xrdp_mm.c:2593` `xrdp_mm_update_allowed_channels` → `libxrdp_disable_channel`. 채널 비활성 확인: `libxrdp/xrdp_channel.c:125,772`.

**클립보드(cliprdr) 방향별 통제** — sesman.ini `[Security]`:
- `RestrictOutboundClipboard=none` (sesman.ini.in:36, 서버→클라이언트) / `RestrictInboundClipboard=none` (:45, 클라이언트→서버). 값: `all`/`none`/`text,file,image` 조합, 별칭 `true=all`/`false=none`.
- 코드: 파싱 `sesman/libsesman/sesman_config.c:366,380`, **기본값 0(제한 없음)** `:321~322` (`restrict_outbound_clipboard=0`, `restrict_inbound_clipboard=0`). 비트마스크 변환 `sesman_clip_restrict.c`. 실제 적용은 chansrv: outbound `sesman/chansrv/clipboard.c:1822~2028` (`CLIP_RESTRICT_FILE/TEXT/IMAGE`), inbound `:2119~2320`. chansrv 자체 기본값 `sesman/chansrv/chansrv_config.c:39~40` `DEFAULT_RESTRICT_OUTBOUND_CLIPBOARD 0`/`INBOUND 0`.
- **→ 클립보드는 기본적으로 양방향 전부 허용**(제한 없음). type/file/image 단위 세밀 제어 가능.

**드라이브/프린터/스마트카드/시리얼(rdpdr)** — chansrv `devredir.c`:
- 디바이스 announce 처리 `sesman/chansrv/devredir.c:954~`: `RDPDR_DTYP_FILESYSTEM`(드라이브, FUSE 공유 생성 `xfuse_create_share`, :975), `RDPDR_DTYP_SMARTCARD`(:978, `scard_device_announce`). **시리얼/패러럴/프린터는 "not supported"로 처리**(`default:` :994~, `STATUS_NOT_SUPPORTED`). 즉 xrdp는 드라이브+스마트카드만 실제 리디렉션, 프린터/시리얼/패러럴은 미지원.
- FUSE 마운트 제어(sesman.ini `[Chansrv]`): `FuseMountName=thinclient_drives`(:203), `FileUmask=077`(:207, 기본 소유자만 접근), `EnableFuseMount`(:209, 기본 활성 — `chansrv_config.c:41` `DEFAULT_ENABLE_FUSE_MOUNT 1`), `FuseDirectIO`(:213 기본 off).
- rdpdr 채널 자체 on/off는 위 `[Channels] rdpdr`.

**오디오(rdpsnd/audin)**: `[Channels] rdpsnd=true`. 입력 캡처 `audin.c`(`audin_init` 호출 `chansrv.c:486`). 세밀 설정은 `SoundNumSilentFrames*`(sesman.ini.in:228~) 품질 워크어라운드 수준.

**원격앱(RAIL)**: `[Channels] rail=true`. 처리 `sesman/chansrv/rail.c`, libxrdp `xrdp_orders_rail.c`.

**USB**: xrdp 코드에서 USB(urbdrc) 서버측 리디렉션 처리는 **미확인**(chansrv에 해당 핸들러 없음; 드라이브/스마트카드만 devredir 처리).

### 4.2 FreeRDP 프록시 — `[Channels]` 섹션 (server/proxy/pf_config.c)

채널별 bool, 파싱 `pf_config_load_channels` (:509~). **기본값**(`pf_config_get_bool`의 fallback 인자):
| 키 | 기본값 | 코드 |
|---|---|---|
| `GFX` | **true** | :509 |
| `DisplayControl` | **true** | :510 |
| `Clipboard` | **false** | :511 |
| `AudioOutput` | **true** | :512 |
| `AudioInput` | **true** | :513 |
| `DeviceRedirection` | **true** | :514 |
| `VideoRedirection` | **true** | :515 |
| `CameraRedirection` | **true** | :517 |
| `RemoteApp` | **false** | :518 |
| `PassthroughIsBlacklist` | **false** | :520 |

- **클립보드와 RemoteApp만 기본 차단**, 나머지 리디렉션은 기본 허용. 입력 `[Input] Keyboard/Mouse/Multitouch` 모두 기본 true(:535~537).
- 임의 채널 통과/차단: `Passthrough`(화이트리스트 기본) / `Intercept`(가로채기 목록) / `PassthroughIsBlacklist`(블랙리스트 전환). 모드 결정 로직 `server/proxy/pf_utils.c:30` `pf_utils_get_channel_mode` → `PF_UTILS_CHANNEL_INTERCEPT`/`BLOCK`/`PASSTHROUGH`/`NOT_HANDLED`. 적용: 정적채널 데이터 `pf_channel.c:262,300` (BLOCK/INTERCEPT는 DROP), 채널 생성 필터 `pf_client.c:385~`(config로 차단된 채널은 `ChannelDefArray`에서 제거), 동적채널 `channels/pf_channel_drdynvc.c:226~`.
- config→FreeRDP 설정 매핑: `pf_client.c:224~266` (`FreeRDP_AudioCapture=AudioInput`, `AudioPlayback=AudioOutput`, `DeviceRedirection`, `RedirectClipboard=Clipboard`), `pf_server.c:557~`.
- 입력 필터 적용: `pf_input.c:83,141` (`if (!config->Keyboard) return TRUE;` → 드롭).

### 4.3 FreeRDP 클라이언트/코어 기본값 (libfreerdp/core/settings.c)

- `RedirectClipboard` 클라이언트 모드 기본 **TRUE** (:1413, 서버 모드 아닐 때). 클립보드 방향 마스크 `ClipboardFeatureMask=CLIPRDR_FLAG_DEFAULT_MASK`(:1029). 방향 플래그: `CLIPRDR_FLAG_LOCAL_TO_REMOTE`(0x01)/`REMOTE_TO_LOCAL`(0x10)/`*_FILES`(0x02/0x20) (`settings_types.h:349~354`).
- 클라이언트 애드인 자동 로딩은 설정 키 기반: `freerdp_client_load_static_channel_addin`(`client/common/cmdline.c:6619~`), 예: `AudioCapture→audin`, `AudioPlayback→rdpsnd`, `RedirectClipboard→cliprdr`, `SupportGraphicsPipeline→rdpgfx`, `RemoteApplicationMode→rail`. `RedirectDrives/SmartCards/Printers` 중 하나라도 켜지면 `DeviceRedirection=TRUE`(:6834~6840) → rdpdr 로딩.

---

## 5. 보안 추상화 — 협상·TLS·인증서·자격증명·세션정책·로깅

### 5.1 보안 계층 협상

**xrdp** (`libxrdp/xrdp_iso.c:100~` `xrdp_iso_negotiate_security`, 설정 파싱 `xrdp_rdp.c:192~`):
- `security_layer=negotiate`(기본, xrdp.ini.in:69) — `tls`/`rdp`/`negotiate`. `SECURITY_LAYER_*` enum (`xrdp_rdp.c:196~`).
- 지원 프로토콜: 기본적으로 **SSL(TLS)와 RDP만** 지원(`xrdp_iso.c:119~128`). HYBRID(NLA)/HYBRID_EX는 **VMConnect 모드에서만** 활성(`xrdp_iso.c:130~133` `if (client_info->vmconnect) mask |= PROTOCOL_HYBRID|HYBRID_EX`). → **xrdp는 일반적으로 서버측 NLA를 지원하지 않는다**(인증은 로그인 화면/sesman에서).
- `security_layer=tls`면 TLS 불가 시 거부(`failureCode = SSL_REQUIRED_BY_SERVER`, :185~203). FIPS 모드면 classic RDP 거부(`:195~`). 그 외 fallback으로 classic RDP 선택 시 경고 로그("Classic RDP security is not secure", :209).
- 프로토콜 enum 매핑 `xrdp_iso.c:56~59`: SSL/HYBRID/RDSTLS/HYBRID_EX.

**FreeRDP** (`libfreerdp/core/nego.c`, 서버측 `connection.c:1591~`):
- 클라이언트 nego 전략: `nego_attempt_rdsaad`→`rdstls`→`ext`→`nla`→`tls`→`rdp` 순 fallback (`nego.c:614~772`). 각 `RequestedProtocols` 설정.
- 서버측 선택(`connection.c:1625~1668`): 우선순위 RDSTLS > HYBRID_EX > HYBRID(NLA) > SSL(TLS) > RDP, 각각 `settings->RdstlsSecurity/ExtSecurity/NlaSecurity/TlsSecurity/RdpSecurity` AND 클라이언트 요청. 매칭 실패 시 `PROTOCOL_FAILED_NEGO`와 사유(`HYBRID_REQUIRED_BY_SERVER`/`SSL_REQUIRED_BY_SERVER`/`SSL_NOT_ALLOWED_BY_SERVER`).
- 기본 활성(settings.c:1098~1104): `AadSecurity=FALSE`, `ExtSecurity=TRUE`, `NlaSecurity=TRUE`, `TlsSecurity=TRUE`, `RdpSecurity=TRUE`, `RdstlsSecurity=FALSE`, `NegotiateSecurityLayer=TRUE`. 즉 **클라이언트 기본은 NLA+TLS+RDP 모두 허용**(호환성 우선, 서버가 고름). `UseRdpSecurityLayer=FALSE`.

**pyrdp** (`mitm/X224MITM.py:57~`):
- MITM은 클라이언트 nego를 파싱 후 `chosenProtocols &= self.state.config.authMethods`로 **다운그레이드**(:84). 기본 `authMethods = SSL`(`config.py`, cli.py `--auth tls` 기본 → `NegotiationProtocols.SSL`). `--auth ssp`면 `SSL|CRED_SSP`(cli.py:294~297).
- Restricted Admin Mode(NLA 강제) 감지 시 경고(:71~73).

### 5.2 TLS 버전·암호군

**xrdp** (`common/ssl_calls.c`):
- `ssl_protocols=TLSv1.2, TLSv1.3`(xrdp.ini.in:86, **기본 TLS1.2+1.3만**). 파싱 `ssl_get_protocols_from_string`(:1527~, `SSL_OP_NO_*`로 나머지 비활성). 적용 `ssl_tls_accept`(:1121~): `SSL_OP_NO_SSLv2`(:1134), `SSL_OP_NO_COMPRESSION`(:1152), ini 기반 프로토콜 옵션(`options |= ssl_protocols`), `SSL_CTX_set_options`(:1183).
- `tls_ciphers`(xrdp.ini.in:88 주석, 기본 미설정=OpenSSL 기본): `SSL_CTX_set_cipher_list`(:1216), 1글자 초과일 때만 적용.
- TLS pre-master 로깅(디버그): `tls_pms_log_file`(xrdp.ini.in:82), `SSL_CTX_set_keylog_callback(log_pre_master_secret)`(:1264), `HAS_KEYLOG_CALLBACK`(:57). → 운영시 활성화하면 세션 복호화 가능(보안 주의 지점).

**FreeRDP** (`libfreerdp/crypto/tls.c`):
- `TLSMinVersion=TLS1_2_VERSION`(settings.c:1443), `TLSMaxVersion=0`(:1445, 상한 없음). 적용 `SSL_CTX_set_min_proto_version`(tls.c:935).
- `TlsSecLevel=FREERDP_TLS_SECLEVEL_112BIT`(settings.c:1439). 적용 `SSL_CTX_set_security_level`(tls.c:948).
- `AllowedTlsCiphers`면 `SSL_CTX_set_cipher_list`(tls.c:951~953). `SSL_OP_NO_SSLv2/SSLv3`(tls.c:999~1000), `SSL_OP_NO_COMPRESSION`(:1042), `SSL_OP_NO_RENEGOTIATION`(:1305).
- 채널 바인딩: `tls_get_channel_bindings`(tls.c:776) — NLA가 TLS 채널에 바인딩됨. keylog: `SSLKEYLOGFILE` 환경 기반 `SSLCTX_keylog_cb`(tls.c:860,983).

**pyrdp** (`core/ssl.py`):
- `ClientTLSContext`(:19, 서버 연결용) `SSL.SSLv23_METHOD`(모든 버전 허용, Debian 버그 회피 주석) + `OP_NO_TLSv1_3`(:34 — **TLS1.3 복호화 미지원이라 강제 비활성**). `ServerTLSContext`(:38, 클라이언트 대면)도 `OP_NO_TLSv1_3`(:53). → MITM이므로 의도적으로 가장 약한 쪽에 맞춤.

### 5.3 인증서 검증 콜백과 기본 동작

**FreeRDP** (`libfreerdp/crypto/tls.c` `freerdp_tls_verify_certificate`):
- 우선순위: `VerifyX509Certificate` 콜백 있으면 우선(:1835, 없으면 에러 :1839). 그 다음 `IgnoreCertificate`면 **경고 후 무조건 통과**(:1854~, "[DANGER] This prevents MITM attacks from being detected!"). `AuthenticationLevel==0`이면 통과(:1861). 그 외 OpenSSL + `~/.freerdp/certs` 스토어로 검증(`freerdp_certificate_verify` :1872) + 호스트명 매칭(:1884).
- 불일치/미검증 시 `VerifyChangedCertificateEx`/`VerifyCertificateEx`/`VerifyX509Certificate` 콜백으로 사용자 확인(:1953~1990). `AutoAcceptCertificate`(:1943,1922), `AutoDenyCertificate`(:1948,1925) 설정으로 자동화 가능. 기본 CLI 동작은 사용자에게 Y/T/N 프롬프트(`client/common/client.c:840` `client_cli_accept_certificate`, stdin 없으면 거부 :850~852).
- `CertificateCallbackPreferPEM=FALSE`(settings.c:1151).
- **기본 동작은 안전**: IgnoreCertificate/AutoAccept는 명시적 opt-in, 미설정 시 검증+프롬프트.

**xrdp**: 서버이므로 자신의 인증서를 제시(`certificate=`/`key_file=`, xrdp.ini.in:78~79; 미설정 시 `XRDP_CFG_PATH/cert.pem`,`key.pem` 기본 경로 `xrdp_rdp.c:224,256`). 클라이언트 인증서 검증은 해당 없음(RDP 서버는 클라 인증서 검증 안 함).

**xrdp neutrinordp 백엔드**(원격 RDP로 재접속 시 — 프록시 역할): `lfreerdp_verify_certificate`(`neutrinordp/xrdp-neutrinordp.c:2497`)가 **무조건 1(수락) 반환**("no code here"), 그리고 `instance->settings->ignore_certificate = 1`(:2088). → **neutrinordp 프록시 모드는 백엔드 서버 인증서를 검증하지 않는다**(보안 취약 지점, 방어 요구사항 근거).

**FreeRDP proxy**(`server/proxy/pf_client.c`): 타깃 서버 인증서 검증은 `TargetCertPolicy`로 제어. `pf_client_verify_X509_certificate`(:1208) → `pf_client_verify_by_policy`(:1175): `VERIFIED_HOST_MATCH`/`VERIFIED`(:1189~1192 strict), `PINNED`(:1193 해시/PEM 핀), `ALLOW`(:1195 무검증), `DENY`(:1199 거부). **기본값 DENY**(`pf_config.c:1662` `pf_config_policy_from_str` 기본). `PINNED`은 해시/PEM 필수(`pf_config.c:476~`). → 프록시 기본은 안전(DENY), 운영자가 명시 설정 필요.

### 5.4 자격증명 처리·로그인 실패

**xrdp(sesman)**:
- 인증 흐름: SCP 프로토콜로 sesman에 로그인 요청(`sesman/scp_process.c:74` `process_sys_login_request`) → sesexec 프로세스가 PAM/crypt로 검증(`login_info.c:265~`). 재시도 루프 `login_info.c:272` `retry_count = g_cfg->sec.login_retry`(기본 3, `sesman_config.c:319`; sesman.ini `MaxLoginRetry=4` :20).
- 접근 제어 `sesman_access.c:54` `access_login_allowed`: `AllowRootLogin`(기본 코드상 0=불허 `sesman_config.c:318`, 단 sesman.ini.in:19는 `AllowRootLogin=true`), `TerminalServerUsers` 그룹 검사, `AlwaysGroupCheck`(그룹 미정의 시 기본 허용 — 주의 지점). admin 판정 `access_login_is_admin`.
- PAM: `verify_user_pam.c` `common_pam_login`(:241) — `pam_authenticate`(:290), `pam_acct_mgmt`(:306), `PAM_RHOST`에 client_ip 설정(:273). 세션 `pam_open_session`(:468)/`pam_setcred`. 암호 변경 지원 `auth_check_pwd_chg`/`auth_change_pwd`(WIP).
- 로그인 실패 시 `xrdp_mm.c:2740` `xrdp_mm_process_login_response`: `require_credentials`면 fatal(:2766), retry limit 도달 시 "Login retry limit reached"(:2778) 후 연결 종료.
- 사용자 정보 유출 방지 의도 명시: "We won't check for the user being valid here, as this might lead to information leakage"(`scp_process.c:116~`).

**FreeRDP**: 자격증명은 `AuthenticateEx` 콜백으로 수집, NLA는 `nla.c`의 CredSSP/TSCredentials. 로그온 에러 `LogonErrorInfo` 콜백, `STATUS_ACCOUNT_LOCKED_OUT`(nla.c:2354), `ERRCONNECT_LOGON_FAILURE`/`ACCOUNT_LOCKED_OUT`(errconnect.c:79,87). `AuthenticationOnly`(settings.c:1146 기본 FALSE)면 인증만 하고 종료(freerdp.c:185).

### 5.5 NLA/CredSSP 추상화 (FreeRDP nla.c)

- 클라/서버 상태머신: `nla_client_authenticate`/`nla_server_authenticate`(nla.c:684,808), `nla_authenticate`(:971). CredSSP 메시지: `negoToken`/`authInfo`/`pubKeyAuth`(구조 주석 :70~71, SecBuffer :126~127).
- **공개키 echo/hash 바인딩**(MITM 방지 핵심): `nla_encrypt_public_key_echo`/`_hash`(:1019,1045), 복호 검증 `nla_decrypt_public_key_echo`(:1093): `memcmp(public_key, nla->PublicKey)` 불일치 시 "Could not verify server's public key echo"(:1111~1113). 공개키는 TLS 트랜스포트에서 추출(`nla_sec_buffer_alloc_from_data(&nla->PublicKey, data, ...)` :486,740 via `GetPublicKey`). → NLA는 TLS 채널의 서버 공개키에 바인딩되어, **중간자가 서로 다른 TLS 세션을 이으면 공개키 echo 검증에서 실패**.
- 서버측 자격증명 수신/디코드: `nla_server_recv_credentials`(:782), `nla_read_ts_credentials`(:1447) — 암호/스마트카드/RemoteGuard(Kerberos/NTLM) 지원. `peer->RemoteCredentials` 콜백(:1603).
- Early User Auth: `nla_set_early_user_auth`(:164).

### 5.6 세션 정책 (xrdp sesman)

sesman.ini `[Sessions]` + `config_sessions`(`sesman_config.h`), 기본값 `sesman_config.c:438~446`:
- `Policy=Default`(=UB, 사용자별+bpp별 세션 분리; `Separate`/`U`/`B`/`D`/`I`/`N` 조합) — enum `SESMAN_CFG_SESS_POLICY_*`(sesman_config.h:37~47), 매칭 `session_list.c:194~291`. U,B는 항상 활성.
- `MaxSessions=50`(sesman.ini.in:84; 코드 기본 0=무제한 `sesman_config.c:441`). 적용 `scp_process.c:487~488`(초과 시 생성 거부).
- `KillDisconnected=false`(:99, 기본 0), `DisconnectedTimeLimit=0`(:105 `max_disc_time`), `IdleTimeLimit=0`(:111 `max_idle_time`, 0=비활성). **타임아웃 enforcement는 sesman이 직접 안 하고 환경변수로 세션에 전달**: `XRDP_SESMAN_MAX_IDLE_TIME`/`MAX_DISC_TIME`/`KILL_DISCONNECTED`(`sesman/sesexec/session.c:416~423`) → 실제 종료는 세션 스크립트/WM 책임. 재접속 정책 `ReconnectScript=reconnectwm.sh`, `AlwaysRunReconnect`(sesman.ini.in:12~16).
- `StartupWaitTime=1500ms`(:445).

**FreeRDP**: `AutoReconnectionEnabled=FALSE`(settings.c:1342, 단 프록시는 서버측 TRUE `pf_server.c`), `AutoReconnectMaxRetries=20`(:1343). 유휴/최대세션 정책은 FreeRDP 코어 밖(서버 구현 책임).

### 5.7 로깅

- **xrdp**: `[Logging] LogLevel=INFO`(xrdp.ini.in:225), syslog 활성(:226). sesman `xrdp-sesman.log`(sesman.ini.in:141), chansrv는 디스플레이별 로그 `xrdp-chansrv.${DISPLAY}.log`(sesman.ini.in:233~). 연결/인증 이벤트는 `LOG(LOG_LEVEL_INFO,...)`로 다수 기록(예 client computer name `xrdp_sec.c:1674`, 로그인 요청 `scp_process.c:89`).
- **FreeRDP**: WLog 프레임워크(`winpr/libwinpr/log.h`), `WLog_ERR/WARN/DBG`. 메트릭 `rdpMetrics`/`metrics.c`. 스트림 덤프 `streamdump.c`(`TransportDump*` 설정, PCAP류 분석용).
- **pyrdp**: 세션 ID별 구조화 로깅(`logging/`), 공격자/NTLM/RC4 전용 로거(`RDPMITM.py:67,141`), JSON/통계(`StatCounter`). 모든 가로챈 이벤트가 로그 대상.

---

## 6. 세션 녹화·감사 — pyrdp가 무엇을 노출시키는가 (방어 요구사항 근거)

pyrdp는 "MITM이 성공했을 때 RDP 세션에서 무엇이 평문으로 드러나는가"의 완전한 목록이다. 이것이 곧 "NLA/인증서 검증이 없으면 잃는 것"이다.

### 6.1 가로채고 기록하는 것 (PlayerPDUType enum `enum/player.py:15~34`)
- **키 입력(키로깅)**: `FAST_PATH_INPUT`(=1), 스캔코드→텍스트 재구성 `mitm/BasePathMITM.py:52` `onScanCode`(shift/caps/ctrl 상태 추적, inputBuffer 누적). → **키스트로크 전량 캡처**.
- **자격증명**: `CLIENT_INFO`(=3, "Creds on connection"). `SecurityMITM.onClientInfo`(:62~81)가 `username/password/domain/clientAddress`를 평문 로깅(`log.info("Client Info: username=... password=...")` :74) 후 `recorder.record(pdu, CLIENT_INFO)`(:81). 추가로 로그인 휴리스틱(`BasePathMITM.loginAttempt` :33, "Credentials attempt from heuristic")과 rdpdr 기반(`DeviceRedirectionMITM.handleClientLogin` :262).
- **클립보드**: `CLIPBOARD_DATA`(=6). `ClipboardMITM.onFormatDataResponse`(:179~190, 평문 로깅+녹화).
- **파일 전송**: 클립보드 파일(`ClipboardMITM.onFileContentsResponse` :140~, SHA-256로 저장 `FileMapping` `fileDir`) 및 드라이브 리디렉션 파일(`DeviceRedirectionMITM.handleReadResponse` :234, `FileMapping.generate` :230). 파일은 해시 이름으로 디스크 저장(`FileMapping.py:45~61` sha256).
- **화면(그래픽)**: `FAST_PATH_OUTPUT`(=2), `BITMAP`(=14). `RecordingFastPathObserver`로 녹화(`RDPMITM.py:299,303`).
- **NTLM 해시**: `--auth ssp` 또는 NLA 강제 시 `NLAHandler`(`security/nla.py:52~`)가 NetNTLMv2 해시 추출/로깅("[!] NTLMSSP Hash:" :82, `NTLMSSPHashFormatter`).
- **공격 주입**: `AttackerMITM` — 가짜 키입력/마우스 주입, `sendPayload`(`RDPMITM.py:413~`)가 Win+R로 cmd 실행 후 페이로드 타이핑(자동 공격 PoC).

녹화 포맷: `.pyrdp` replay 파일(`FileLayer`, `RDPMITM.py:147`), MP4/JSON 변환(`convert/`). 재생: `player/`(Qt UI), 헤드리스 `HeadlessEventHandler`.

### 6.2 NLA가 MITM을 어떻게 제한하는가 (코드 확인 범위)
- pyrdp가 NLA(CredSSP)를 **투명 중계만** 하고 복호화하지 못함: `NLAHandler`가 "forwarding them transparently"(`nla.py:20~24`). 서버가 NLA 강제(`HYBRID_REQUIRED_BY_SERVER`) 시 `X224MITM.onConnectionConfirm`(:126~)에서 세 가지로만 대응: (1) **리디렉션 호스트**(NLA 없는 다른 서버로), (2) **NTLM 해시 캡처 후 크랙**(온라인 MITM 불가, 오프라인 크랙 필요), (3) 서버 인증서+개인키를 미리 확보해 `--auth ssp`로 가장(`README.md:225~`). → 즉 **NLA + 유효 인증서 + 개인키 비노출이면 실시간 MITM이 성립하지 않는다**. 이것이 방어 요구사항의 직접 근거.
- 반대로 **인증서 검증 없음 + NLA 없음**이면 pyrdp가 TLS를 그대로 양쪽에 수립(`doClientTls`가 서버 인증서를 복제 `CertificateCache.clone` `core/ssl.py:60~88`)하고 위 6.1 전량이 노출. TLS1.3은 복호화 미지원이라 강제 다운그레이드(`OP_NO_TLSv1_3`).

### 6.3 FreeRDP 프록시의 세션 캡처
- 트랜스포트 전체 덤프: `TransportDump`/`TransportDumpFile`(settings_types_private.h:468~, `streamdump.c`).
- 동적 채널 덤프 모듈: `dyn-channel-dump`(`server/proxy/modules/dyn-channel-dump/`), 지정 DVC 데이터를 `.dump` 파일로 저장(:184). 가로채기(INTERCEPT) 모드 채널만 재조립(`pf_channel_drdynvc.c:232`).
- 즉 FreeRDP proxy도 중앙 집중식 세션 기록/감사가 가능(합법적 게이트웨이/감사 용도).

---

## 7. 비기능 — 코덱·대역폭·멀티모니터·프로세스 모델·플랫폼 추상화

### 7.1 코덱 협상
- **xrdp**: GFX 코덱 설정 `xrdp/gfx.toml` — `[codec] order=["H.264","RFX"]`, `h264_encoder="x264"`(또는 OpenH264). EGFX caps 협상 `xrdp_mm.c:1189~` `xrdp_mm_egfx_caps_advertise`(`XR_RDPGFX_CAPVERSION_8~107` :1237~1260), H264/RFX-Pro 매칭(`egfx_flags` XRDP_EGFX_H264/RFX_PRO, :1283~1287). 인코더 선택 `xrdp_encoder.c:187~` `xrdp_encoder_create`: jpeg/h264/rfx/gfx 분기, `process_enc_*` 함수 포인터(`xrdp_encoder.h:42`). H264 라이브러리 런타임 체크(`libh264_loaded` `xrdp_types.h`, 실패 시 GFX fallback). 인코더 백엔드 플러그인: `xrdp_encoder_x264.c`/`xrdp_encoder_openh264.c`.
- **FreeRDP**: 코덱 비트플래그 `FREERDP_CODEC_*`(`codecs.h:48~69`): INTERLEAVED/PLANAR/NSCODEC/REMOTEFX/CLEARCODEC/PROGRESSIVE/AVC420/AVC444/AV1/HEVC. 코덱 구현 `include/freerdp/codec/`(rfx/nsc/h264/progressive/clear/planar/zgfx 등). `GfxH264=FALSE`/`GfxAVC444=FALSE` 기본(settings.c:1349~1350), GFX는 `SupportGraphicsPipeline`.
- **pyrdp**: 자체 디코딩 제한 → downgrade로 GFX/서피스 비활성(아래).

### 7.2 대역폭 적응 (autodetect)
- **FreeRDP**: `rdpAutoDetect`(`autodetect.h:110~`) 함수 포인터 — `RTTMeasureRequest/Response`, `BandwidthMeasureStart/Payload/Stop/Results`, `NetworkCharacteristicsResult/Sync`, `OnConnectTimeAutoDetectBegin`. `NetworkAutoDetect=TRUE` 기본(settings.c:1137). `ConnectionType` 설정.
- **xrdp**: 연결 타입 기반 인코더 튜닝 — `mcs_connection_type`(`xrdp_sec.c:1831` 파싱), `xrdp_encoder.c:261~273`에서 MODEM/BROADBAND_LOW/SATELLITE/BROADBAND_HIGH/WAN/LAN별 분기. gfx.toml에 `[x264.lan]`/`[x264.wan]`/`[x264.modem]` 등 프로파일별 vbv_max_bitrate 설정. autodetect(`AUTODETECT`)는 "not implemented yet"(:273).

### 7.3 멀티 모니터
- **xrdp**: `allow_multimon=true`(xrdp.ini.in:107, 기본). `multimon` 파싱 `xrdp_rdp.c:120`. 동적 모니터 레이아웃 `xrdp_mm.c` `dynamic_monitor_*`(:1551~), `process_display_control_monitor_layout_data`(:1643), 리사이즈 큐 상태머신 `enum display_resize_state`(`xrdp_types.h:361`). `struct monitor_info`(`xrdp_client_info.h:32`), `CLIENT_MONITOR_DATA_MAXIMUM_MONITORS`.
- **FreeRDP**: `UseMultimon`, `MonitorDefArray`(settings_types_private.h:165), disp 채널(`SupportDisplayControl`/`DynamicResolutionUpdate`). 프록시 `UseMultimon=TRUE`(pf_client.c).

### 7.4 프로세스 모델
- **xrdp**: **연결당 fork**. `fork=true`(xrdp.ini.in:6). `xrdp_listen_fork`(`xrdp/xrdp_listen.c:685`, `g_fork()` :692, 자식 `xrdp_child_fork` :698), `fork_list` 관리(:66,885). 각 연결은 `xrdp_process`(`xrdp_types.h:605`)로 독립. 권한 분리: `runtime_user=xrdp`/`runtime_group=xrdp`(xrdp.ini.in:44~45), `g_drop_privileges`(`xrdp/xrdp.c:458`, `check_drop_privileges` :431). 인증/세션은 별도 프로세스 sesman↔sesexec(`sesman/sesexec/`), chansrv는 디스플레이별 프로세스. IPC는 libipm(SCP/EICP/ERCP/CCP 프로토콜, `libipm/`).
- **FreeRDP proxy**: **연결당 스레드**. `pf_server_handle_peer`를 `CreateThread`(`pf_server.c:837`), 클라이언트 측도 스레드(`pf_client_start` :378).
- **pyrdp**: Twisted reactor(단일 프로세스 이벤트루프, `core/twisted.py`), 연결당 `RDPMITM` 인스턴스.

### 7.5 플랫폼 추상화
- **FreeRDP = winpr**: `winpr/libwinpr/` — crt/synch/thread/handle/file/pipe/registry/sspi/sspicli/ncrypt/bcrypt/smartcard/clipboard/sysinfo/winsock/wtsapi 등 Win32 API를 크로스플랫폼으로 재구현. RDP 코어가 이 위에서 OS 독립. (디렉터리 확인)
- **xrdp = common/**: `os_calls`(`g_*` 함수군), `trans`(트랜스포트), `ssl_calls`(OpenSSL 래퍼), `string_calls`, `list`, `fifo` 등 자체 이식 계층.
- **pyrdp**: Python 표준 + Twisted + pyOpenSSL + pycryptodome(RSA/RC4). `core/packing.py`/`per.py`/`ber.py`로 인코딩 추상화.

---

## 8. 추상화가 암시하는 요구사항 (근거 포함)

각 추상화는 "제품이 반드시 가져야 할 능력"을 함의한다.

1. **백엔드/세션 타입을 플러그인으로 교체 가능해야** — 근거: xrdp `struct xrdp_mod` 함수 포인터 테이블 + 공유라이브러리 로딩(`xrdp_types.h:51`, `xrdp_mm.c:5368`), FreeRDP shadow `RDP_SHADOW_ENTRY_POINTS`. VDI는 Xorg/VNC/원격RDP 등 다양한 세션 소스를 동일 인터페이스로 흡수해야 함.
2. **프로토콜 계층 경계를 인터페이스로 분리해야** — 근거: FreeRDP `rdp_transport_io` 함수포인터(transport_io.h:110), pyrdp 레이어 체인. 게이트웨이/프록시/감사 삽입을 위해 트랜스포트와 상위 로직이 분리돼야 함.
3. **가상 채널을 동적으로 등록·on/off·방향별 제어할 수 있어야** — 근거: SVC/DVC 플러그인 계약(svc.h/dvc.h), xrdp `[Channels]` 화이트리스트, 클립보드 방향별 비트마스크(`sesman_clip_restrict`), 프록시 `pf_utils_get_channel_mode`.
4. **보안 계층을 협상하되 서버가 최소수준을 강제할 수 있어야** — 근거: `security_layer=tls` 강제(xrdp_iso.c:185), FreeRDP 서버 nego 우선순위(connection.c:1625), FIPS 모드 classic RDP 거부(xrdp_iso.c:195).
5. **인증 방식이 교체 가능한 추상화여야** — 근거: xrdp `sesman_auth.h`(PAM/crypt/BSD 컴파일 선택), FreeRDP `rdp_auth_reason` + 콜백(AAD/스마트카드/NLA).
6. **인증서 검증 정책을 명시적으로 설정하고 기본은 안전해야** — 근거: FreeRDP `IgnoreCertificate` opt-in+경고(tls.c:1854), 프록시 `TargetCertPolicy` 기본 DENY, 반례로 neutrinordp 무검증(:2497)이 위험으로 지적됨.
7. **세션 정책(동시세션·타임아웃·재접속·분리정책)을 설정화해야** — 근거: sesman `config_sessions`, `Policy=UBDIN`, `MaxSessions`/`IdleTimeLimit`.
8. **연결 격리(프로세스/스레드) + 권한 분리** — 근거: xrdp 연결당 fork + drop_privileges, 인증 전담 프로세스(sesexec) 분리.
9. **코덱/대역폭을 협상·적응하고 안전하게 fallback해야** — 근거: xrdp 런타임 H264 체크 후 GFX fallback, FreeRDP autodetect, 연결타입별 프로파일.
10. **세션 감사/녹화 지점을 가질 수 있어야(합법)** — 근거: FreeRDP proxy 채널 INTERCEPT + 모듈 덤프, streamdump. 동시에 **그 지점이 공격 표면**임(pyrdp가 증명).

---

## 9. 프로젝트 교차: 공통 패턴·차이·유추 요구사항 목록

### 9.1 공통 패턴
- 동일 RDP 계층 스택(TCP/TLS→X.224/TPKT→MCS→보안→채널/fastpath→세션)을 세 관점으로 구현.
- 모든 확장이 **함수 포인터 테이블 / 플러그인 진입점 심볼 / Observer 등록** 중 하나로 추상화됨.
- 리디렉션 채널(cliprdr/rdpdr/rdpsnd/rail)이 공통 통제 대상이며, 켜고 끄는 스위치가 반드시 존재.
- 보안 협상 결과에 따라 능력이 달라지고, 운영자 설정으로 최소 수준 강제 가능.

### 9.2 차이
| 항목 | xrdp | FreeRDP | pyrdp |
|---|---|---|---|
| 역할 | 서버 | 라이브러리/클라/서버/프록시 | MITM/녹화 |
| 확장 추상화 | owner 포인터 체인 + mod 테이블 | ALIGN64 콜백 테이블 + settings 키 | Observer/Layer 체인 |
| 서버측 NLA | VMConnect 외 미지원 | 완전 지원(nla.c) | 중계만(복호 불가) |
| 프로세스 모델 | 연결당 fork + 권한분리 | 프록시=스레드 | reactor 단일 |
| 인증 위치 | sesman/PAM 별도 프로세스 | 콜백/CredSSP in-proc | 가로채기 대상 |
| 채널 기본정책 | 화이트리스트([Channels]) | 혼합(clip/rail만 차단) | 전량 가로채기 |
| 클립보드 기본 | 양방향 허용(제한 0) | 클라 TRUE/프록시 FALSE | 가로채기+탈취 |
| 인증서 검증 | 서버라 해당없음 / neutrinordp 무검증 | 기본 검증+프롬프트, 프록시 기본 DENY | 복제(MITM) |

### 9.3 유추 요구사항 — 기능 (Functional)
- F1. 다중 세션 백엔드(Xorg/VNC/원격RDP 등)를 플러그인 인터페이스로 수용. [xrdp `xrdp_types.h:51`, `xrdp.ini.in:263~347`]
- F2. 정적/동적 가상 채널 등록 및 채널별 라우팅. [FreeRDP `svc.h`/`dvc.h`, xrdp `chansrv.c:433~`]
- F3. 리디렉션 기능 세트: 클립보드(방향·타입별), 드라이브, 프린터, 스마트카드, 오디오 입·출력, USB, 카메라, 원격앱(RAIL), 동적 해상도. [FreeRDP `channels/`, xrdp `devredir.c`, proxy `[Channels]`]
- F4. 멀티 모니터 및 세션 중 동적 리사이즈. [xrdp `dynamic_monitor_*` `xrdp_mm.c:1551`, FreeRDP disp]
- F5. 세션 재접속/연결 유지(reconnect script, auto-reconnect). [sesman.ini `ReconnectScript`, FreeRDP `AutoReconnectionEnabled`]
- F6. 게이트웨이/프록시로 중계 및 타깃 주소 결정 훅. [FreeRDP proxy `ServerFetchTargetAddr`, xrdp gateway `MMCS_GATEWAY_LOGIN`]
- F7. 세션 녹화·재생·감사(합법적 감사용). [pyrdp recording, FreeRDP streamdump/dyn-channel-dump]

### 9.4 유추 요구사항 — 비기능 (Non-functional)
- N1. 연결 격리(연결당 프로세스/스레드) + 비권한 사용자 실행 + 권한 분리. [xrdp fork `xrdp_listen.c:685` + `g_drop_privileges` `xrdp.c:458`]
- N2. 플랫폼 추상화 계층(OS API 래핑). [FreeRDP winpr, xrdp common/]
- N3. 코덱 협상 및 런타임 능력 탐지 후 안전 fallback. [xrdp `libh264_loaded` fallback, `xrdp_encoder.c:187`]
- N4. 네트워크 대역폭 자동탐지 및 연결타입별 품질 프로파일. [FreeRDP `rdpAutoDetect`, xrdp gfx.toml `[x264.wan]` 등]
- N5. ABI/버전 호환성 설계(구조체 패딩·오프셋·버전 필드). [xrdp `mod_dumby`, FreeRDP ALIGN64+offset, proxy plugin reserved]
- N6. 설정의 외부화(ini/toml) 및 세션 타입별 override. [xrdp.ini/sesman.ini/gfx.toml, FreeRDP settings 키, proxy ini]
- N7. 구조화된 로깅/메트릭(연결·인증·채널 이벤트). [xrdp LOG+syslog, FreeRDP WLog/metrics, pyrdp StatCounter]

### 9.5 유추 요구사항 — 보안 (Security)
- S1. **강한 보안계층 강제 옵션**: 서버가 TLS/NLA 미만을 거부할 수 있어야. classic RDP는 기본 비권장/FIPS 거부. [xrdp `security_layer=tls` `xrdp_iso.c:185`, FIPS `:195`; FreeRDP 서버 nego `connection.c:1651`]
- S2. **NLA(CredSSP) 지원 및 서버 공개키 바인딩**: 중간자 차단의 핵심. [FreeRDP `nla.c:1111` 공개키 echo 검증; pyrdp가 NLA 시 실시간 MITM 불가함을 역증명 `X224MITM.py:126`]
- S3. **TLS 버전·암호군 하한 설정, 안전한 기본값**: TLS1.2+ 기본, 레거시 비활성. [xrdp `ssl_protocols=TLSv1.2,TLSv1.3` `xrdp.ini.in:86`; FreeRDP `TLSMinVersion=TLS1_2` `settings.c:1443`, `TlsSecLevel=112BIT`]
- S4. **인증서 검증을 기본 수행, 무검증은 명시 opt-in + 경고**: [FreeRDP `IgnoreCertificate` 경고 `tls.c:1854`, proxy `TargetCertPolicy` 기본 DENY; 반례 neutrinordp 무검증 `:2497`을 제거/경고 대상으로]
- S5. **리디렉션 채널 기본 차단 가능 + 방향·타입별 세밀 통제**: 특히 클립보드 inbound/outbound와 드라이브. [xrdp `RestrictInbound/OutboundClipboard`, `[Channels]` 화이트리스트; FreeRDP proxy Clipboard 기본 FALSE] — **주의**: xrdp 클립보드 기본은 제한 없음(`sesman_config.c:321`)이므로 하드닝 가이드 필요.
- S6. **인증 재시도 제한 + 계정 잠금 + 그룹 기반 접근제어 + root 로그인 차단**: [xrdp `MaxLoginRetry` `login_info.c:272`, `access_login_allowed` `sesman_access.c:54`, `AllowRootLogin`; FreeRDP `STATUS_ACCOUNT_LOCKED_OUT`]
- S7. **자격증명 처리 시 정보 유출 방지 및 평문 노출 최소화**: [xrdp "might lead to information leakage" `scp_process.c:116`; pyrdp가 CLIENT_INFO로 평문 username/password를 캡처함 `SecurityMITM.py:74` → 역으로 전송구간 보호 필수]
- S8. **세션 타임아웃/유휴·끊김 정책 및 동시 세션 제한 설정화 + 실제 enforcement**: [sesman `IdleTimeLimit`/`KillDisconnected`; 단 xrdp는 enforcement를 세션 스크립트에 위임하므로(`session.c:416`) 제품은 서버측 강제 필요]
- S9. **감사/녹화 지점의 접근통제 + TLS keylog류 디버그 기능의 운영 비활성**: [xrdp `tls_pms_log_file`·`ssl_calls.c:1264` keylog, FreeRDP `TransportDump`/SSLKEYLOGFILE — 민감자료이므로 보호]
- S10. **플러그인/모듈 로딩 경로 제한(full-path 금지, 설치경로만)**: [FreeRDP addin/proxy module `README.md` "loading by full path is not supported"]
- S11. **MITM 위협 모델 명시**: NLA 미사용 + 인증서 미검증 + TLS1.3 미지원 조합에서 키입력·자격증명·클립보드·파일·화면 전량이 노출됨. [pyrdp 전체가 근거; `enum/player.py:15~34`가 노출 자료 목록]

---

## 10. 미확인 항목
- xrdp 서버측 USB(urbdrc) 리디렉션: chansrv에 핸들러 미발견(드라이브/스마트카드만 처리). 지원 여부 미확인.
- xrdp `AllowRootLogin` 기본값: 코드 기본 0(불허, `sesman_config.c:318`)과 배포 템플릿 `true`(`sesman.ini.in:19`)가 상이 — 실제 운영 기본은 설치본에 의존, 미확인.
- FreeRDP `rdpUpdate`/`rdpInput`의 전체 서버측 페이로드 처리 세부는 핵심 콜백만 확인, 전수 미확인.
- RDSTLS/AAD(RDSAAD) 세부 핸드셰이크 흐름: 파일 존재(`rdstls.c`/`aad.c`)와 nego 비트만 확인, 내부 로직 미확인.
- xrdp 세션 타임아웃의 실제 종료 주체(세션 스크립트 `startwm.sh`/WM 내부): 환경변수 전달까지만 확인, 스크립트 내부 미확인.
- TLS pre-master/keylog 기능이 운영 기본에서 꺼져 있는지: 설정 주석상 기본 비활성(주석 처리)으로 보이나 배포본 미확인.

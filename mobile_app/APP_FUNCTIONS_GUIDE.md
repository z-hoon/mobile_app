# 📱 Smart Diffuser 앱(Mobile App) 함수 정리 및 연관 시스템 사양서

본 문서는 **Smart Diffuser (AromaSync)** 앱(React / TypeScript / Capacitor) 내부에서 사용되는 모든 함수, 훅(Hooks), 상태 관리 로직을 정리한 사양서입니다.  
펌웨어 및 하드웨어 자체의 코드 함수는 직접 다루지 않으나, **각 앱 함수가 하드웨어(Hardware), 펌웨어(Firmware), 클라우드/백엔드(Cloud/Backend), 앱 UI/상태(App UI/State) 중 어느 영역과 연관되어 작동하는지** 명확히 기술합니다.

---

## 🏗️ 연관 구성요소 정의

- 🤖 **하드웨어 (Hardware)**: 4종 카트리지 노즐(솔레노이드 밸브), 에어 펌프, 팬, RGB LED 스트립, I2S 오디오 스피커, 무게 측정 로드셀 Sensor (HX711), 마이크로폰
- ⚙️ **펌웨어 (Firmware / ESP32)**: 노즐 제어 및 안전 상태 머신, MQTT/HTTP Mailbox 통신, 무게 센서 EMA 필터링, I2S 소음 분석, 타임아웃 & RTC 스케줄 제어, NVS 초기화
- ☁️ **클라우드/백엔드 (Cloud / AWS Lambda & DynamoDB)**: 
  - `device_handler.py` (기기 상태 폴링 및 Mailbox 명령 등록)
  - `voice_handler.py` (Gemini AI 기반 음성 분석 및 향기/LED/음악 추천)
  - `core_mode_handler.py` (날씨 API 연동, 감정 분석 AI 모드, 취향 리포트 생성)
  - `feedback_handler.py` (좋아요/싫어요 피드백 수집 및 개인화 모델 학습)
- 📱 **앱 UI 및 상태 (App UI / State)**: React Context API, LocalStorage, React Router, UI 컴포넌트 이벤트 핸들러

---

## 1. 🌐 네트워크 & API 통신 계층 (`api.ts`)

CapacitorHttp를 사용하여 AWS Lambda 백엔드 서버와 통신하는 API 함수 모음입니다.

| 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `requestApi(data)` | CapacitorHttp를 사용한 공통 POST API 요청 처리 함수 | ☁️ Cloud (AWS Lambda URL) |
| `apiSendVoiceData(base64Audio, deviceId, contentType)` | 녹음된 음성 바이너리(Base64)를 AWS Lambda로 전송 | 🤖 Hardware (마이크)<br>☁️ Cloud (`voice_handler` / Gemini AI) |
| `apiLogin(email, pass)` | 사용자 로그인 검증 (`action: LOGIN`) | ☁️ Cloud (DynamoDB 회원 DB) |
| `apiSignup(email, pass, region, authCode, deviceId)` | 신규 회원가입 및 기기 ID 바인딩 (`action: SIGNUP`) | ☁️ Cloud (DynamoDB 회원/기기 DB) |
| `apiSendAuthCode(email)` | 이메일 인증번호 발송 요청 (`action: SEND_AUTH`) | ☁️ Cloud (AWS SES / Auth) |
| `apiUpdateUser(email, oldPass, newPass, region)` | 사용자 비밀번호 및 위치 정보 수정 (`action: UPDATE_USER`) | ☁️ Cloud (DynamoDB) |
| `apiPollDeviceState(email, deviceId)` | 5초 간격으로 실시간 기기 상태(무게, LED, dB, 모드) 조회 (`action: POLL`) | ⚙️ Firmware (ESP32 Mailbox)<br>☁️ Cloud (`device_handler`)<br>🤖 Hardware (무게/소음 센서, LED, 노즐) |
| `apiSendData(options)` | 수동 제어, 모드 전환, LED, 음량, 타이머, 피드백 명령 전송 | ☁️ Cloud (Lambda Handlers)<br>⚙️ Firmware (ESP32 Mailbox 명령 수신)<br>🤖 Hardware (솔레노이드 밸브, 펌프, 팬, LED, 스피커) |
| `apiFetchDiaries(email)` | 저장된 일기 및 AI 분석 내역 목록 조회 (`action: GET_DIARIES`) | ☁️ Cloud (DynamoDB 일기 DB) |

---

## 2. 🔑 인증 전역 상태 관리 (`AuthContext.tsx`)

사용자 세션 및 회원 로그인/가입 상태를 전역 관리하는 컨텍스트입니다.

| 함수명 / Hook | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `AuthProvider` | 로그인 상태 유지 및 LocalStorage 동기화 프로바이더 | 📱 App UI / State |
| `registerUser(email, pass, region, code, deviceId)` | 회원가입 API 호출 및 성공 시 사용자 로컬 등록 | ☁️ Cloud (DynamoDB)<br>📱 App State |
| `loginUser(email, pass)` | 로그인 API 호출 및 세션 로컬 저장 | ☁️ Cloud (DynamoDB)<br>📱 App State (LocalStorage) |
| `logoutUser()` | 세션 삭제 및 로그인 정보 초기화 | 📱 App State (LocalStorage) |
| `updateUser(email, data, oldPass)` | 클라우드 사용자 정보 업데이트 및 로컬 상태 반영 | ☁️ Cloud (DynamoDB)<br>📱 App State |
| `updateLocalUser(data)` | API 응답으로 전달된 로컬 정보(음악 트랙 등) 세션 업데이트 | 📱 App State |
| `useAuth()` | 인증 컨텍스트 호출 Custom Hook | 📱 App UI / State |

---

## 3. 🔌 기기 상태 & 제어 센터 (`DeviceContext.tsx`)

디퓨저의 모든 센서 데이터, 실시간 5초 폴링, 제어 명령 전송을 총괄하는 핵심 컨텍스트입니다.

| 함수명 / Hook | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `DeviceProvider` | 기기 전역 상태 및 5초 주기 폴링(`useEffect`) 관리 | 📱 App State<br>⚙️ Firmware (ESP32 상태 동기화) |
| `refreshDeviceState()` | 5초마다 기기 상태 조회, 통신 상태(Wi-Fi 강도) 계산, 카트리지 무게% 및 잔량(g) 동기화, 카트리지 종류 매핑, 소음(dB/표준편차/Spike) 파싱, active_mode/LED/음악 코드 파싱 | ⚙️ Firmware (ESP32 상태 파싱)<br>☁️ Cloud (DynamoDB)<br>🤖 Hardware (로드셀, 마이크, LED, 노즐) |
| `sendDeviceData(action, val, region, diary, led)` | 사용자 명령(모드 변경, 정지, LED, 수동분사 등)을 백엔드로 전달하고 로컬 UI 상태 즉시 갱신 | ☁️ Cloud (AWS Lambda)<br>⚙️ Firmware (ESP32 모터/LED/음악 실행)<br>🤖 Hardware (솔레노이드 밸브, 펌프, 팬, LED, 스피커) |
| `handleVolumeChange(newVol)` | 오디오 음량(0~10) 조절 명령 전송 (`SET_VOLUME`) | 🤖 Hardware (I2S 오디오 스피커)<br>⚙️ Firmware (ESP32 DAC/Volume)<br>☁️ Cloud |
| `handleIntensityChange(newLevel)` | 분사 농도/강도(0~100) 조절 명령 전송 (`SET_INTENSITY`) | 🤖 Hardware (솔레노이드 밸브 PWM / 에어 펌프)<br>⚙️ Firmware (분사 시간 타이머) |
| `updateTimerSettings(enabled, start, end)` | 자동 스케줄러 (시작/종료 시간) 설정 전송 | ⚙️ Firmware (ESP32 RTC/스케줄러)<br>☁️ Cloud |
| `handleTimerChange(minutes)` | 수동 수면/동작 타이머(분) 설정 또는 정지 | ⚙️ Firmware (ESP32 1회성 타이머)<br>☁️ Cloud |
| `updateScentSlot(id, slot)` | 앱 화면 상의 카트리지 슬롯 정보(이름, 색상) 수정 | 📱 App State (LocalStorage) |
| `calibrateWeight()` | 무게 센서 영점 조절 명령 전송 (`CALIBRATE`) | 🤖 Hardware (HX711 로드셀 무게 센서)<br>⚙️ Firmware (ESP32 센서 Calibration) |
| `useDevice()` | 기기 제어 컨텍스트 호출 Custom Hook | 📱 App UI / State |

---

## 4. 📖 일기장 & UI 전역 상태 (`DiaryContext.tsx`, `UIContext.tsx`)

| 파일 및 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `DiaryProvider` | 일기 기록 전역 프로바이더 | 📱 App State |
| `addDiaryEntry(email, entry)` | 로컬 일기 목록에 항목 추가 | 📱 App State |
| `fetchDiaries(email)` | 서버에서 일기 히스토리 가져오기 | ☁️ Cloud (AWS Lambda / DynamoDB) |
| `useDiary()` | 일기 컨텍스트 호출 Hook | 📱 App UI / State |
| `UIProvider` | 음악 선택 등 UI 임시 상태 프로바이더 | 📱 App State |
| `useUI()` | UI 컨텍스트 호출 Hook | 📱 App UI / State |

---

## 5. 🏠 메인 스마트 모드 화면 (`ModesScreen.tsx`)

5대 스마트 모드(날씨, 감정, AI 리플레이, 소음 센싱, 수동) 제어 및 실시간 소음 그래프, 음성 명령 화면입니다.

| 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `fetchWeatherPreview()` | 현재 설정 지역의 날씨 정보 미리보기 요청 | ☁️ Cloud (OpenWeather API / AWS Lambda) |
| `handlePowerOff()` | 전원 OFF (모든 노즐, 펌프, LED, 오디오 즉시 정지) | 🤖 Hardware (모든 구동부)<br>⚙️ Firmware (ESP32 안전 정지)<br>☁️ Cloud |
| `toggleVoiceRecording()` | 음성 녹음 시작/중지 (`MediaRecorder`), Base64 변환 후 `apiSendVoiceData` 전송, AI 분석 결과에 따른 향기 분사/LED 구동 | 🤖 Hardware (마이크, 솔레노이드 밸브, LED)<br>☁️ Cloud (Gemini AI Voice Analysis)<br>⚙️ Firmware |
| `handleModeClick(modeId)` | 스마트 모드(날씨/감정/소음/수동) 변경 명령 전송 | ☁️ Cloud (Core Mode Handlers)<br>⚙️ Firmware (ESP32 상태머신) |
| `handleApplyManual()` | 수동 조합 분사(단일 및 블렌딩 믹스) 명령 전송 | 🤖 Hardware (솔레노이드 밸브 1~4, 에어 펌프)<br>⚙️ Firmware |
| `handleFeedback(type)` | AI 추천에 대해 '좋아요/별로예요' 피드백 전송. '별로예요' 선택 시 3초 후 대체 향기로 자동 교체 분사 | ☁️ Cloud (`feedback_handler.py` AI 학습)<br>⚙️ Firmware (노즐 전환)<br>🤖 Hardware (솔레노이드 밸브) |
| `handleApplyTimer()` | 커스텀 시간/분 타이머 설정 모달 완료 처리 | ⚙️ Firmware (ESP32 타임아웃 카운터) |

---

## 6. 📔 감정 일기장 & AI 취향 리포트 (`DiaryScreen.tsx`, `AiTasteReportScreen.tsx`)

| 파일 및 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `getEmotionTheme(emotion)` | 감정(신남/화남/슬픔/편안)별 LED Hex 색상, 추천 음악 장르, UI 테마 매핑 | 🤖 Hardware (RGB LED 색상 매핑)<br>📱 App UI |
| `getActualScentName(code)` | 분사 코드(단일 1~4, 블렌딩 12 등)를 실제 향기 이름("시트러스 & 센달우드")으로 변환 | 🤖 Hardware (4종 카트리지 노즐 매핑) |
| `handleSubmit(e)` | 3줄 일기 텍스트 제출 ➡️ AI 감정 분석, 맞춤 향기 분사, LED 색상 및 음악 재생 명령 전달 | ☁️ Cloud (AWS Lambda Gemini AI)<br>⚙️ Firmware (ESP32 분사 및 LED)<br>🤖 Hardware (노즐, LED, 스피커) |
| `handleDiaryFeedback(type)` | 일기 분석 결과 피드백. '별로예요' 시 ESP32 모터 정지(3초 대기) 후 즉시 대체 향기로 전환 분사 | ☁️ Cloud (`feedback_handler.py`)<br>⚙️ Firmware (ESP32 모터 멈춤/재가동)<br>🤖 Hardware (솔레노이드 밸브) |
| `fetchReport()` | 사용자의 누적 피드백 기반 AI 취향 리포트(선호 향기, 날씨/감정별 믹스) 가져오기 | ☁️ Cloud (AWS Lambda 취향 분석 엔진) |
| `getContextIcon(label)` | 리포트 내 날씨/감정 태그에 대응하는 UI 아이콘 반환 | 📱 App UI |

---

## 7. 💡 LED 조명 제어 (`LedSettingsScreen.tsx`)

무드등 LED의 전원, 컬러 휠(Color Wheel), 밝기 슬라이더 제어 화면입니다.

| 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `hexToRgb(hex)` | Hex 색상 코드를 RGB로 변환하고 **Gamma 2.0 보정**을 적용하여 실물 LED 발색을 선명하게 개선 | 🤖 Hardware (RGB LED 발색 교정)<br>⚙️ Firmware (PWM 출력) |
| `hslToHex(h, s, l)` | HSL 컬러 휠 값(각도/채도/밝기)을 Hex 코드 문자열로 변환 | 📱 App UI Color Math |
| `hexToThumbPos(hex)` | Hex 색상 값을 컬러 휠 상의 (x, y) 포인터 좌표로 역산 | 📱 App UI Color Wheel |
| `ColorWheel(...)` | 터치/드래그 포인터 이벤트를 감지하여 실시간 색상을 선출하는 원형 컬러 휠 컴포넌트 | 📱 App UI Component |
| `syncWithHardware(...)` | 300ms 디바운스를 적용하여 사용자가 선택한 RGB, 밝기(0~255), 전원 상태를 기기로 전송 (`SET_LED`) | 🤖 Hardware (RGB LED Strip)<br>⚙️ Firmware (ESP32 LED Controller)<br>☁️ Cloud |

---

## 8. 🎵 음악 & 카트리지 재생 목록 (`MusicSettingsScreen.tsx`)

| 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `openSelectModal(slotId)` | 특정 향기 슬롯(1~4번)의 음원 선택 모달 열기 | 📱 App UI |
| `toggleTrackForActiveSlot(...)` | 슬롯별 재생할 다중 음원 아이템 선택/해제 | 📱 App UI |
| `moveTrackUp()` / `moveTrackDown()` | 슬롯 내 음원 연속 재생 순서(▲/▼) 변경 | 📱 App UI |
| `removeTrackFromSlot(...)` | 재생 목록에서 선택 음원 삭제 | 📱 App UI |
| `handleSave()` | 4개 슬롯의 재생 음원 트랙 번호를 직렬화(`1,2_6,7_11_16`)하여 백엔드 및 로컬 회원 프로필에 저장 | 🤖 Hardware (I2S 오디오 스피커)<br>⚙️ Firmware (ESP32 SD/DFPlayer 재생 큐)<br>☁️ Cloud (DynamoDB) |

---

## 9. 🧪 카트리지 잔량 & 무게 관리 (`ScentsScreen.tsx`)

| 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `handleReplace(newName, newColor)` | 새 향기 카트리지 교체 등록 (잔량 100% 리셋), 슬롯-향기 종류 매핑 딕셔너리 구성 후 `SET_MAPPING` 전송 | 🤖 Hardware (4종 카트리지 물리 교체 및 솔레노이드 밸브 매핑)<br>⚙️ Firmware (ESP32 Nozzle Mapping)<br>☁️ Cloud (DynamoDB) |

---

## 10. 🔒 개인정보 & 계정 수정 (`PrivacyScreen.tsx`)

| 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `handleSave()` | 기존 비밀번호 검증 후 새 비밀번호 및 거주 지역 정보 수정 요청 | ☁️ Cloud (AWS Lambda `UPDATE_USER` / DynamoDB)<br>📱 App State |

---

## 11. 🔧 기기 유지보수 & 공장 초기화 (`DeviceManagementScreen.tsx`)

| 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `handleWifiReset()` | 기기의 Wi-Fi 설정을 삭제하고 재부팅하도록 `WIFI_RESET` 명령 전송 | ⚙️ Firmware (ESP32 NVS 메모리 삭제 / AP 설정모드 진입 / 재부팅) |
| `handleFactoryReset()` | 클라우드 DB의 모든 취향 데이터, 일기 기록, 음악 스케줄을 영구 삭제하는 `RESET_DATABASE` 전송 | ☁️ Cloud (DynamoDB 테이블 초기화)<br>⚙️ Firmware (기기 설정 초기화) |
| `saveDeviceId()` | 사용자 계정에 바인딩된 물리 기기 ID(Device ID) 업데이트 | ☁️ Cloud (DynamoDB User Device Binding) |

---

## 12. ⚙️ 설정 & 계정 관리 (`SettingsScreen.tsx`, `Login.tsx`, `SignupScreen.tsx`)

| 파일 및 함수명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `handleLogout()` | 로컬 세션 제거 및 로그인 화면 이동 | 📱 App State |
| `handleLogin(e)` | 이메일/비밀번호 로그인 처리 | ☁️ Cloud (AWS Lambda Auth) |
| `handleSendCode()` | 회원가입 시 이메일 인증코드 6자리 발송 | ☁️ Cloud (AWS Lambda SES) |
| `handleSignup(e)` | 이메일 인증 확인 후 신규 회원가입 및 Device ID 바인딩 완료 | ☁️ Cloud (AWS Lambda / DynamoDB) |

---

## 13. 🎨 UI 컴포넌트 & 데이터 유틸리티 (`tracks.ts`, `ModeButton.tsx`, `BottomNav.tsx`, `routes.tsx`)

| 파일 및 함수/컴포넌트명 | 설명 및 주요 역할 | 연관 구성요소 (Association) |
| :--- | :--- | :--- |
| `TRACKS` (`tracks.ts`) | 앱 전체에서 참조하는 26종 음원 트랙 마스터 데이터 구조체 | 📱 App UI<br>🤖 Hardware (I2S 오디오 매핑) |
| `ModeButton(...)` | 스마트 모드 선택 카드 버튼 UI 컴포넌트 | 📱 App UI |
| `BottomNav()` | 하단 고정 탭 네비게이션 컴포넌트 | 📱 App UI Navigation |
| `Root()` / `router` (`routes.tsx`) | React Router 라우팅 경로 설정 및 페이지 이동 시 스크롤 상단 리셋 | 📱 App UI Routing |

---

## 🧭 요약 시스템 데이터 흐름도

```
[ 📱 Mobile App ]
      │  
      │ 1. API Call (apiSendData, apiPollDeviceState)
      ▼
[ ☁️ AWS Lambda Backend ]
      │  
      │ 2. Mailbox Pattern & State DB (DynamoDB)
      ▼
[ ⚙️ ESP32 Firmware ]
      │  
      │ 3. Hardware Actuators & Sensors Control
      ▼
[ 🤖 Hardware ] ──▶ (Solenoid Valves 1~4 / Air Pump / RGB LED / Speaker / Load Cell Sensor)
```

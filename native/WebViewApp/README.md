# WebViewApp — 로컬 웹앱 테스트용 Android 앱

맥에서 실행 중인 Vite(React) 웹앱과 Spring Boot API를 테스트폰에서 직접 확인하기 위한 최소 WebView 앱입니다.

---

## 사전 조건

| 항목 | 확인 방법 |
|---|---|
| USB 디버깅 활성화 | 설정 → 개발자 옵션 → USB 디버깅 ON |
| ADB 연결 | `adb devices` → `device` 상태 확인 |
| 맥에서 Vite 실행 | `npm run dev` (기본 포트 5173) |
| 맥에서 Spring Boot 실행 | `./gradlew bootRun` (기본 포트 8080) |

---

## 1. 포트 포워딩 (adb reverse)

테스트폰이 맥의 로컬호스트에 접근하도록 포워딩합니다.

```bash
# Vite 개발 서버
adb reverse tcp:5173 tcp:5173

# Spring Boot (Vite의 /api 프록시를 사용하면 불필요)
adb reverse tcp:8080 tcp:8080

# 설정 확인
adb reverse --list
```

> **Vite 프록시 사용 중이라면** `5173` 포워딩만으로 충분합니다.
> 웹앱이 `http://localhost:8080`을 직접 호출한다면 `8080`도 추가합니다.

---

## 2. APK 빌드 및 설치

```bash
# 프로젝트 루트에서 실행
cd native/WebViewApp

# 디버그 APK 빌드
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home ./gradlew assembleDebug

# 연결된 테스트폰에 설치
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home ./gradlew installDebug
```

생성된 APK 경로:
```
app/build/outputs/apk/debug/app-debug.apk
```

---

## 3. 앱 사용 방법

1. 앱 실행 → 상단 URL 입력란에 `http://localhost:5173` 입력
2. **열기** 버튼 또는 키보드 이동(Go) 키 → 웹앱 로드
3. **새로고침** 버튼 → 현재 페이지 새로고침
4. 안드로이드 뒤로가기 → WebView 방문 기록 뒤로 이동 (기록 없으면 앱 종료)
5. 마지막으로 연 URL은 자동 저장되어 앱 재실행 시 복원됩니다.
6. 로드 실패 시 화면에 URL과 오류 코드가 표시되며 **다시 시도** 버튼으로 재시도합니다.

---

## 4. API 호출 실패 시 디버깅

### Chrome DevTools (WebView 원격 디버깅)
디버그 빌드에서만 활성화됩니다.

1. 테스트폰을 맥에 USB로 연결
2. 맥 Chrome 주소창에 `chrome://inspect/#devices` 입력
3. WebViewApp의 WebView 항목에서 **inspect** 클릭
4. Network 탭에서 요청 URL, HTTP 상태 코드, 응답 헤더 확인

### 확인 항목

| 증상 | 확인 사항 |
|---|---|
| 페이지 로딩 불가 | `adb reverse` 포워딩 여부, Vite 실행 여부 |
| API 응답 없음 | Spring Boot 실행 여부, `8080` 포워딩 여부 |
| CORS 오류 | Spring Boot의 CORS 설정 (`@CrossOrigin` 또는 WebMvcConfigurer) |
| `net::ERR_CLEARTEXT_NOT_PERMITTED` | `adb reverse` 미설정 또는 HTTP URL을 릴리즈 빌드로 접근 |

### Logcat 확인
```bash
adb logcat | grep -E "WebView|Chromium|CORS"
```

---

## 디버그 빌드 전용 설정

| 기능 | 적용 조건 |
|---|---|
| `localhost` HTTP(cleartext) 허용 | 디버그 빌드 (`src/debug/res/xml/network_security_config.xml`) |
| Chrome DevTools WebView 디버깅 | 디버그 빌드 (`BuildConfig.DEBUG` 조건부) |
| 릴리즈 빌드 | cleartext 차단, DevTools 비활성화 |

# CLAUDE.md

## 프로젝트 개요

BankEx는 로그인, 계좌, 거래, 대출을 다루는 모의 은행 서비스다.
Android 앱에서 React 웹앱을 WebView로 열고, Spring Boot API와 연동한다.

| 영역 | 기술 | 경로 |
|---|---|---|
| Android | Kotlin, Android SDK, WebView, Gradle | 기존 Android 모듈을 우선 사용. 신규 생성 시 `bankEx/android/`를 기본 후보로 사용 |
| 프론트엔드 | React 19, TypeScript, Vite, Axios | `bankEx/front/bankEx_Front/` |
| 백엔드 | Spring Boot 4.0.5, Java 17, Gradle | `bankEx/backend/bankEx/` |
| 데이터베이스 | Oracle XE, JPA, MyBatis | 백엔드 `src/main/resources/application.yml`에서 설정 확인 |

위 경로는 저장소 루트 기준이다. 작업 전에 실제 모듈 경로, 의존성 버전, 구현 상태를 확인한다. 기존 모듈이 있다면 해당 모듈의 빌드 설정, 패키지 이름, UI 구성 방식을 재사용한다.

## 아키텍처와 개발 기준

### Android · Kotlin

- Android 앱은 WebView의 로딩, 화면 이동, 로드 오류, 네이티브 생명주기를 담당한다. 은행 업무 규칙은 Spring Boot에 둔다.
- 테스트 앱 작업에서는 기존 명세에 따라 URL 입력, 열기, 새로고침, 마지막 URL 복원, WebView 뒤로가기, 메인 페이지 로드 오류 표시를 구현한다.
- React 화면 구동에 필요한 JavaScript와 DOM Storage를 활성화한다. URL 입력은 `http`와 `https` 스킴으로 제한한다.
- 로컬 HTTP 허용과 WebView 원격 디버깅은 디버그 빌드에만 적용한다. TLS 인증서 오류는 기본 검증 흐름을 유지한다.
- 네이티브와 웹 사이의 데이터 전달이 실제 요구사항에 포함되면, 호출 가능한 웹 출처와 메시지 스키마를 명시하고 입력을 검증한다.

### 프론트엔드 · React + TypeScript

- `src/api/axios.ts`에서 Axios 인스턴스와 API 기본 경로를 관리한다. `src/api/bank_api.ts`의 요청·응답 타입은 실제 백엔드 DTO와 맞춘다.
- `src/router/path.ts`에서 경로 상수를, `src/router/index_router.tsx`에서 라우트 등록을 확인한다. `src/pages/`는 페이지, `src/components/`는 재사용 UI를 담당한다.
- API 주소는 기존 환경 설정을 우선 사용한다. 로컬 WebView 연동을 새로 구성한다면 Vite의 `/api` 프록시와 상대 경로 호출을 우선 검토한다. 프록시 구성은 개발 서버에 적용되므로 배포 환경의 `/api` 라우팅은 별도로 확인한다.
- 대출 화면을 수정할 때는 `src/pages/loan.tsx`의 존재 여부, 라우트 등록, 상대 import가 현재 코드와 맞는지 확인한다.

### 백엔드 · Spring Boot

- `/api/bank/user`를 기존 API 기본 경로로 보고 실제 컨트롤러 매핑을 확인한다.
- 컨트롤러는 HTTP 요청·응답과 입력 검증을 담당한다. 서비스는 업무 규칙과 트랜잭션을 담당한다. 데이터 접근은 해당 기능에서 이미 사용 중인 JPA 리포지터리 또는 MyBatis 매퍼를 따른다.
- 로그인·계좌·거래·대출 작업은 현재 엔드포인트, DTO, 서비스, 테스트를 먼저 확인한다. 기존 API 계약을 변경해야 하면 React 호출부와 영향받는 Android 연동도 함께 확인한다.
- 로컬 Oracle XE 예시는 `localhost:1521/XE`, 스키마는 `hanati`다. 실제 접속 정보는 로컬 설정을 확인한다. 테스트에는 가상 고객·계좌 데이터를 사용하고 비밀번호와 토큰은 커밋하거나 로그에 남기지 않는다.

## 빌드 및 검증 명령

각 명령은 해당 모듈 디렉터리에서 실행한다.

### Android

```bash
./gradlew assembleDebug  # 디버그 APK 생성
./gradlew installDebug   # 연결된 테스트폰에 설치
```

기존 빌드 변형이나 모듈 설정에 따라 task 이름이 다르면 실제 Gradle task를 사용한다. 완료 보고에 생성된 APK 경로를 적는다.

### 프론트엔드

```bash
npm run dev      # 로컬 개발 서버, 기본 5173
npm run build    # TypeScript 검사 및 Vite 빌드
npm run lint     # ESLint
npm run preview  # 빌드 결과 미리보기
```

### 백엔드

```bash
./gradlew bootRun
./gradlew build
./gradlew test
./gradlew test --tests "com.hanati.bank.bankEx.SomeTest"
```

`SomeTest`는 예시이므로 실제 테스트 클래스명으로 바꾼다. 백엔드 예시 포트는 `8080`이며, 실행 설정을 우선한다.

## Android 실기기 로컬 연동

1. 맥에서 Spring Boot와 Vite를 실행한다. USB 디버깅을 켠 테스트폰을 연결하고 `adb devices`에서 상태가 `device`인지 확인한다.
2. 테스트폰의 웹 포트를 맥으로 전달한다.

   ```bash
   adb reverse tcp:5173 tcp:5173
   adb reverse --list
   ```

3. 테스트폰 Chrome에서 `http://localhost:5173`을 열고, Android 앱의 WebView에서 같은 주소를 연다.
4. Vite가 `/api` 요청을 Spring Boot로 프록시하면 `5173`만 전달한다. 웹앱이나 Kotlin 코드가 `http://localhost:8080`을 직접 호출하면 `adb reverse tcp:8080 tcp:8080`도 실행한다. WebView의 JavaScript가 `5173`에서 `8080`으로 직접 요청하는 경우에는 백엔드 CORS 설정도 확인한다.
5. 테스트 데이터로 화면 로딩, 새로고침, 뒤로가기, 메인 페이지 로드 실패를 확인한다. API 변경이 있으면 영향받는 사용자 흐름과 응답을 확인한다. 실패 시 WebView DevTools 또는 Logcat에서 요청 URL, HTTP 상태 코드, 오류 내용을 확인한다.

테스트폰 연결이나 DB 준비가 안 된 경우에는 실행 가능한 빌드와 테스트를 먼저 수행하고, 실기기·DB 연동 검증은 미실행 사유를 기록한다.

## 작업 원칙

1. **코드부터 확인한다.** 요청과 관련된 진입점, 호출부, 설정, 테스트를 읽는다. 결과에 영향을 주는 가정은 명시한다. 되돌릴 수 있는 구현 선택은 기존 패턴에 맞춰 진행하고, 외부 API 계약이나 데이터 모델을 크게 바꾸는 결정이 불명확할 때는 질문한다.
2. **요구사항을 충족하는 최소 변경을 한다.** 기존 구조와 스타일을 따른다. Android 테스트 앱 작업은 Android 모듈을 중심으로 진행하며, 웹이나 서버 변경이 필요하면 구체적인 이유와 영향 범위를 먼저 설명한다.
3. **계약이 바뀌면 연결된 계층을 확인한다.** API 경로, 요청·응답 필드, 오류 처리 변경은 React 호출부까지 추적한다. WebView URL, 네이티브 브리지, 세션 동작에 영향이 있으면 Android도 확인한다.
4. **완료 기준을 검증한다.** 작업 규모가 크면 변경 단계마다 확인 방법을 적는다. 기존 테스트를 활용하고, 기존 검증으로 놓칠 회귀가 있으면 해당 사례를 테스트로 추가한다. 관련 없는 코드 정리나 포맷 변경은 범위에서 제외한다.
5. **검증 결과를 보고한다.** 변경 파일, 실행한 명령과 결과, 미실행 항목과 사유를 적는다. Android 변경 시 APK 경로와 테스트폰에서 확인한 동작을 함께 적는다.

# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
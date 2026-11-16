# IDEA-010 — SDKSignal

> **한 줄 설명:** 모바일 앱 개발팀이 사용하는 서드파티 SDK의 새 버전에서 개인정보 선언(privacy manifest)이 어떻게 달라졌는지 배포 전에 확인하도록, SDK 버전별 선언 이력을 수집하고 앱의 의존성 목록에 맞춰 변경 알림을 제공하는 개발자용 SaaS.

- **조사 기준일:** 2026-11-16
- **상태:** 초기 사업 가설 — 고객 인터뷰 및 유료 파일럿 전
- **제품명:** SDKSignal은 작업명이며 상표·도메인 사용 가능 여부는 확인하지 않았다.
- **초기 시장:** iOS 앱을 여러 개 운영하는 소규모 앱 스튜디오, 모바일 개발 에이전시, SDK를 정기적으로 업데이트하는 독립 개발팀
- **핵심 경계:** SDK의 공개 배포 패키지에 포함된 개인정보 선언 메타데이터의 버전 차이를 추적한다. 앱의 실제 네트워크 동작을 검사하거나 개인정보 법률 준수 여부를 인증하지 않는다.

## 1. 발견한 불편함

모바일 앱은 분석, 로그인, 결제, 광고, 충돌 보고, 푸시 알림 등 여러 기능을 서드파티 SDK에 의존한다. 앱 개발자가 직접 작성한 코드가 바뀌지 않아도 SDK를 업데이트하면 앱에 포함되는 개인정보 선언과 Required Reason API 관련 메타데이터가 달라질 수 있다.

Apple은 서드파티 SDK의 개인정보 처리 관행을 privacy manifest로 설명하도록 지원하며, Xcode는 앱에 포함된 SDK들의 manifest를 하나의 보고서로 모을 수 있다고 안내한다. Apple은 개발자가 자신이 포함한 서드파티 코드의 개인정보 처리 관행을 파악할 책임이 있고, App Store Connect에 제출하는 개인정보 응답도 실제 앱과 파트너의 처리 관행에 맞게 최신 상태로 유지해야 한다고 설명한다.

실무에서는 다음과 같은 질문이 생길 수 있다.

- 현재 사용하는 SDK의 최신 버전은 언제 개인정보 manifest를 바꿨는가?
- 새 버전에서 수집 데이터 유형, 수집 목적, 사용자 연결 여부 또는 추적 여부가 달라졌는가?
- 새 버전이 기존에 없던 Required Reason API 선언이나 추적 도메인을 추가했는가?
- 현재 앱의 잠금 파일(lockfile)에 기록된 SDK 버전과 새로 배포된 SDK 버전 사이에 어떤 차이가 있는가?
- 이 변경이 앱의 App Store 개인정보 응답을 다시 검토해야 할 이유가 되는가?
- 팀에서 누가 이 변경을 확인했고, 어떤 판단을 내렸는가?

SDK별 릴리스 노트, 저장소, 패키지 배포 페이지와 manifest 파일을 매번 직접 확인해야 한다면 여러 앱을 운영하는 소규모 팀에는 반복적인 조사 비용이 발생할 수 있다.

**핵심 문제 가설:** 앱 개발팀은 앱 자체의 개인정보 스캔뿐 아니라, 자신들이 의존하는 SDK의 새 버전에서 공개 개인정보 선언이 어떻게 바뀌었는지 사전에 추적하고, 그 변경을 앱별 검토 작업으로 연결할 가벼운 도구를 원할 수 있다.

이 문제가 얼마나 자주 발생하는지, 개발팀이 별도 구독료를 지불할지는 아직 검증되지 않았다.

## 2. 문제의 근본 원인

1. **SDK 버전별 선언 이력의 분산:** SDK의 manifest는 패키지나 저장소의 특정 릴리스에 포함될 수 있다. 현재 버전의 선언만 확인하면 이전 버전과의 차이를 놓칠 수 있다.
2. **배포 채널의 분산:** Swift Package Manager, CocoaPods, XCFramework, GitHub Releases 등 SDK별 배포 방식과 메타데이터 위치가 다르다.
3. **앱별 의존성 차이:** 같은 개발사가 여러 앱을 운영해도 각 앱의 SDK 구성과 버전이 다르다. 하나의 SDK 변경이 어떤 앱에 영향을 주는지 별도로 확인해야 한다.
4. **변경의 의미를 파악하기 어려움:** 단순 파일 해시 차이는 어떤 개인정보 선언이 바뀌었는지 알려주지 않는다. 구조화된 항목 단위 비교가 필요하다.
5. **검토 기록의 부재:** 변경 내용을 확인했더라도 어떤 앱에서 누가 검토했고, 어떤 이유로 업데이트 또는 보류했는지 GitHub 이슈·메신저·개인 메모에 흩어질 수 있다.
6. **선언과 실제 동작의 차이:** manifest는 SDK 제공자의 선언이다. 선언이 실제 런타임 동작을 완전하게 증명하지 않으므로 별도의 검증이 필요하다.

위 원인은 제품 설계를 위한 가설이다. 실제 개발팀의 업무 기록과 SDK 업데이트 절차를 인터뷰해 가장 비용이 큰 원인을 확인해야 한다.

## 3. 기존 해결책 및 경쟁 서비스 조사

**조사 기준일: 2026-11-16.** 공개 제품 설명, 공식 문서, 공개 저장소를 기준으로 조사했다. 실제 고객별 사용량, 가격, 탐지 정확도, 시장 점유율은 확인하지 못했다.

| 서비스·대안 | 공개 자료에서 확인한 기능 | SDKSignal과의 관계 |
|---|---|---|
| [Apple — Third-party SDK requirements](https://developer.apple.com/support/third-party-SDK-requirements/) | SDK privacy manifest와 서명 요구사항, Xcode의 통합 privacy report, 요구 대상 SDK 목록을 안내 | 플랫폼의 기준 자료이자 핵심 데이터 출처 |
| [Xcode privacy report](https://developer.apple.com/videos/play/wwdc2023/10060/) | 앱에 포함된 SDK의 privacy manifest를 모아 개발자가 개인정보 응답을 작성하도록 지원 | 기본 도구. 다만 팀이 SDK 공급자의 새 릴리스를 지속 추적하고 앱별 검토를 기록하는 별도 업무 흐름은 제품 기회 가설 |
| [sdk-privacy-scan](https://github.com/sunwoo05091/mobile-sdk-privacy-scan) | lockfile에서 의존성을 찾아 SDK manifest를 읽고, 개인정보 선언 파일을 생성하며 기준선 비교와 CI 실패 처리를 제공하는 공개 CLI | 매우 가까운 대안. 로컬 분석·선언 생성·CI 게이트 기능을 직접 경쟁 영역으로 간주 |
| [Respectlytics Dependency Privacy Scanner](https://respectlytics.com/tools/dependency-privacy-scanner/) | Podfile.lock, Package.resolved, Gradle, pubspec.lock 등을 브라우저에서 분석하고 공개 SDK 카탈로그를 기준으로 개인정보 관련 항목을 보여주는 도구 | lockfile 기반 스캔의 직접 대안 |
| [crasowas privacy manifest analyser](https://github.com/crasowas/app_store_required_privacy_manifest_analyser) | Xcode 프로젝트·패키지 의존성·소스 코드와 바이너리에서 API 사용 및 manifest 누락 가능성을 분석하는 스크립트 | 로컬 검사 및 누락 탐지 대안 |
| [Privado AI App Auditor](https://www.privado.ai/products/app-auditor) | iOS·Android 앱을 스캔하고 SDK, 데이터 흐름, 동의 동작을 분석하며 App Store 개인정보 보고서 생성을 지원 | 더 넓은 범위의 앱 개인정보 감사 제품 |
| [TruePrivacy Mobile App Privacy](https://trueprivacy.io/platform/mobile) | 정적·동적 앱 분석, SDK 탐지, 네트워크 트래픽 검사, 스토어 개인정보 라벨 검증, CI/CD 연동을 제공 | 기능 범위가 넓은 직접·인접 경쟁 제품 |
| [AppCensus](https://www.appcensus.io/getting-started) | 모바일 앱의 실제 데이터 전송과 권한을 분석하고 여러 앱 버전의 스캔 결과를 제공 | 실제 앱 동작 분석 중심의 대안 |

### 경쟁 조사 결론

개인정보 manifest 검사, lockfile 분석, CI 게이트, 앱 바이너리 스캔은 이미 제공되고 있다. 따라서 SDKSignal을 '최초의 모바일 개인정보 스캐너' 또는 '기존 도구가 전혀 없는 시장'으로 주장하지 않는다.

**차별화 가설은 앱을 매번 스캔하는 것이 아니라, SDK 공급자가 새 버전을 공개할 때마다 공개 배포물의 privacy manifest를 버전별로 수집·정규화하고, 그 SDK를 사용하는 앱에 변경 내용을 전달하는 upstream 변경 인텔리전스에 집중하는 것**이다.

예를 들어 앱 A가 SDK X 1.4.0을 사용 중이고 SDK X 1.5.0이 공개되면 다음을 제공한다.

- 1.4.0과 1.5.0의 manifest 구조화 비교
- 새로 추가·삭제·수정된 개인정보 선언 항목
- 원본 패키지 버전, 파일 해시, 확인 시각과 원본 출처
- 해당 SDK를 사용하는 앱 목록
- 담당자가 검토할 체크리스트와 승인 이력
- GitHub 이슈 또는 PR 코멘트로 전달할 변경 요약

현재 공개 조사에서 이 전체 흐름을 SDK 공급자 릴리스 감시와 앱별 의존성 연결을 중심으로 제공하는 독립 제품은 확인하지 못했다. 이는 부재의 증명이 아니므로 추가 경쟁사 조사와 고객 검증이 필요하다.

## 4. 제품 설명

SDKSignal은 앱의 소스 코드나 바이너리를 대신 검사하는 보안 스캐너가 아니라, **SDK 버전별 공개 개인정보 선언 변경을 추적하는 개발자용 데이터·알림 레이어**다.

### 핵심 데이터 모델

각 SDK 버전별로 다음 정보를 저장한다.

- 패키지 이름, 배포 채널, 버전, 릴리스 날짜
- 배포 원본 URL과 패키지 식별 정보
- privacy manifest 원본 파일의 SHA-256 해시
- 수집 데이터 유형과 목적
- 데이터가 사용자와 연결되는지 여부
- 추적 여부 및 추적 도메인
- Required Reason API 범주와 선언된 사유 코드
- manifest 파일의 존재 여부와 구조 검증 결과
- 서명 관련 메타데이터의 확인 가능 여부
- 수집 시각, 파서 버전, 데이터 신뢰도와 검토 상태

### 주요 기능

1. **SDK 카탈로그:** 공개 배포 채널에서 확인한 SDK와 버전별 개인정보 선언을 검색한다.
2. **릴리스 감시:** 새 SDK 버전이 게시되면 패키지의 공개 manifest를 확인한다.
3. **구조화 diff:** 단순 파일 비교가 아니라 데이터 유형, 목적, 추적 여부, 도메인, API 사유 코드별 차이를 보여준다.
4. **앱별 의존성 연결:** 사용자가 등록한 lockfile 또는 읽기 전용 GitHub 연동을 통해 어떤 앱이 해당 SDK를 사용하는지 연결한다.
5. **변경 알림:** 앱이 사용하는 SDK의 새 버전에서 개인정보 선언이 달라지면 담당자에게 알린다.
6. **검토 워크플로:** 변경 사항을 확인, 검토 필요, 보류, 앱 선언 업데이트 검토 완료 등으로 분류하고 담당자와 메모를 남긴다.
7. **릴리스 증빙:** SDK 버전, 원본 해시, 변경 요약, 검토자, 결정 시각을 묶은 Markdown·JSON 보고서를 만든다.
8. **개발 도구 연동:** GitHub PR 코멘트, 이슈, 이메일 또는 웹훅으로 변경을 전달한다.

### 첫 화면의 핵심 질문

사용자가 대시보드에 들어오면 다음 세 가지를 우선 확인할 수 있도록 한다.

- 현재 사용 중인 SDK 중 새 개인정보 선언이 공개된 항목은 무엇인가?
- 새 버전에서 개인정보 선언이 달라진 SDK는 무엇인가?
- 아직 사람이 검토하지 않은 변경은 어떤 앱에 연결되어 있는가?

## 5. 차별화 전략

| 구분 | 기존 대안의 중심 | SDKSignal의 차별화 가설 |
|---|---|---|
| 데이터 수집 | 현재 앱의 lockfile·바이너리·소스 검사 | SDK 공급자의 새 배포 버전과 공개 manifest를 지속 추적 |
| 분석 단위 | 현재 앱 또는 현재 의존성 목록 | SDK 공급자·패키지·버전별 선언 이력 |
| 알림 시점 | 앱 빌드·스캔·CI 실행 시점 | 앱이 새 SDK 버전을 채택하기 전에도 변경 사실을 알림 |
| 앱 연결 | 스캔 실행 시점의 프로젝트 | 여러 앱이 같은 SDK를 사용하는 관계를 지속 관리 |
| 결과 | 스캔 결과·manifest 생성·CI 게이트 | upstream 변경 피드와 앱별 검토 큐 |
| 데이터 보존 | 도구별 정책에 따름 | 원본 출처·파일 해시·수집 시각·검토 결정의 버전 이력 |

**중요한 한계:** 기존 로컬 도구와 CI 도구가 이미 충분한 기능을 제공한다면 SDKSignal의 별도 구독 가치는 낮을 수 있다. 특히 공개 SDK 카탈로그와 lockfile 분석만으로는 유료 제품이 되기 어렵다. 실제 차별화는 새 SDK 릴리스의 자동 수집 품질, 앱별 영향 연결, 변경 알림의 신뢰도와 팀 검토 이력에서 만들어져야 한다.

## 6. 타깃 고객

### 1차 고객

- iOS 앱을 2개 이상 운영하는 소규모 앱 스튜디오
- 여러 고객 앱을 관리하는 모바일 개발 에이전시
- 결제·분석·인증·광고 SDK를 자주 업데이트하는 앱 개발팀
- 자체 SDK를 배포하고 고객에게 개인정보 변경을 안내해야 하는 SDK 제작사

### 초기 고객에서 제외할 대상

- 앱이 하나뿐이고 SDK 업데이트가 드문 개인 개발자
- 이미 전담 개인정보 엔지니어와 자체 SDK 모니터링 시스템을 운영하는 대기업
- 법률 자문이나 규제 준수 인증을 원하는 고객

## 7. 사람들이 사용해야 할 이유

고객이 얻을 수 있는 가치는 다음과 같다.

1. SDK 업데이트 전에 개인정보 선언의 변경 여부를 확인할 수 있다.
2. 여러 앱이 동일 SDK를 사용해도 영향받는 앱을 따로 찾을 필요가 줄어든다.
3. SDK 버전별 원본 선언과 비교 결과를 한곳에서 확인할 수 있다.
4. 담당자가 바뀌어도 과거에 어떤 변경을 검토했는지 남는다.
5. 개발자는 모든 SDK 릴리스 페이지를 직접 확인하는 반복 업무를 줄일 수 있다.
6. 앱을 실제로 스캔하는 도구와 함께 사용해 공급자 선언 변경과 런타임 동작 검증을 구분할 수 있다.

이 가치는 아직 가설이다. 고객이 기존 GitHub Actions, 로컬 CLI, 수동 확인으로 충분하다고 판단하면 구매 이유가 약해진다.

## 8. 수익 모델 및 가격 가설

| 플랜 | 가격 가설 | 제공 범위 |
|---|---:|---|
| Free | 무료 | 공개 SDK 카탈로그 검색, 앱 1개, 수동 조회 |
| Solo | 월 $9 | 앱 3개, SDK 변경 이메일 알림, 기본 이력 |
| Studio | 월 $39 | 앱 10개, 팀 검토, GitHub 연동, 변경 보고서 |
| Agency | 월 $99 | 앱 40개, 고객별 워크스페이스, 역할·권한, 웹훅 |
| SDK Publisher | 별도 견적 | 공급사 자체 SDK의 공개 manifest 버전 이력과 소비자 알림 페이지 |

가격은 고객 인터뷰 전의 가설이며 시장에서 검증된 가격이 아니다.

**초기 과금 단위 가설:** 사용자 좌석보다 등록된 앱 수와 감시하는 SDK 수를 기준으로 과금한다. 실제 비용과 사용량을 측정한 뒤 조정한다.

## 9. MVP 범위

첫 4주 동안은 iOS 생태계만 대상으로 한다.

### 반드시 구현할 기능

1. Swift Package Manager의 Package.resolved와 CocoaPods의 Podfile.lock에서 SDK 이름·버전 추출
2. 공개 GitHub 릴리스 및 CocoaPods 배포 자료에서 SDK 새 버전 탐지
3. 공개 패키지 안의 PrivacyInfo.xcprivacy 파일 수집
4. manifest를 구조화된 JSON으로 정규화
5. SDK 버전 간 데이터 유형·목적·추적 여부·도메인·Required Reason API 차이 비교
6. 원본 URL·파일 해시·수집 시각을 결과에 포함
7. 사용자가 변경을 검토하고 승인·보류 상태를 기록
8. 이메일 또는 GitHub 이슈 알림
9. Markdown·JSON 변경 보고서 내보내기

### MVP에서 제외할 기능

- 앱 바이너리의 정적·동적 보안 분석
- 실제 네트워크 트래픽 캡처
- App Store Connect 개인정보 응답 자동 수정·게시
- 개인정보 법률 준수 여부 자동 판정
- Android 전체 SDK 카탈로그
- 모든 패키지 관리자와 모든 플랫폼 지원

### 권장 기술 구성

- Frontend: Next.js + TypeScript
- Backend: Cloudflare Workers
- Database: PostgreSQL 또는 Supabase
- Scheduler: Cloudflare Cron Triggers
- Object storage: Cloudflare R2
- SDK metadata collector: TypeScript 작업자
- Manifest parser: XML/Property List parser + JSON Schema validation
- GitHub integration: read-only GitHub App, PR comment·issue 생성 권한은 별도 동의
- Notifications: 이메일 + GitHub webhook

핵심 diff는 결정론적 규칙으로 수행한다. AI를 도입하더라도 변경 설명 초안에만 사용하고, 선언의 의미나 법률적 판단을 AI가 확정하지 않도록 한다.

## 10. 고객 확보 전략

1. **무료 공개 카탈로그:** 주요 SDK 30~50개의 버전별 manifest 이력을 공개해 검색 유입을 확보한다.
2. **오픈소스 CLI:** lockfile에서 SDK 목록을 추출하고 SDKSignal 공개 카탈로그와 비교하는 무료 CLI를 배포한다.
3. **GitHub Marketplace:** 개발자가 기존 CI에 한 단계로 연결할 수 있도록 GitHub Action을 제공한다.
4. **개발자 커뮤니티:** Swift, iOS, CocoaPods, Swift Package Manager 관련 커뮤니티에 실제 변경 사례와 비교 보고서를 공개한다.
5. **모바일 개발 에이전시:** 고객 앱 여러 개를 관리하는 소규모 에이전시를 대상으로 파일럿을 제안한다.
6. **SDK 제작사:** 자체 SDK의 manifest 변경을 소비자에게 알리는 공개 changelog 페이지를 제공하고 공급사 요금제로 확장한다.

광고보다 무료 CLI, GitHub Action, 공개 변경 사례를 통해 제품을 발견하도록 하는 전략을 우선한다.

## 11. 초기 개발·운영 비용 가설

| 항목 | 초기 비용 가설 |
|---|---:|
| 도메인 | 연 $10~20 |
| 초기 서버·DB·스토리지 | 월 $0~30 |
| 이메일 알림 | 무료 구간 또는 월 $0~10 |
| SDK 패키지 수집 작업 | 월 $0~20 추가 예상 |
| AI 설명 기능 | 초기 MVP에서는 비활성화하거나 사용량 제한 |
| 개발 인력 | 창업자 직접 개발, 인건비 별도 |

초기에는 월 $10~60 범위에서 실험할 수 있다는 가설이다. 실제 비용은 수집 주기, SDK 버전 수, 파일 보관량, GitHub API 사용량에 따라 달라진다. 이 수치는 공급사 견적이 아니며 무료 구간·요금 변경을 확인해야 한다.

## 12. 확장성

1. iOS의 CocoaPods·SwiftPM에서 시작해 XCFramework와 Carthage를 지원한다.
2. Flutter·React Native 프로젝트의 iOS 네이티브 의존성까지 연결한다.
3. Android SDK의 Data Safety 관련 공개 자료와 버전별 변경 추적으로 확장한다.
4. SDK 제공사가 자신의 선언 변경을 검토·게시하는 공급사 포털을 제공한다.
5. 앱별 SBOM 또는 dependency inventory와 연결해 보안·라이선스 변경 이벤트까지 통합할 가능성을 검토한다.

확장은 고객이 실제로 비용을 지불하는 핵심 기능이 확인된 뒤 진행한다.

## 13. 법적·운영상 리스크

| 리스크 | 대응 |
|---|---|
| SDK manifest가 실제 동작과 다를 수 있음 | 공급자 선언과 런타임 동작을 구분하고, 실제 동작을 검증했다고 표현하지 않는다. |
| manifest 변경이 법적 개인정보 응답 변경을 자동 의미하지 않음 | 변경 항목과 검토 질문만 제시하고 최종 판단은 앱 개발자에게 맡긴다. |
| 패키지 배포 자료 수집 제한 | 공개 API·정식 배포 경로를 우선 사용하고 각 패키지 저장소의 이용 조건을 확인한다. |
| SDK 버전과 manifest 파일 매칭 오류 | 원본 패키지 버전·파일 해시·출처·수집 시각을 보존하고 불확실한 결과를 별도 표시한다. |
| 카탈로그 데이터가 오래되거나 누락됨 | 마지막 확인 시각과 커버리지 상태를 노출하고 미확인 버전을 추측하지 않는다. |
| 경쟁사가 유사 기능을 빠르게 추가 | 앱 스캔이 아닌 공급사 릴리스 이력·앱별 영향 관계·검토 이력에 집중한다. |
| 비공개 저장소의 의존성 정보 노출 | 최소 권한, lockfile만 읽는 연동, 암호화, 데이터 삭제 기능을 제공한다. |
| 상표·저작권·패키지 배포 조건 | 원본 패키지를 재배포하지 않고 필요한 선언 메타데이터와 출처 링크만 저장한다. |

**제품의 한계:** SDKSignal은 Apple이나 Google의 공식 인증 도구가 아니다. 개인정보 법률 자문, App Store 심사 통과 보장, 실제 데이터 수집 행위의 완전한 탐지 또는 규제 준수 인증을 제공하지 않는다.

## 14. 30일 시장 검증 계획

### 1주 차 — 문제 검증
- iOS 앱 개발자와 소규모 모바일 에이전시 15명 인터뷰
- 최근 3개월 동안 SDK 업데이트 전후 개인정보 관련 검토를 한 적이 있는지 질문
- SDK 변경을 확인하는 현재 방법, 소요 시간, 누락 사례 수집
- 최근 SDK 릴리스 30개에서 manifest 변경 사례를 수동 조사

### 2주 차 — 데이터 검증
- 인기 SDK 30개와 각 3개 이상 버전의 manifest 수집 가능성 확인
- CocoaPods와 SwiftPM에서 버전 매칭 정확도 측정
- 동일 SDK 버전의 패키지 재배포·해시 차이와 manifest 차이를 구분
- 변경 결과를 개발자 5명에게 보여주고 오탐·누락 피드백 수집

### 3주 차 — MVP 파일럿
- CLI 또는 간단한 웹 화면 구현
- 앱별 lockfile 등록과 SDK 릴리스 변경 알림 제공
- GitHub 이슈 또는 PR 코멘트로 검토 요청 생성
- 3~5개 앱 프로젝트에 시험 적용

### 4주 차 — 지불 의사 검증
- 파일럿 팀에 Solo·Studio 가격 제시
- 월 구독 의향과 실제 결제 의사 구분
- 알림 열람률, 검토 완료율, 변경 확인에 걸린 시간 측정
- 3개 이상 유료 파일럿 의향이 없으면 가격·타깃·문제 가설 재검토

## 15. 검증 지표 및 중단 기준

### 핵심 지표

- 추적 대상 SDK의 새 버전 감지 성공률
- 패키지 버전과 manifest 원본의 정확한 매칭률
- 변경 항목 diff의 오탐률·누락률
- 새 변경 알림을 받은 뒤 검토를 완료한 비율
- 앱별 주간 활성 검토자 수
- GitHub 연동 후 2주차 재사용률
- 무료에서 유료로 전환한 팀 비율
- 앱당 월간 지원·수집 비용

### 초기 통과 가설

- 인터뷰 대상 15명 중 8명 이상이 최근 3개월 내 SDK 개인정보 선언 검토를 반복했다고 응답
- 파일럿 SDK 30개에서 새 버전과 manifest 매칭 성공률 90% 이상
- 파일럿 5개 앱 중 3개 이상이 2주 이상 알림을 실제 검토
- 최소 3개 팀이 월 $19 이상 유료 파일럿을 수락

이 기준은 목표치이며 시장에서 확인된 평균이 아니다.

### 중단 또는 방향 전환 기준

- 대부분의 고객이 Xcode 기본 보고서와 기존 무료 CLI로 충분하다고 답한다.
- SDK별 manifest 변경이 드물어 지속적인 알림 가치가 낮다.
- 공개 패키지에서 버전별 manifest를 안정적으로 수집할 수 없다.
- 고객이 앱별 영향 연결보다 실제 런타임 데이터 흐름 검사만을 원한다.
- 15개 인터뷰 후에도 월 $19 이상 지불 의향이 있는 팀을 3곳 이상 확보하지 못한다.

## 16. 참고 출처

1. [Apple — Third-party SDK requirements](https://developer.apple.com/support/third-party-SDK-requirements/)
2. [Apple — App privacy details on the App Store](https://developer.apple.com/app-store/app-privacy-details/)
3. [Apple — Get started with privacy manifests (WWDC23)](https://developer.apple.com/videos/play/wwdc2023/10060/)
4. [Apple — TN3184: Adding data collection details to your privacy manifest](https://developer.apple.com/documentation/technotes/tn3184-adding-data-collection-details-to-your-privacy-manifest)
5. [sdk-privacy-scan — 공개 lockfile 기반 개인정보 분석 CLI](https://github.com/sunwoo05091/mobile-sdk-privacy-scan)
6. [Respectlytics — Dependency Privacy Scanner](https://respectlytics.com/tools/dependency-privacy-scanner/)
7. [crasowas — App Store Required Privacy Manifest Analyser](https://github.com/crasowas/app_store_required_privacy_manifest_analyser)
8. [Privado AI — App Privacy Auditor](https://www.privado.ai/products/app-auditor)
9. [TruePrivacy — Mobile App Privacy](https://trueprivacy.io/platform/mobile)
10. [AppCensus — Getting Started](https://www.appcensus.io/getting-started)
11. [OWASP — Software Composition Analysis of iOS Dependencies](https://mas.owasp.org/MASTG/techniques/ios/MASTG-TECH-0133/)

## 17. 최종 판단

**현재 상태: 초기 사업 가설 — 고객 인터뷰 및 유료 파일럿 전.**

모바일 앱 개인정보 검사는 이미 경쟁이 존재하며 무료 CLI도 확인했다. SDKSignal이 성립하려면 단순 manifest 검사나 lockfile 분석을 반복하는 제품이 아니라, SDK 공급자 측의 새 릴리스와 개인정보 선언 변화를 지속 추적하고, 여러 앱에 대한 영향을 연결하는 데이터 서비스가 되어야 한다.

가장 먼저 검증할 것은 SDK 개발팀이 실제로 이 변경을 추적하는 데 시간을 쓰고 있는지, 기존 도구만으로 해결되지 않는 문제가 있는지, 그리고 그 문제에 구독료를 지불할 의사가 있는지다.

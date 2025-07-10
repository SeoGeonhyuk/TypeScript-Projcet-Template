# Extended Copy - 프로젝트 계획서

## 📋 프로젝트 개요

### 목표
맥 환경에서 Command + C의 기본 동작은 유지하면서, Command + Option + C를 통해 누적 복사 기능을 제공하는 네이티브 macOS 애플리케이션 개발

### 핵심 가치
- **기존 워크플로우 보존**: Command + C는 기존 동작 100% 유지
- **직관적인 확장**: Option 키로 명확한 기능 구분
- **심플한 초기화**: Command + C 실행 시 누적 모드 자동 종료

---

## ⌨️ 기능 명세

### 키보드 동작
```
Command + C           → 일반 복사 (기존 동작) + 누적 모드 초기화
Command + Option + C  → 누적 복사 (새 내용을 기존에 추가)
Command + V           → 붙여넣기 (누적된 전체 내용)
```

### 초기화 조건
1. **수동 초기화**: `Command + C` 실행 시 즉시 초기화
2. **앱 전환 초기화**: 다른 앱으로 포커스 이동 시 자동 초기화
3. **시간 기반 초기화**: 5분 이상 비활성 시 자동 초기화

### 시각적 피드백
- **상태바 아이콘**: 📋 (일반) → 📋+ (누적 시작) → 📋3 (누적 개수)
- **토스트 알림**: "3개 항목 누적됨" 형태의 간단한 피드백

---

## 🏗 시스템 아키텍처

### 전체 구조
```
Extended Copy
├── App Layer (UI/UX)
│   ├── StatusBarController    // 상태바 인터페이스
│   ├── NotificationManager    // 토스트 알림
│   └── SettingsView          // 환경설정 (구분자 등)
│
├── Core Layer (비즈니스 로직)
│   ├── AccumulativeClipboard  // 누적 복사 핵심 로직
│   ├── ClipboardState        // 상태 관리 (일반/누적 모드)
│   └── TextProcessor         // 텍스트 조합 및 처리
│
├── System Layer (시스템 연동)
│   ├── KeyboardEventMonitor  // 전역 키보드 후킹
│   ├── ClipboardManager      // NSPasteboard 래핑
│   └── PermissionManager     // Accessibility 권한 관리
│
└── Foundation Layer (기반)
    ├── AppDelegate           // 앱 생명주기
    ├── Configuration         // 설정 관리
    └── Logger               // 로깅 시스템
```

### 핵심 클래스 설계

#### 1. AccumulativeClipboard
```swift
class AccumulativeClipboard {
    private var items: [ClipboardItem] = []
    private var separator: String = "\n"
    private var isActive: Bool = false
    
    func addItem(_ text: String)
    func getAllContent() -> String
    func reset()
    func setActive(_ active: Bool)
}
```

#### 2. KeyboardEventMonitor
```swift
class KeyboardEventMonitor {
    private var eventTap: CFMachPort?
    
    func startMonitoring()
    func stopMonitoring()
    private func handleKeyEvent(_ event: CGEvent) -> Bool
}
```

#### 3. StatusBarController
```swift
class StatusBarController {
    private var statusItem: NSStatusItem
    private var menu: NSMenu
    
    func updateIcon(_ state: ClipboardState)
    func showNotification(_ message: String)
}
```

---

## 🛠 기술 스택

### 개발 환경
- **언어**: Swift 5.8+
- **최소 지원 버전**: macOS 12.0 (Monterey)
- **개발 도구**: Xcode 15+
- **패키지 매니저**: Swift Package Manager

### 핵심 프레임워크
- **AppKit**: 네이티브 macOS UI
- **Carbon**: 전역 키보드 이벤트 감지
- **CoreGraphics**: Quartz Event Services (CGEventTap)
- **Foundation**: 기본 데이터 처리

### 서드파티 라이브러리 (최소화)
- 가능한 한 시스템 프레임워크만 사용
- 필요 시 SwiftUI 조합 (설정 화면)

---

## 📁 프로젝트 구조

```
ExtendedCopy/
├── ExtendedCopy.xcodeproj
├── ExtendedCopy/
│   ├── App/
│   │   ├── AppDelegate.swift
│   │   ├── main.swift
│   │   └── Info.plist
│   │
│   ├── Core/
│   │   ├── AccumulativeClipboard.swift
│   │   ├── ClipboardState.swift
│   │   ├── TextProcessor.swift
│   │   └── Configuration.swift
│   │
│   ├── System/
│   │   ├── KeyboardEventMonitor.swift
│   │   ├── ClipboardManager.swift
│   │   └── PermissionManager.swift
│   │
│   ├── UI/
│   │   ├── StatusBarController.swift
│   │   ├── NotificationManager.swift
│   │   └── SettingsView.swift
│   │
│   └── Resources/
│       ├── Assets.xcassets
│       ├── StatusBarIcon.png
│       └── Localizable.strings
│
├── ExtendedCopyTests/
│   ├── Core/
│   │   ├── AccumulativeClipboardTests.swift
│   │   └── TextProcessorTests.swift
│   │
│   ├── System/
│   │   └── ClipboardManagerTests.swift
│   │
│   └── TestHelpers/
│       └── MockClipboard.swift
│
├── ExtendedCopyUITests/
│   ├── StatusBarUITests.swift
│   └── KeyboardInteractionTests.swift
│
└── Documentation/
    ├── TEST_CASES.md
    ├── SENIOR_DEV_BEST_PRACTICES.md
    ├── DEVELOPMENT_GUIDELINES.md
    └── API_REFERENCE.md
```

---

## 🔧 핵심 구현 방법

### 1. 전역 키보드 이벤트 감지
```swift
// CGEventTap을 사용한 키보드 후킹
let eventMask = (1 << CGEventType.keyDown.rawValue)
let eventTap = CGEvent.tapCreate(
    tap: .cgSessionEventTap,
    place: .headInsertEventTap,
    options: .defaultTap,
    eventsOfInterest: CGEventMask(eventMask),
    callback: { (proxy, type, event, refcon) in
        let keyCode = event.getIntegerValueField(.keyboardEventKeycode)
        let flags = event.flags
        
        // Command+Option+C 감지 (C = keyCode 8)
        if keyCode == 8 && flags.contains([.maskCommand, .maskAlternate]) {
            handleAccumulativeCopy()
            return nil // 이벤트 소비
        }
        
        // Command+C 감지 (누적 모드 초기화)
        if keyCode == 8 && flags.contains(.maskCommand) && !flags.contains(.maskAlternate) {
            handleNormalCopy()
        }
        
        return Unmanaged.passUnretained(event)
    },
    userInfo: nil
)
```

### 2. 클립보드 관리
```swift
class ClipboardManager {
    private let pasteboard = NSPasteboard.general
    
    func getCurrentText() -> String? {
        return pasteboard.string(forType: .string)
    }
    
    func setText(_ text: String) {
        pasteboard.clearContents()
        pasteboard.setString(text, forType: .string)
    }
    
    func addToAccumulated(_ newText: String, separator: String = "\n") {
        let current = getCurrentText() ?? ""
        let combined = current.isEmpty ? newText : "\(current)\(separator)\(newText)"
        setText(combined)
    }
}
```

### 3. 상태 관리
```swift
enum ClipboardState {
    case normal
    case accumulating(count: Int)
    
    var statusBarIcon: String {
        switch self {
        case .normal:
            return "📋"
        case .accumulating(let count):
            return count == 1 ? "📋+" : "📋\(count)"
        }
    }
}
```

---

## 🚦 개발 단계

### Phase 1: 기반 구조 (주 1) ✅
- [x] 프로젝트 문서 작성
- [x] Xcode 프로젝트 생성 및 기본 구조 설정
- [x] AppDelegate 및 기본 앱 생명주기 구현
- [x] 로깅 시스템 구축

### Phase 2: 코어 기능 (주 2-3) 🔄
- [x] ClipboardManager 구현 (NSPasteboard) - 테스트 완료
- [x] AccumulativeClipboard 클래스 구현 - 테스트 완료
- [x] PermissionManager 구현 (Accessibility 권한) - 테스트 완료
- [x] KeyboardEventMonitor 구현 (CGEventTap) - 테스트 완료
- [ ] ClipboardCoordinator 구현 (전체 워크플로우 조정) - 진행 중
- [ ] 기본 누적 로직 및 초기화 메커니즘 통합 테스트

### Phase 3: UI 구현 (주 4) ✅
- [x] StatusBarController 구현
- [x] 상태바 아이콘 및 메뉴 시스템
- [x] NotificationManager (토스트 알림)
- [ ] 기본 환경설정 인터페이스

### Phase 4: 시스템 통합 (주 5) ✅
- [x] PermissionManager (Accessibility 권한) - Phase 2에서 완료
- [x] 앱 전환 감지 및 자동 초기화 - SystemEventManager로 구현
- [x] 시간 기반 초기화 구현 - 타임아웃 타이머 구현
- [x] 에러 처리 및 복구 메커니즘 - 전체 컴포넌트에 구현

### Phase 5: 테스트 및 최적화 (주 6-7) 🔄
- [x] 단위 테스트 작성 (ClipboardManager, AccumulativeClipboard, PermissionManager, KeyboardEventMonitor)
- [ ] 통합 테스트 구현 - 진행 예정
- [ ] 성능 최적화 (메모리, 응답성)
- [ ] 다양한 앱에서의 호환성 테스트

### Phase 6: 배포 준비 (주 8)
- [ ] 코드 리뷰 및 리팩토링
- [ ] 문서 완성 (사용자 가이드, API 문서)
- [ ] 앱 아이콘 및 에셋 최종화
- [ ] 배포 패키지 준비

---

## 🔒 보안 및 권한

### 필요 권한
1. **Accessibility API**: 전역 키보드 이벤트 감지
2. **Background Processing**: 백그라운드에서 키보드 모니터링

### 권한 요청 프로세스
```swift
class PermissionManager {
    func checkAccessibilityPermission() -> Bool {
        return AXIsProcessTrusted()
    }
    
    func requestAccessibilityPermission() {
        let options = [kAXTrustedCheckOptionPrompt.takeUnretainedValue(): true]
        AXIsProcessTrustedWithOptions(options as CFDictionary)
    }
}
```

### 개인정보 보호
- 클립보드 내용은 메모리에만 저장 (디스크 저장 없음)
- 앱 종료 시 모든 누적 내용 자동 삭제
- 사용자 활동 로깅 최소화

---

## ⚡ 성능 최적화

### 메모리 관리
- 누적 텍스트 크기 제한 (기본 10MB)
- 자동 가비지 컬렉션 활용
- 대용량 텍스트에 대한 경고 시스템

### 응답성 최적화
- 키보드 이벤트 처리: 비동기 큐 사용
- UI 업데이트: 메인 스레드에서 즉시 처리
- 클립보드 접근: 백그라운드 스레드에서 처리

### 시스템 리소스
- CPU 사용률 모니터링
- 배터리 영향 최소화
- 시스템 이벤트 필터링 최적화

---

## 🧪 테스트 전략

### 단위 테스트 (Unit Tests)
- AccumulativeClipboard 로직 테스트
- TextProcessor 기능 테스트
- Configuration 관리 테스트
- 목표 커버리지: 80% 이상

### 통합 테스트 (Integration Tests)
- 키보드 이벤트 → 클립보드 변경 플로우
- 권한 관리 시스템
- 상태바 UI 업데이트

### UI 테스트 (UI Tests)
- 상태바 아이콘 변경
- 알림 표시 정확성
- 메뉴 시스템 동작

### 성능 테스트 (Performance Tests)
- 대용량 텍스트 처리
- 장시간 연속 사용
- 메모리 누수 검사

---

## 📊 성공 지표

### 기능적 지표
- [x] 핵심 컴포넌트 단위 테스트 통과 (93개+)
- [ ] 통합 테스트 케이스 통과 (25개 목표)
- [ ] 주요 앱 10개에서 정상 동작 검증
- [ ] 24시간 연속 사용 시 크래시 없음

### 성능 지표
- [ ] 키보드 이벤트 응답 시간 < 100ms
- [ ] 메모리 사용량 < 50MB (일반 사용)
- [ ] CPU 사용률 < 1% (비활성 시)

### 사용성 지표
- [ ] 권한 설정 완료 시간 < 2분
- [ ] 사용자 학습 시간 < 5분
- [ ] 에러 상황에서 복구 가능

---

## 🚀 향후 확장 계획

### v1.1 - 고급 기능
- [ ] 사용자 정의 구분자
- [ ] 최대 누적 개수 설정
- [ ] 히스토리 보기 기능

### v1.2 - 생산성 향상
- [ ] 템플릿 기능 (자주 사용하는 텍스트 조합)
- [ ] 텍스트 포맷팅 옵션
- [ ] 다국어 지원

### v1.3 - 팀 기능
- [ ] 설정 동기화 (iCloud)
- [ ] 사용 통계
- [ ] 키보드 단축키 커스터마이징

---

## 📞 지원 및 피드백

### 이슈 트래킹
- GitHub Issues를 통한 버그 리포트
- 기능 요청 및 개선 제안
- 사용자 피드백 수집

### 업데이트 정책
- 보안 패치: 즉시 배포
- 버그 수정: 2주 주기
- 새 기능: 분기별 업데이트

---

이 계획서는 Extended Copy 프로젝트의 전체적인 방향성과 구체적인 구현 방법을 제시합니다. TDD 방식으로 개발하여 안정적이고 고품질의 macOS 애플리케이션을 완성할 예정입니다.
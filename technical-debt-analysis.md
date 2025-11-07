# Technical Debt Analysis

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Outdated Frameworks & Dependencies](#outdated-frameworks--dependencies)
3. [Code Duplication Analysis](#code-duplication-analysis)
4. [Coupling Issues](#coupling-issues)
5. [Architecture Concerns](#architecture-concerns)
6. [Refactoring Priorities](#refactoring-priorities)
7. [PWA Migration Considerations](#pwa-migration-considerations)
8. [Recommended Actions](#recommended-actions)

## Executive Summary

Both LIFTUPP iPad apps (OSCE and Develop) are mature production applications with **moderate technical debt**. While the core architecture is sound, several areas require attention for long-term maintainability and potential web migration.

### Debt Score by Category

| Category | Severity | Impact | Effort to Fix |
|----------|----------|--------|---------------|
| **Outdated Frameworks** | 🟡 Medium | High | Medium |
| **Code Duplication** | 🟡 Medium | Medium | Low-Medium |
| **Tight Coupling** | 🟢 Low | Medium | Medium |
| **Architecture** | 🟢 Low | Low | N/A |
| **Testing Coverage** | 🔴 High | High | High |
| **Documentation** | 🟢 Low | Low | Complete (this repo) |

**Overall Assessment**: ⚠️ **Medium Priority** - Address before major feature work or migration

## Outdated Frameworks & Dependencies

### 1. iOS Deployment Target (iOS 12.0)

**Current**: iOS 12.0 (Released September 2018)
**Latest iOS**: 17.x (as of 2024)
**Support Status**: iOS 12 no longer receives security updates

**Impact**:
- Cannot use modern Swift/iOS features (SwiftUI, Combine, async/await)
- Missing performance optimizations from newer iOS versions
- Potential security vulnerabilities in unsupported OS

**Recommendation**:
```diff
- Deployment Target: iOS 12.0
+ Deployment Target: iOS 15.0 (minimum supported by Apple)
```

**Breaking Changes**: Minimal - iOS 12-14 users would need OS update

---

### 2. Objective-C vs. Swift

**Current**: ~95% Objective-C, ~5% Swift (KeychainStore, StringVault only)
**Industry Trend**: Swift is the preferred language for iOS development

**Impact**:
- Harder to find Objective-C developers
- Missing modern language features (optionals, value types, protocol extensions)
- Verbose syntax compared to Swift
- No web compatibility (PWA migration requires JavaScript/TypeScript)

**Technical Debt Indicators**:
```objc
// Objective-C: Verbose, error-prone
NSArray *year3Students = [DSStudent fetchAllWithPredicate:
    [NSPredicate predicateWithFormat:@"year == %@", @"Year 3"]];

for (DSStudent *student in year3Students) {
    if (student.forename != nil && student.surname != nil) {
        NSString *fullName = [NSString stringWithFormat:@"%@ %@",
                             student.forename, student.surname];
        NSLog(@"%@", fullName);
    }
}
```

```swift
// Swift equivalent: Concise, type-safe
let year3Students = Student.fetch(where: \.year == "Year 3")

for student in year3Students {
    let fullName = "\(student.forename ?? "") \(student.surname ?? "")"
    print(fullName)
}
```

**Recommendation**:
- **Short-term**: Keep Objective-C for stability
- **Long-term**: Gradual Swift migration for new features
- **PWA Migration**: Complete rewrite in TypeScript/React

---

### 3. AFNetworking 4.0

**Current**: AFNetworking 4.0 (last major update: 2020)
**Alternative**: URLSession (built-in, modern, actively maintained)
**Status**: AFNetworking is in maintenance mode, not actively developed

**Impact**:
- Dependency on third-party library for basic networking
- Missing modern async/await networking patterns
- URLSession has feature parity and better iOS integration

**Current Usage**:
```objc
// AFNetworking wrapper
@interface DSHTTPSessionManager : AFHTTPSessionManager
+ (void)syncClient:(NSDictionary *)parameters
           success:(void (^)(NSData *data))success
           failure:(void (^)(NSError *error))failure;
@end
```

**Modern Alternative**:
```swift
// Native URLSession with async/await
func syncClient(parameters: [String: Any]) async throws -> Data {
    let url = URL(string: "https://api.liftupp.com/syncClient")!
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.httpBody = try JSONSerialization.data(withJSONObject: parameters)

    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw NetworkError.invalidResponse
    }
    return data
}
```

**Recommendation**:
- **Short-term**: Keep AFNetworking (working, stable)
- **Medium-term**: Migrate to URLSession
- **Effort**: Low (DSHTTPSessionManager already abstracts AFNetworking)

---

### 4. CocoaLumberjack (OSCE only)

**Current**: CocoaLumberjack 3.8 for logging (OSCE app)
**Alternative**: OSLog / os_log (native, better Xcode integration)
**Status**: CocoaLumberjack actively maintained but native logging is simpler

**Impact**:
- Additional dependency for logging
- OSLog provides better performance and privacy
- Inconsistency: Develop app uses native logging, OSCE uses CocoaLumberjack

**Current Usage (OSCE)**:
```objc
#import <CocoaLumberjack/CocoaLumberjack.h>

DDLogDebug(@"Starting sync");
DDLogInfo(@"Synced %lu objects", count);
DDLogError(@"Sync failed: %@", error);
```

**Native Alternative**:
```objc
#import <os/log.h>

os_log_debug(OS_LOG_DEFAULT, "Starting sync");
os_log_info(OS_LOG_DEFAULT, "Synced %lu objects", count);
os_log_error(OS_LOG_DEFAULT, "Sync failed: %@", error.localizedDescription);
```

**Recommendation**:
- **Short-term**: Migrate OSCE to native logging (consistency with Develop)
- **Benefit**: Remove dependency, better Xcode Console integration
- **Effort**: Very Low (search & replace)

---

### 5. Firebase Crashlytics

**Current**: Firebase Crashlytics for crash reporting
**Status**: Actively maintained by Google
**Alternative**: None superior for iOS crash reporting

**Impact**: ✅ No technical debt - modern, well-maintained

**Recommendation**: Keep Firebase Crashlytics

---

### 6. CocoaPods vs. Swift Package Manager

**Current**: CocoaPods for dependency management
**Alternative**: Swift Package Manager (SPM) - Apple's official solution
**Trend**: Industry moving to SPM

**Impact**:
- CocoaPods requires `pod install` and `.xcworkspace`
- SPM integrates directly into Xcode
- Some libraries now SPM-only

**Recommendation**:
- **Short-term**: Keep CocoaPods (stable, working)
- **Long-term**: Migrate to SPM when updating dependencies
- **Effort**: Low-Medium (change Podfile to Package.swift)

## Code Duplication Analysis

### 1. View Controllers (Potential Duplication)

**Finding**: OSCE and Develop have **19 and 20 unique view controllers** respectively, but may share similar patterns.

**Analysis**:
```
Common Patterns:
- Form rendering (both apps)
- Student selection (both apps)
- Attendance marking (both apps)
- Settings/sync UI (both apps)

OSCE-Specific:
- Timer management (DSTimerViewController, DSCircuitTimerViewController)
- Comment hierarchy (Circuit/Station/Candidate)
- Assistance requests (DSAssistanceViewController)

Develop-Specific:
- Patient management (DSPatientViewController, DSPatientLogViewController)
- Diagnosis selection (DSDiagnosisViewController)
- Procedure recording (DSProcedureViewController)
- Threshold evaluation (DSThresholdViewController)
```

**Technical Debt**: 🟢 **Low** - Duplication is minimal, most unique VCs serve different purposes

**Recommendation**: No action needed - separation is appropriate

---

### 2. Entity Class Implementations

**Finding**: Both apps have similar entity base class patterns but different entity counts (37 vs 63).

**Shared Entities (27)**: No duplication - same code used from `liftupp-app-library`
**Unique Entities**: Each app's entities serve distinct purposes

**Technical Debt**: 🟢 **Low** - Appropriate separation

---

### 3. Sync Logic

**Finding**: Both apps use **identical sync architecture** from shared library.

**Shared Code**:
- `DSModel.m` (1,018 lines) - shared
- `DSSyncManager.m` - shared
- `DSHTTPSessionManager.m` - shared

**App-Specific**:
- Entity sync metadata (userInfo in Core Data model)
- Upload/download entity lists

**Technical Debt**: 🟢 **Low** - Excellent code reuse (estimated ~65%)

**Recommendation**: No action needed - this is a strength, not debt

---

### 4. UI Components

**Finding**: Both apps use same UI component library (`liftupp-app-library/DS Views`).

**Shared Components**:
- DSButton, DSRoundedButton
- DSLabel, DSImageView
- DSTableView, DSTableViewCell
- DSNavigationController
- 60+ category extensions

**Technical Debt**: 🟢 **Low** - Excellent reuse

---

### 5. Settings & Configuration

**Finding**: DSSettingsManager (1,023 lines) handles both apps' configuration.

**Potential Improvement**:
```objc
// Current: Institution checks in monolithic file
+ (BOOL)isLiverpool {
    return [[[self sharedSettingsManager] institutionCode]
            caseInsensitiveCompare:@"liverpool"] == NSOrderedSame;
}

+ (BOOL)isManchester { ... }
+ (BOOL)isQMUL { ... }
// ... 13+ similar methods
```

**Refactoring Suggestion**:
```swift
// Better: Enum-based institutions
enum Institution: String {
    case liverpool = "liverpool"
    case manchester = "manchester"
    case qmul = "qmul"
    // ...

    var name: String {
        switch self {
        case .liverpool: return "University of Liverpool"
        case .manchester: return "University of Manchester"
        // ...
        }
    }
}

// Usage
let institution = Institution(rawValue: code)
```

**Technical Debt**: 🟡 **Medium** - Methods are repetitive

**Recommendation**: Refactor to enum-based approach

**Effort**: Low (1-2 hours)

## Coupling Issues

### 1. Singleton Pattern Overuse

**Finding**: 4 singletons in the system

**Singletons**:
1. `DSModel.sharedModel`
2. `DSSettingsManager.sharedSettingsManager`
3. `DSSyncManager.manager`
4. `DSHTTPSessionManager.defaultSessionManager`

**Impact**:
- Hard to test (global state)
- Can't easily mock for unit tests
- Implicit dependencies (view controllers assume singletons exist)

**Example Coupling**:
```objc
@implementation DSStudentViewController

- (void)saveStudent {
    // Tightly coupled to DSModel singleton
    [[DSModel sharedModel].managedObjectContext save:nil];
}

@end
```

**Better Approach (Dependency Injection)**:
```swift
class StudentViewController: UIViewController {
    let dataManager: DataManaging  // Protocol, not concrete class

    init(dataManager: DataManaging) {
        self.dataManager = dataManager
        super.init(nibName: nil, bundle: nil)
    }

    func saveStudent() {
        dataManager.save()  // Mockable for testing
    }
}
```

**Technical Debt**: 🟡 **Medium** - Impacts testability

**Recommendation**:
- **Short-term**: Keep singletons (refactoring is high effort)
- **Long-term**: Introduce dependency injection for new code
- **PWA Migration**: Use proper DI from the start

---

### 2. Core Data Context Access

**Finding**: View controllers directly access `managedObjectContext` property.

**Current Pattern**:
```objc
@implementation DSFormViewController

- (void)saveForm {
    NSManagedObjectContext *context = [DSModel sharedModel].managedObjectContext;
    [context save:nil];  // Direct access, tight coupling
}

@end
```

**Impact**:
- View controllers depend on Core Data implementation
- Hard to swap persistence layer
- Can't easily test without Core Data stack

**Better Approach**:
```swift
protocol DataRepository {
    func save() throws
    func fetch<T>(_ type: T.Type, predicate: NSPredicate?) -> [T]
}

class FormViewController: UIViewController {
    let repository: DataRepository

    func saveForm() {
        try? repository.save()  // No Core Data knowledge
    }
}
```

**Technical Debt**: 🟡 **Medium** - Common in Core Data apps

**Recommendation**:
- **Short-term**: Accept coupling (standard for Core Data apps)
- **PWA Migration**: Use abstracted data layer from start

---

### 3. Network Manager Coupling

**Finding**: Sync code directly uses DSHTTPSessionManager.

**Current**:
```objc
[DSHTTPSessionManager syncClient:data success:^(NSData *response) {
    // Process response
} failure:^(NSError *error) {
    // Handle error
}];
```

**Impact**:
- Can't easily test sync logic without network
- Hard to mock responses

**Technical Debt**: 🟢 **Low** - DSHTTPSessionManager already abstracts AFNetworking

**Recommendation**: Current abstraction is sufficient

## Architecture Concerns

### 1. MVC Architecture (Not MVVM)

**Current**: Strict MVC pattern
**Alternative**: MVVM (Model-View-ViewModel) for better testability

**Impact**:
- View controllers contain business logic
- Hard to unit test view controllers
- Fat view controllers (some >500 lines)

**Example**:
```objc
@implementation DSStudentOverviewViewController  // 650 lines

- (void)viewDidLoad {
    [super viewDidLoad];
    [self fetchStudentData];         // Data fetching in VC
    [self calculateStatistics];      // Business logic in VC
    [self updateUI];                 // View logic in VC
}

- (void)fetchStudentData {
    // Core Data queries in view controller
    NSPredicate *pred = [NSPredicate predicateWithFormat:@"..."];
    self.students = [DSStudent fetchAllWithPredicate:pred];
}

- (void)calculateStatistics {
    // Business logic in view controller
    self.averageScore = // ... calculation ...
}

@end
```

**MVVM Alternative**:
```swift
// ViewModel: Testable business logic
class StudentOverviewViewModel {
    func fetchStudents() -> [Student] { ... }
    func calculateStatistics() -> Statistics { ... }
}

// View Controller: Just binds data to UI
class StudentOverviewViewController: UIViewController {
    let viewModel: StudentOverviewViewModel

    override func viewDidLoad() {
        super.viewDidLoad()
        let students = viewModel.fetchStudents()
        let stats = viewModel.calculateStatistics()
        updateUI(with: students, stats: stats)
    }
}
```

**Technical Debt**: 🟡 **Medium** - Common iOS pattern, but testability suffers

**Recommendation**:
- **Short-term**: Keep MVC (major refactoring required for MVVM)
- **Long-term**: Consider MVVM for new features
- **PWA Migration**: Use React/Vue component architecture (better than MVC)

---

### 2. Storyboards vs. Programmatic UI

**Current**: Mix of Storyboards and programmatic UI
**Trend**: Industry moving away from Storyboards to programmatic or SwiftUI

**Storyboards Used**:
- `MainStoryBoard.storyboard` - Main interface
- `Launch_Screen.storyboard` - Launch screen
- 7 XIB files for custom cells/views

**Impact**:
- Storyboards hard to review in version control (XML)
- Merge conflicts difficult to resolve
- Can't easily reuse components

**Technical Debt**: 🟢 **Low** - Standard for mature iOS apps

**Recommendation**:
- **Short-term**: Keep existing storyboards
- **New features**: Consider programmatic UI
- **PWA Migration**: HTML/CSS components (simpler than storyboards)

---

### 3. No Modular Architecture

**Current**: Monolithic app structure
**Alternative**: Modular architecture with Swift Packages/Frameworks

**Impact**:
- Long build times
- Hard to isolate features
- Can't easily extract reusable modules

**Recommendation**:
- **Short-term**: Not worth refactoring (stable codebase)
- **PWA Migration**: Start modular from day 1

## Refactoring Priorities

### Priority 1: Critical (Do Before New Features)

#### 1.1 Update iOS Deployment Target
- **From**: iOS 12.0
- **To**: iOS 15.0
- **Effort**: 2 hours
- **Benefit**: Security updates, modern APIs
- **Risk**: Very Low (iOS 15+ has 95% market share)

#### 1.2 Migrate OSCE to Native Logging
- **From**: CocoaLumberjack
- **To**: os_log (OSLog)
- **Effort**: 4 hours
- **Benefit**: Consistency with Develop app, remove dependency
- **Risk**: None

#### 1.3 Add Unit Test Coverage
- **Current**: 0% (no tests in repository)
- **Target**: 30% (core business logic)
- **Effort**: High (40+ hours)
- **Benefit**: Catch regressions, enable refactoring

---

### Priority 2: Important (Next Quarter)

#### 2.1 Refactor DSSettingsManager Institution Checks
- **Current**: 13+ repetitive methods
- **Target**: Enum-based approach
- **Effort**: 2 hours
- **Benefit**: Code clarity, type safety

#### 2.2 Abstract Core Data Access
- **Current**: Direct context access
- **Target**: Repository pattern
- **Effort**: 20 hours
- **Benefit**: Better testability, looser coupling

#### 2.3 Migrate to URLSession
- **Current**: AFNetworking 4.0
- **Target**: Native URLSession
- **Effort**: 8 hours
- **Benefit**: Remove dependency, modern async patterns

---

### Priority 3: Nice to Have (Long-term)

#### 3.1 Gradual Swift Migration
- **Current**: 95% Objective-C
- **Target**: 50% Swift (new features in Swift)
- **Effort**: Ongoing (6-12 months)
- **Benefit**: Modern language features, easier hiring

#### 3.2 MVVM for New Features
- **Current**: MVC only
- **Target**: MVVM for complex view controllers
- **Effort**: Ongoing
- **Benefit**: Better testability

#### 3.3 Migrate to Swift Package Manager
- **Current**: CocoaPods
- **Target**: SPM
- **Effort**: 4 hours
- **Benefit**: Native Xcode integration

## PWA Migration Considerations

If migrating to Progressive Web App (PWA), consider:

### 1. Technology Stack Recommendation

**Frontend**:
- **Framework**: React or Vue.js
- **State Management**: Redux or Vuex
- **UI Library**: Material-UI or Tailwind CSS
- **Offline**: Service Workers
- **Data**: IndexedDB (local storage) + REST API

**Backend** (unchanged):
- Keep existing PHP web portal
- Same sync API endpoints
- Add WebSocket for real-time features (assistance requests)

---

### 2. Architecture Mapping

| iOS Component | PWA Equivalent |
|--------------|----------------|
| **UIViewController** | React Component / Vue Component |
| **Core Data** | IndexedDB + REST API |
| **DSModel** | Data Service Layer (TypeScript classes) |
| **DSSyncManager** | Sync Service (background sync API) |
| **DSSettingsManager** | LocalStorage + Config Service |
| **AFNetworking** | Fetch API / Axios |
| **CocoaLumberjack** | Console.log / Sentry |
| **Firebase Crashlytics** | Sentry / LogRocket |
| **Storyboards** | JSX / HTML Templates |

---

### 3. Feature Parity Challenges

**Easy to Migrate**:
- ✅ Form-based assessments (HTML forms)
- ✅ Student lists (React tables)
- ✅ Attendance tracking (checkboxes)
- ✅ Data sync (Fetch API + Service Workers)
- ✅ Settings/configuration (LocalStorage)

**Moderate Difficulty**:
- ⚠️ **[OSCE] Timers**: Web timers + Web Push Notifications
- ⚠️ **Image caching**: Service Workers + Cache API
- ⚠️ **Offline support**: Service Workers + IndexedDB
- ⚠️ **Multi-device sync**: WebSockets for real-time

**Hard to Replicate**:
- ⚠️ **iPad-only UX**: Master-detail split view (needs responsive design)
- ⚠️ **Keychain storage**: Web Crypto API (different security model)
- ⚠️ **QR code scanning**: WebRTC camera access (permissions complex)
- ⚠️ **Background sync**: Limited on iOS Safari (workarounds needed)

---

### 4. PWA Limitations on iOS

**Known Issues**:
- **No Push Notifications**: iOS Safari doesn't fully support Web Push (as of iOS 16+, partial support added)
- **Limited Background Tasks**: Service Workers restricted on iOS
- **Storage Limits**: IndexedDB has stricter quotas than Core Data
- **Home Screen**: "Add to Home Screen" less discoverable than App Store

**Workarounds**:
- Use WebSockets for real-time updates (instead of push)
- Implement polling for background sync
- Warn users about storage limits
- Provide clear installation instructions

---

### 5. Migration Strategy

**Option A: Big Bang Rewrite**
- ❌ High risk
- ❌ No rollback plan
- ❌ Long development time (12-18 months)

**Option B: Incremental Migration** ✅ Recommended
1. **Phase 1**: Build PWA alongside existing apps (6 months)
2. **Phase 2**: Pilot test with 1-2 institutions (3 months)
3. **Phase 3**: Gradual rollout, keep iOS apps as fallback (6 months)
4. **Phase 4**: Full migration, deprecate iOS apps (ongoing)

**Option C: Hybrid Approach**
- Keep iOS apps for iPad users
- Provide PWA for web/desktop users
- Share backend API

---

### 6. Estimated PWA Migration Effort

| Task | Effort (Months) | Team Size |
|------|-----------------|-----------|
| **Frontend (React)** | 8 months | 2 developers |
| **Data Layer (IndexedDB + Sync)** | 4 months | 1 developer |
| **UI/UX Redesign** | 3 months | 1 designer |
| **Backend API Updates** | 2 months | 1 backend developer |
| **Testing & QA** | 3 months | 1 QA engineer |
| **Total** | **12 months** | **5 people** (with overlap) |

**Cost Estimate**: £300k - £500k (depending on team rates)

## Recommended Actions

### Immediate (Next Sprint)

1. ✅ **Update iOS Deployment Target** to iOS 15.0
   - Update project settings
   - Test on iOS 15 device
   - Update documentation

2. ✅ **Migrate OSCE to Native Logging**
   - Replace CocoaLumberjack with os_log
   - Remove CocoaPods dependency
   - Update log statements

3. ✅ **Document Current Architecture** (DONE!)
   - Create component diagrams (done)
   - Document design patterns (done)
   - Identify technical debt (this document)

---

### Short-term (Next Quarter)

4. **Add Unit Test Coverage**
   - Set up XCTest targets
   - Write tests for DSModel sync logic
   - Write tests for DSSettingsManager
   - Aim for 30% code coverage

5. **Refactor DSSettingsManager**
   - Replace institution checks with enum
   - Extract institution configuration to JSON file
   - Add new institutions without code changes

6. **Security Audit**
   - Review Keychain usage
   - Update SSL/TLS configuration
   - Check for hardcoded credentials
   - Ensure compliance with institution security policies

---

### Long-term (Next Year)

7. **Gradual Swift Migration**
   - Write new features in Swift
   - Migrate utility functions to Swift
   - Use Swift/Objective-C interop

8. **MVVM for Complex View Controllers**
   - Identify fat view controllers (>500 lines)
   - Extract business logic to ViewModels
   - Improve testability

9. **Evaluate PWA Migration**
   - Create PWA prototype
   - Pilot test with 1 institution
   - Decide on migration strategy

---

### Maintenance (Ongoing)

10. **Dependency Updates**
    - Update AFNetworking
    - Update Firebase Crashlytics
    - Monitor for security vulnerabilities

11. **Performance Monitoring**
    - Track sync performance
    - Monitor app crashes
    - Optimize slow view controllers

12. **Documentation**
    - Keep architecture docs updated
    - Document new features
    - Update API documentation

## Conclusion

The LIFTUPP platform has **moderate technical debt** that is manageable with focused refactoring. The shared library architecture is a strength, minimizing duplication between apps.

**Key Strengths**:
- ✅ Excellent code reuse (~65% shared)
- ✅ Clear separation of concerns (4-tier architecture)
- ✅ Well-documented (now with this repo)
- ✅ Production-stable with 13+ institutions

**Key Weaknesses**:
- ⚠️ iOS 12.0 deployment target (security risk)
- ⚠️ No unit test coverage (regression risk)
- ⚠️ Objective-C only (hiring/maintenance challenge)
- ⚠️ Singleton overuse (testability issue)

**Overall**: The platform is **well-architected for its age** and can be maintained long-term with incremental improvements. PWA migration is feasible but requires significant investment.

**Next Steps**: Focus on Priority 1 items (iOS 15, native logging, unit tests) before considering major architectural changes.

---

**For more information, see:**
- [Component Architecture](component-architecture.md) - Detailed component map
- [Architecture Overview](architecture-overview.md) - High-level system design
- [Data Model Reference](data-model.md) - Entity documentation
- [Sync Architecture](sync-architecture.md) - Synchronization details

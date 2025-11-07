# Technical Guide

## Table of Contents
1. [Development Setup](#development-setup)
2. [Project Structure](#project-structure)
3. [Build & Run](#build--run)
4. [Coding Standards](#coding-standards)
5. [Testing](#testing)
6. [Debugging](#debugging)
7. [Common Tasks](#common-tasks)
8. [Troubleshooting](#troubleshooting)

## Development Setup

### Prerequisites

- **macOS**: 11.0 (Big Sur) or later
- **Xcode**: 13.0 or later
- **iOS SDK**: 12.0+
- **CocoaPods**: 1.10.0 or later
- **Git**: 2.0 or later

### Initial Setup

1. **Clone the repository**:
   ```bash
   git clone https://github.com/DanSmithTII/osce-ios.git
   cd osce-ios
   ```

2. **Install dependencies**:
   ```bash
   pod install
   ```

3. **Open workspace** (NOT .xcodeproj):
   ```bash
   open OSCE.xcworkspace
   ```

4. **Select scheme**: Choose appropriate build configuration
   - **Debug**: Local development
   - **Debug Live**: Production server with debug logging
   - **Live**: Production release

### CocoaPods Dependencies

The project uses CocoaPods for dependency management. Key dependencies:

```ruby
# Podfile
platform :ios, '12.0'

target 'OSCE' do
  use_frameworks!

  pod 'AFNetworking', '~> 4.0'
  pod 'CocoaLumberjack', '~> 3.8'
  pod 'TPKeyboardAvoiding', '~> 1.3'
end
```

**Update dependencies**:
```bash
pod update
pod install
```

### Xcode Configuration

**Build Settings**:
- **Deployment Target**: iOS 12.0
- **Supported Devices**: iPad only
- **Supported Interface Orientations**: Landscape Left, Landscape Right
- **Swift Language Version**: Swift 5.0 (for KeychainStore/StringVault)
- **Objective-C Bridging Header**: `LIFTUPP/Supporting Files/LIFTUPP-Bridging-Header.h`

**Signing**:
- Configure automatic signing with your Apple Developer account
- Bundle ID: `com.liftupp.osce` (or your institution's ID)

## Project Structure

### Directory Organization

```
osce-ios/
├── OSCE.xcworkspace           # Open this in Xcode
├── OSCE.xcodeproj              # Project file
├── Podfile                     # CocoaPods dependencies
├── Pods/                       # Third-party libraries
│
├── LIFTUPP/                    # Main app source
│   ├── App Delegate/
│   │   └── AppDelegate.h/m
│   ├── Root View Controller/
│   │   ├── HomepageViewController.h/m
│   │   └── DSMasterViewController.h/m
│   ├── Clinic View Controller/
│   ├── Clinic Detail View Controller/
│   ├── Marking matrix/
│   ├── Timers/
│   ├── Supporting Files/
│   │   ├── OSCE-Info.plist
│   │   ├── LIFTUPP-Bridging-Header.h
│   │   ├── DSConfigurations.h/m
│   │   └── Configurations.plist
│   └── Images.xcassets/
│
├── liftupp-app-library/        # Shared library
│   ├── Data Model/
│   ├── View Controllers/
│   ├── LIFTUPPCategories/
│   ├── DS Views/
│   └── Initial App Configuration/
│
├── Model/                      # Core Data model
│   ├── LIFTUPP.xcdatamodeld/   # Core Data schema
│   ├── Entities/
│   │   ├── Generated/          # Auto-generated
│   │   └── Custom/             # Hand-written
│   ├── KeychainStore.swift
│   └── StringVault.swift
│
└── README.md
```

### Key Files

| File | Purpose |
|------|---------|
| `OSCE.xcworkspace` | Workspace (open this) |
| `AppDelegate.m` | App entry point |
| `LIFTUPP.xcdatamodeld` | Core Data model |
| `Configurations.plist` | Server URLs per build config |
| `OSCE-Info.plist` | App metadata |
| `Podfile` | CocoaPods dependencies |

## Build & Run

### Build Configurations

The project includes multiple build configurations for different environments:

| Configuration | Server | Purpose |
|--------------|--------|---------|
| **Debug** | qa-examsoft.com | Local development |
| **Debug Live** | lu-apiprod.examsoft.com | Production with logging |
| **Live** | lu-apiprod.examsoft.com | Production release |
| **Debug Nick/Dan/Adam** | Local dev servers | Developer-specific |

**Select configuration**:
1. Product → Scheme → Edit Scheme
2. Run → Build Configuration → Select config

### Running on Simulator

1. Select iPad simulator (any model)
2. Product → Run (⌘R)

**Note**: Some features unavailable in simulator:
- QR code scanning (camera required)
- Keychain may behave differently

### Running on Device

1. Connect iPad via USB
2. Select device in Xcode
3. Ensure signing is configured
4. Product → Run (⌘R)

**First Launch**:
- App will prompt for QR code configuration
- Use test QR code from web portal admin

### Build Output

**Artifacts**:
- `.app` bundle: `DerivedData/OSCE/Build/Products/Debug-iphoneos/OSCE.app`
- `.ipa` archive: Product → Archive → Export

**Build Logs**:
- Console output in Xcode console pane
- CocoaLumberjack logs to file (if configured)

## Coding Standards

### Language Conventions

#### Objective-C

**Naming**:
- Classes: `DSPrefixClassName` (DS = Dentistry/School)
- Methods: `camelCase` starting with lowercase
- Properties: `camelCase` starting with lowercase
- Constants: `kConstantName` or `CONSTANT_NAME`

**Example**:
```objc
@interface DSStudentAttendanceViewController : UIViewController

@property (nonatomic, strong) DSStudent *student;
@property (nonatomic) BOOL isPresent;

- (void)markStudentPresent:(DSStudent *)student;

@end

static NSString * const kDefaultStudentGroup = @"Unassigned";
```

#### Swift

**Used only for security components** (KeychainStore, StringVault).

**Naming**:
- Classes/Structs: `PascalCase`
- Methods/Properties: `camelCase`
- Constants: `camelCase` with `let`

**Example**:
```swift
struct StringVault {
    static func appUUID() -> String? {
        return KeychainStore.shared.get(forKey: "app_uuid")
    }
}
```

### Code Organization

**File Structure**:
```objc
// Header (.h)
#import <Foundation/Foundation.h>

@class DSRelatedClass;  // Forward declarations

@interface DSMyClass : NSObject

@property (nonatomic, strong) NSString *publicProperty;

- (void)publicMethod;

@end

// Implementation (.m)
#import "DSMyClass.h"
#import "DSRelatedClass.h"

@interface DSMyClass ()

@property (nonatomic, strong) NSString *privateProperty;

@end

@implementation DSMyClass

#pragma mark - Lifecycle

- (instancetype)init {
    if (self = [super init]) {
        _publicProperty = @"Default";
    }
    return self;
}

#pragma mark - Public Methods

- (void)publicMethod {
    // Implementation
}

#pragma mark - Private Methods

- (void)privateMethod {
    // Implementation
}

@end
```

**Pragma Marks**: Use to organize code sections
- `#pragma mark - Lifecycle`
- `#pragma mark - Public Methods`
- `#pragma mark - Private Methods`
- `#pragma mark - UITableViewDataSource`
- `#pragma mark - Notifications`

### Core Data Best Practices

1. **Always use contexts correctly**:
   ```objc
   [context performBlockAndWait:^{
       // Fetch or modify objects here
       [context save:nil];
   }];
   ```

2. **Use base class methods**:
   ```objc
   DSStudent *student = [DSStudent insert];  // Good
   // NOT: [[DSStudent alloc] initWithEntity:...]  // Bad
   ```

3. **Fetch efficiently**:
   ```objc
   NSPredicate *pred = [NSPredicate predicateWithFormat:@"year == %@", @"Year 3"];
   NSArray *students = [DSStudent fetchAllWithPredicate:pred];  // Good

   // NOT:
   NSArray *all = [DSStudent fetchAll];
   NSArray *filtered = [all filteredArrayUsingPredicate:pred];  // Bad (loads all)
   ```

4. **Save contexts properly**:
   ```objc
   [[DSModel sharedModel].managedObjectContext saveContextAndAncestorContexts];
   ```

### Memory Management

**Use ARC** (Automatic Reference Counting):
```objc
@property (nonatomic, strong) DSStudent *student;  // Strong reference
@property (nonatomic, weak) id<DSDelegate> delegate;  // Weak reference (avoid retain cycles)
@property (nonatomic, copy) NSString *name;  // Copy (for mutable objects)
```

**Avoid retain cycles**:
```objc
// BAD: Creates retain cycle
self.block = ^{
    [self doSomething];  // Captures self strongly
};

// GOOD: Use weak self
__weak typeof(self) weakSelf = self;
self.block = ^{
    [weakSelf doSomething];
};
```

### Logging

**Use CocoaLumberjack**:
```objc
DDLogDebug(@"Debug info: %@", variable);     // Development only
DDLogInfo(@"Info: %@", message);             // General info
DDLogWarn(@"Warning: %@", warning);          // Warnings
DDLogError(@"Error: %@", error);             // Errors
```

**Do NOT use**:
```objc
NSLog(@"...");  // Avoid (use DDLog instead)
```

## Testing

### Unit Testing

The project includes an `OSCE Tests` target for unit tests.

**Run tests**:
- Product → Test (⌘U)
- Or: Click test diamond in gutter next to test method

**Example test**:
```objc
@interface DSModelTests : XCTestCase
@end

@implementation DSModelTests

- (void)testStudentCreation {
    DSStudent *student = [DSStudent insert];
    student.forename = @"John";
    student.surname = @"Smith";

    XCTAssertNotNil(student);
    XCTAssertEqualObjects(student.forename, @"John");
}

- (void)testSyncPredicate {
    NSArray *pending = [DSDataForm fetchAllObjectsToSync];
    XCTAssertNotNil(pending);
}

@end
```

### Manual Testing

**Test Scenarios**:

1. **Offline Mode**:
   - Enable Airplane Mode on device
   - Perform assessment
   - Verify changes queued
   - Disable Airplane Mode
   - Verify auto-sync

2. **Sync Reliability**:
   - Create 100+ assessment forms
   - Trigger sync
   - Verify all uploaded
   - Check server data

3. **Timer Accuracy**:
   - Start station timer
   - Verify countdown accuracy
   - Test pause/resume
   - Test alerts (5 min, 2 min, 0 min)

4. **Multi-Device**:
   - Use 2 iPads
   - Create data on device A
   - Sync device A
   - Sync device B
   - Verify data appears on device B

## Debugging

### Xcode Debugger

**Breakpoints**:
1. Click line number gutter to add breakpoint
2. Run app (⌘R)
3. Execution pauses at breakpoint
4. Inspect variables in Variables View
5. Step through code: Step Over (F6), Step Into (F7), Continue (⌘^Y)

**Symbolic Breakpoint**:
- Debug → Breakpoints → Create Symbolic Breakpoint
- Symbol: `objc_exception_throw` (catches all exceptions)

**Conditional Breakpoint**:
- Right-click breakpoint → Edit Breakpoint
- Add condition: `student.dsid == 101`

### LLDB Commands

In debugger console:

```lldb
# Print object
(lldb) po student
<DSStudent: 0x...> {forename: "John", surname: "Smith"}

# Print variable
(lldb) p student.forename
(NSString *) $0 = @"John"

# Call method
(lldb) expr (void)[student markAsPresent]

# Print view hierarchy
(lldb) po [[[UIApplication sharedApplication] keyWindow] recursiveDescription]
```

### Logging

**Enable verbose logging**:
```objc
// In AppDelegate.m
[DDLog addLogger:[DDTTYLogger sharedInstance]];  // Console
[DDLog addLogger:[DDOSLogger sharedInstance]];   // System log

DDTTYLogger.sharedInstance.logFormatter = [[DSLogFormatter alloc] init];

// Set level
DDTTYLogger.sharedInstance.level = DDLogLevelVerbose;  // All logs
```

**Log levels**:
- `DDLogLevelOff`: No logging
- `DDLogLevelError`: Errors only
- `DDLogLevelWarning`: Warnings and errors
- `DDLogLevelInfo`: Info, warnings, errors
- `DDLogLevelDebug`: Debug and above
- `DDLogLevelVerbose`: Everything

### Core Data Debugging

**Enable SQL logging**:
1. Edit Scheme → Run → Arguments
2. Add launch argument: `-com.apple.CoreData.SQLDebug 1`

**Output**:
```
CoreData: sql: SELECT * FROM ZSTUDENT WHERE ZDSID = ?
```

**Enable concurrency debugging**:
1. Add launch argument: `-com.apple.CoreData.ConcurrencyDebug 1`
2. Crashes if Core Data accessed incorrectly across threads

### Network Debugging

**View HTTP requests**:
```objc
// In DSHTTPSessionManager
AFHTTPSessionManager *manager = [self defaultSessionManager];
[manager.requestSerializer setTimeoutInterval:60];

// Enable logging (already in code)
DDLogDebug(@"Request: %@", request.URL);
DDLogDebug(@"Response: %@", [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding]);
```

**Use proxy** (Charles, Proxyman):
1. Configure proxy on iPad: Settings → Wi-Fi → HTTP Proxy
2. Install proxy SSL certificate on iPad
3. View all HTTP traffic in proxy app

## Common Tasks

### Task 1: Add a New Entity

1. **Open Core Data model**:
   - Open `Model/LIFTUPP.xcdatamodeld/V_1.0.5.xcdatamodel`

2. **Add entity**:
   - Editor → Add Entity
   - Name: `DSNewEntity`
   - Parent: `NSManagedObject` (or `DSManagedObject` for sync support)

3. **Add attributes**:
   - Add attributes (dsid, name, etc.)
   - Set types (Integer 32, String, Boolean, etc.)

4. **Configure sync** (if needed):
   - Select entity
   - View → Utilities → Data Model Inspector
   - User Info → Add:
     - `uk.ac.liv.liftupp.coredata.sync` = `YES`
     - `uk.ac.liv.liftupp.coredata.sync.up` = `YES` (or `.down`)
     - `uk.ac.liv.liftupp.coredata.syncpoint` = `new_entity`
     - `uk.ac.liv.liftupp.coredata.pk` = `dsid`

5. **Generate classes**:
   - Select entity
   - Editor → Create NSManagedObject Subclass
   - Save to `Model/Entities/Generated/`

6. **Create custom class** (optional):
   - Create `Model/Entities/Custom/DSNewEntity.h/m`
   - Add custom methods

7. **Update Core Data version** (if changing existing model):
   - Editor → Add Model Version
   - Set as current version

### Task 2: Add a New View Controller

1. **Create files**:
   ```bash
   touch LIFTUPP/MyViewController.h
   touch LIFTUPP/MyViewController.m
   ```

2. **Header** (`MyViewController.h`):
   ```objc
   #import <UIKit/UIKit.h>

   @interface MyViewController : UIViewController

   @property (nonatomic, strong) DSStudent *student;

   @end
   ```

3. **Implementation** (`MyViewController.m`):
   ```objc
   #import "MyViewController.h"

   @interface MyViewController ()

   @property (nonatomic, strong) UITableView *tableView;

   @end

   @implementation MyViewController

   - (void)viewDidLoad {
       [super viewDidLoad];
       self.view.backgroundColor = [UIColor whiteColor];
       [self setupTableView];
   }

   - (void)setupTableView {
       // Setup code
   }

   @end
   ```

4. **Add to storyboard** (if using Interface Builder):
   - Open `MainStoryBoard.storyboard`
   - Drag View Controller from Object Library
   - Set Custom Class to `MyViewController`
   - Create segue from previous view controller

5. **Add to project**:
   - Drag files into Xcode
   - Ensure "OSCE" target is checked

### Task 3: Modify Sync Behavior

**Example: Add new field to sync**:

1. **Add attribute** to entity in Core Data model

2. **Configure sync** for attribute:
   - Select attribute
   - User Info → Add:
     - `uk.ac.liv.liftupp.coredata.sync.up` = `YES`

3. **Update server API** to handle new field

4. **Test**:
   - Create object with new field
   - Trigger sync
   - Verify field uploaded to server

### Task 4: Add Institution Support

1. **Add institution check** in `DSSettingsManager.h`:
   ```objc
   + (BOOL)isNewUniversity;
   ```

2. **Implement** in `DSSettingsManager.m`:
   ```objc
   + (BOOL)isNewUniversity {
       return [[[self sharedSettingsManager] institutionCode]
               caseInsensitiveCompare:@"new_university"] == NSOrderedSame;
   }
   ```

3. **Add configuration** in web portal

4. **Generate QR code** for new institution

5. **Test** configuration on iPad

## Troubleshooting

### Issue: Build Fails with "Pods not found"

**Solution**:
```bash
pod install
# Open OSCE.xcworkspace (NOT .xcodeproj)
```

### Issue: Core Data Migration Error

**Symptoms**: App crashes on launch with Core Data error

**Solution**:
1. Delete app from simulator/device
2. Clean build folder: Product → Clean Build Folder (⇧⌘K)
3. Rebuild and run

**For production**: Create Core Data migration mapping

### Issue: Sync Fails Repeatedly

**Debug steps**:
1. Check network: Is device online?
2. Check logs: Look for HTTP errors
3. Check credentials: App UUID/passcode valid?
4. Check server: Is API responding?
5. Test with curl:
   ```bash
   curl -X POST https://lu-apiprod.examsoft.com/syncClient \
     -u liftupp_ipad:iPad_LiftUpp_Pass \
     -d "data={...}"
   ```

### Issue: Memory Warning / Crash

**Debug steps**:
1. Instruments → Allocations: Track memory usage
2. Check for retain cycles: Instruments → Leaks
3. Profile app: Product → Profile (⌘I)

**Common causes**:
- Not using `@autoreleasepool` in loops
- Strong reference cycles
- Large images not released
- Fetching too many Core Data objects at once

### Issue: UI Freezing During Sync

**Solution**: Ensure sync operations on background thread
```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    // Perform sync
    dispatch_async(dispatch_get_main_queue(), ^{
        // Update UI
    });
});
```

### Issue: QR Code Scanner Not Working

**Symptoms**: Camera permission denied or scanner doesn't recognize code

**Solution**:
1. Check Info.plist: `NSCameraUsageDescription` present?
2. Check device settings: Camera permission granted?
3. Test with known working QR code from web portal

### Issue: Keychain Access Errors

**Symptoms**: `errSecItemNotFound` or similar keychain errors

**Solution**:
1. Delete app from device/simulator
2. Reset simulator: Device → Erase All Content and Settings
3. For device: Check that app is signed correctly

---

**For more information:**
- [Architecture Overview](architecture-overview.md)
- [Data Model Reference](data-model.md)
- [Shared Library Documentation](shared-library.md)
- [Sync Architecture](sync-architecture.md)

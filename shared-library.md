# Shared Library Documentation (liftupp-app-library)

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Components](#core-components)
4. [Categories & Extensions](#categories--extensions)
5. [Custom UI Components](#custom-ui-components)
6. [Usage Patterns](#usage-patterns)

## Overview

The **liftupp-app-library** is a shared codebase containing 80+ Objective-C classes that provide common functionality for both LIFTUPP iPad applications. This library encapsulates data management, networking, UI components, and utility functions.

### Directory Structure

```
liftupp-app-library/
├── Data Model/                     # Core Data & Sync
│   ├── DSModel.h/m                # Singleton Core Data manager
│   ├── DSManagedObject.h/m        # Base entity class
│   ├── DSManagedObjectContext.h/m # Enhanced context
│   ├── DSSyncManager.h/m          # Sync coordinator
│   └── DSBlockURLRequest/         # Networking
│       ├── DSHTTPSessionManager.h/m
│       ├── DSImageCache.h/m
│       └── DSRemoteImageManager.h/m
├── View Controllers/               # Shared view controllers
│   └── Main Clinic View Controllers/
│       ├── DSLiftuppSyncViewController.h/m
│       ├── DSClinicInstancesViewController.h/m
│       └── DSStaffSelectionTableViewCell.h/m
├── LIFTUPPCategories/             # 60+ Extension categories
│   ├── NSDate+*.h/m
│   ├── NSString+*.h/m
│   ├── UIView+*.h/m
│   └── UIViewController+*.h/m
├── DS Views/                       # Custom UI components
│   ├── DSView.h/m
│   ├── DSRoundedButton.h/m
│   ├── DSLabel.h/m
│   └── DSNavigationBarTitleSubtitle.h/m
├── Initial App Configuration/      # QR code setup
│   └── DSAppSetupViewController.h/m
└── Third Party Plugins/            # Embedded libs
    └── MBProgressHUD/
```

### Key Statistics

- **80+ Objective-C files** (40 headers, 40 implementations)
- **60+ category extensions** for Foundation/UIKit
- **20+ custom UI components**
- **3-tier architecture**: Data, Business Logic, Presentation
- **Thread-safe**: Proper use of Core Data contexts and GCD

## Architecture

### Layered Architecture

```
┌─────────────────────────────────────────────┐
│  View Controllers & UI Components          │  Presentation Layer
├─────────────────────────────────────────────┤
│  Business Logic (DSSettingsManager, etc.)  │  Business Layer
├─────────────────────────────────────────────┤
│  Data Access (DSModel, DSManagedObject)    │  Data Layer
├─────────────────────────────────────────────┤
│  Networking (DSHTTPSessionManager)         │  Infrastructure Layer
└─────────────────────────────────────────────┘
```

### Design Patterns

1. **Singleton**: DSModel, DSSettingsManager, DSSyncManager
2. **Repository**: DSManagedObject provides CRUD operations
3. **Observer**: Core Data notifications
4. **Category**: Code organization and extension
5. **Inheritance**: Base classes (DSView, DSManagedObject)

## Core Components

### DSModel - Core Data Manager

**File**: `liftupp-app-library/Data Model/DSModel.h`

**Purpose**: Central coordinator for Core Data stack and synchronization.

**Key Features**:
- Three-context architecture (Main, Sync, Saving)
- Cached date formatters for performance
- Background sync support
- Thread-safe operations

**Public Interface**:
```objc
@interface DSModel : NSObject

// Singleton
+ (DSModel *)sharedModel;

// Core Data Stack
@property (readonly, strong, nonatomic) NSManagedObjectContext *managedObjectContext;
@property (readonly, strong, nonatomic) NSManagedObjectContext *synchronizationManagedObjectContext;
@property (readonly, strong, nonatomic) NSManagedObjectModel *managedObjectModel;
@property (readonly, strong, nonatomic) NSPersistentStoreCoordinator *persistentStoreCoordinator;

// Date Formatters (Cached)
@property (nonatomic, strong, readonly) NSDateFormatter *yearFormatter;        // "2000"
@property (nonatomic, strong, readonly) NSDateFormatter *dateFormatter;        // "2000-12-21"
@property (nonatomic, strong, readonly) NSDateFormatter *dateTimeFormatter;    // "2000-12-21 10:00:00"
@property (nonatomic, strong, readonly) NSDateFormatter *timeFormatter;        // "10:00:00"

// Sync Operations
- (void)startSyncWithProgressChangeCallback:(DSModelSyncProgressChangeCallback)progressChangeCallback
                         stageChangeCallback:(DSModelSyncStageChangeCallback)stageChangeCallback
                                  completion:(DSModelSyncCompletionCallback)completion;

- (void)prepareUploadDataWithCompletion:(void(^)(NSDictionary *uploadData, NSSet *uploadObjects))completion;

- (void)processSyncData:(NSDictionary *)syncDictionary
        uploadedObjects:(NSSet *)uploadedObjects
               callback:(dispatch_block_t)callback
    stageChangeCallback:(DSModelSyncStageChangeCallback)stageChangeCallback
 progressChangeCallback:(DSModelSyncProgressChangeCallback)progressChangeCallback;

// State Management
- (void)saveState;
- (void)saveState:(dispatch_block_t)completion;

@end
```

**Usage Example**:
```objc
// Access singleton
DSModel *model = [DSModel sharedModel];

// Get main context
NSManagedObjectContext *context = model.managedObjectContext;

// Trigger sync
[model startSyncWithProgressChangeCallback:^(float progress) {
    NSLog(@"Sync progress: %.0f%%", progress * 100);
} stageChangeCallback:^(DSModelSyncProgressType stage) {
    NSLog(@"Sync stage: %ld", (long)stage);
} completion:^(DSSyncSuccessType success) {
    if (success == DSSyncSuccessTypeSuccess) {
        NSLog(@"Sync completed successfully");
    }
}];

// Format date
NSString *dateStr = [model.dateTimeFormatter stringFromDate:[NSDate date]];
```

### DSManagedObject - Base Entity Class

**File**: `liftupp-app-library/Data Model/DSManagedObject.h`

**Purpose**: Base class for all Core Data entities, providing CRUD operations and sync support.

**Key Features**:
- Static factory methods for entity creation
- Automatic type conversion (String ↔ Date, Number)
- Dictionary-based initialization from JSON
- Sync change tracking

**Public Interface**:
```objc
@interface DSManagedObject : NSManagedObject

// CRUD Operations
+ (id)insert;
+ (id)insertInManagedObjectContext:(NSManagedObjectContext *)context;
+ (NSArray *)fetchAll;
+ (NSArray *)fetchAllInManagedObjectContext:(NSManagedObjectContext *)context;
+ (id)managedObjectWithId:(NSString *)uniqueValue;
+ (id)managedObjectWithId:(NSString *)uniqueValue 
     managedObjectContext:(NSManagedObjectContext *)context;

// Dictionary Creation
+ (id)managedObjectWithDictionary:(NSDictionary *)dictionary;
+ (id)managedObjectWithDictionary:(NSDictionary *)dictionary 
             managedObjectContext:(NSManagedObjectContext *)context;

// Sync Support
+ (NSArray *)fetchAllObjectsToSync;
+ (NSArray *)fetchAllObjectsToSyncInManagedObjectContext:(NSManagedObjectContext *)context;
- (void)markAsSynced;
- (NSDictionary *)syncUpDictionary;
- (id)syncValueForKey:(NSString *)key;

// Entity Metadata
+ (NSEntityDescription *)entity;
+ (NSString *)entityName;
+ (NSString *)uniqueIdentifierPropertyName;

// Last Update Tracking
+ (NSString *)lastUpdate;
+ (void)setLastUpdate:(NSString *)lastUpdate;

// Sync Flags
@property (nonatomic) NSNumber *syncNew;
@property (nonatomic) NSNumber *syncError;
@property (nonatomic) NSSet *syncChangedKeys;

@end
```

**Usage Example**:
```objc
// Create new entity
DSStudent *student = [DSStudent insert];
student.forename = @"John";
student.surname = @"Smith";
student.studentNumber = @"12345";

// Fetch all students
NSArray *students = [DSStudent fetchAll];

// Fetch by ID
DSStudent *student = [DSStudent managedObjectWithId:@"101"];

// Create from dictionary (JSON from server)
NSDictionary *dict = @{
    @"dsid": @"101",
    @"forename": @"Emma",
    @"surname": @"Johnson",
    @"email": @"emma@example.com"
};
DSStudent *emma = [DSStudent managedObjectWithDictionary:dict];

// Fetch objects needing sync
NSArray *pendingSync = [DSDataForm fetchAllObjectsToSync];

// Mark as synced
[dataForm markAsSynced];
```

**Automatic Type Conversion**:
```objc
// String → Date
[object setValue:@"2023-10-15" forKey:@"date"];  // Automatically converts to NSDate

// String → Number
[object setValue:@"42" forKey:@"count"];         // Automatically converts to NSNumber

// Date → String (for sync)
id syncValue = [object syncValueForKey:@"date"]; // Returns "2023-10-15 10:00:00"
```

### DSSyncManager - Sync Coordinator

**File**: `liftupp-app-library/Data Model/DSSyncManager.h`

**Purpose**: Manages sync requests and responses.

**Public Interface**:
```objc
typedef NS_ENUM(NSInteger, DSSyncSuccessType) {
    DSSyncSuccessTypeNone,
    DSSyncSuccessTypeSuccess,
    DSSyncSuccessTypeFailure,
    DSSyncSuccessTypeFailureOffline
};

typedef void(^DSSyncCompletionBlock)(NSDictionary *syncDictionary, DSSyncSuccessType successType);

@interface DSSyncManager : NSObject

+ (instancetype)manager;

- (void)syncPostData:(NSDictionary *)postData 
      withCompletion:(DSSyncCompletionBlock)completion;

- (NSString *)lastSyncTimeString;  // "2 hours ago", "Yesterday", etc.

@end
```

**Usage Example**:
```objc
DSSyncManager *syncMgr = [DSSyncManager manager];

NSDictionary *uploadData = @{
    @"key": @{@"uuid": @"...", @"passcode": @"..."},
    @"data": @{...}
};

[syncMgr syncPostData:uploadData withCompletion:^(NSDictionary *response, DSSyncSuccessType success) {
    if (success == DSSyncSuccessTypeSuccess) {
        NSLog(@"Sync successful: %@", response);
    } else if (success == DSSyncSuccessTypeFailureOffline) {
        NSLog(@"Device is offline");
    } else {
        NSLog(@"Sync failed");
    }
}];

// Display last sync time
NSString *lastSync = [syncMgr lastSyncTimeString];  // "2 hours ago"
```

### DSHTTPSessionManager - Network Manager

**File**: `liftupp-app-library/Data Model/DSBlockURLRequest/DSHTTPSessionManager.h`

**Purpose**: Wrapper around AFNetworking for API requests.

**Public Interface**:
```objc
@interface DSHTTPSessionManager : AFHTTPSessionManager

+ (instancetype)defaultSessionManager;

// API Endpoints
+ (void)appSetup:(NSDictionary *)parameters 
         success:(void (^)(NSData *data))success 
         failure:(void (^)(NSError *error))failure;

+ (void)syncClient:(NSDictionary *)parameters 
           success:(void (^)(NSData *data))success 
           failure:(void (^)(NSError *error))failure;

+ (void)getStationAttendances:(NSNumber *)dsid 
                      success:(void (^)(NSDictionary *response))success 
                      failure:(void (^)(NSError *error))failure;

+ (void)toggleAssistanceRequiredWithURLString:(NSString *)urlString 
                                      success:(void (^)(NSString *response))success 
                                      failure:(void (^)(NSError *error))failure;

// Reachability
- (void)startReachabilityMonitoring;
- (BOOL)isNetworkAvailable;

@end
```

**Usage Example**:
```objc
// Check network
if ([[DSHTTPSessionManager defaultSessionManager] isNetworkAvailable]) {
    // Sync data
    [DSHTTPSessionManager syncClient:uploadData success:^(NSData *data) {
        NSLog(@"Sync successful");
    } failure:^(NSError *error) {
        NSLog(@"Sync failed: %@", error);
    }];
}

// Get station attendances
[DSHTTPSessionManager getStationAttendances:@(42) success:^(NSDictionary *response) {
    NSArray *attendances = response[@"attendances"];
} failure:^(NSError *error) {
    NSLog(@"Failed to fetch attendances");
}];
```

### DSSettingsManager - Configuration Manager

**File**: `liftupp-app-library/Data Model/DSSettingsManager.h`

**Purpose**: Manages multi-institution configuration.

**Public Interface**:
```objc
@interface DSSettingsManager : NSObject

+ (DSSettingsManager *)sharedSettingsManager;

// Institution
@property (nonatomic, strong) NSString *institutionCode;      // "liverpool", "manchester"
@property (nonatomic, strong) NSString *institutionName;      // "University of Liverpool"
@property (nonatomic, strong) NSString *institutionType;      // "Dentistry", "Medicine"
@property (nonatomic, strong) UIImage *institutionLogo;

// Branding
@property (nonatomic, strong) UIColor *mainColor;
@property (nonatomic, strong) UIColor *secondaryColor;
- (void)setMainColorWithHexCode:(NSString *)hexColorCode;
- (void)setSecondaryColorWithHexCode:(NSString *)hexColorCode;

// API URLs (Keychain-backed)
@property (nonatomic, strong) NSArray *syncClientServerURLs;
@property (nonatomic, strong) NSURL *syncClientServerURL;  // First URL in array
@property (nonatomic, strong) NSString *syncStudentImageVariableURLString;
@property (nonatomic, strong) NSString *syncStaffImageVariableURLString;

// Configuration
+ (void)configureAppWithJSON:(NSDictionary *)json 
            configurationURL:(NSURL *)configurationURL 
             completionBlock:(void(^)(DSConfigurationSuccessType success))callback;

+ (void)reconfigureAppSettings:(NSURL *)configurationURL;

// Institution Checks
+ (BOOL)isLiverpool;
+ (BOOL)isManchester;
+ (BOOL)isQMUL;
+ (BOOL)isKings;
// ... more institution checks

@end
```

**Usage Example**:
```objc
DSSettingsManager *settings = [DSSettingsManager sharedSettingsManager];

// Get institution
NSString *institution = settings.institutionName;  // "University of Liverpool"

// Set brand colors
[settings setMainColorWithHexCode:@"#3FA9F5"];

// Get API URL
NSURL *apiURL = settings.syncClientServerURL;

// Configure app from QR code
NSDictionary *config = /* parsed from QR code */;
[DSSettingsManager configureAppWithJSON:config 
                       configurationURL:url 
                        completionBlock:^(DSConfigurationSuccessType success) {
    if (success == DSConfigurationSuccessTypeSuccess) {
        NSLog(@"App configured for %@", settings.institutionName);
    }
}];

// Check institution
if ([DSSettingsManager isLiverpool]) {
    // Liverpool-specific logic
}
```

## Categories & Extensions

### Date Categories

#### NSDate+DateArithmetic
```objc
+ (NSDate *)date:(NSDate *)date withAdditionalDays:(NSInteger)days;
- (NSDate *)dateAdditionalDays:(NSInteger)days;
- (NSDate *)dateAdditionalWeeks:(NSInteger)weeks;
- (NSDate *)dateAdditionalMonths:(NSInteger)months;
```

**Example**:
```objc
NSDate *tomorrow = [[NSDate date] dateAdditionalDays:1];
NSDate *nextWeek = [[NSDate date] dateAdditionalWeeks:1];
```

#### NSDate+DateChecks
```objc
+ (NSDate *)largestDate:(NSArray *)dates;
+ (BOOL)dateIsToday:(NSDate *)date;
+ (BOOL)dateIsYesterday:(NSDate *)date;
+ (NSTimeInterval)timeBetweenDates:(NSDate *)first secondDate:(NSDate *)second;
```

**Example**:
```objc
if ([NSDate dateIsToday:someDate]) {
    NSLog(@"Date is today");
}

NSDate *latest = [NSDate largestDate:arrayOfDates];
```

#### NSDate+InBetweenChecks
```objc
+ (BOOL)date:(NSDate *)date isWithinLastMinutes:(NSInteger)minutes;
+ (BOOL)date:(NSDate *)date isWithinLastHours:(NSInteger)hours;
```

**Example**:
```objc
// Check if synced within last 10 minutes
if ([NSDate date:lastSyncDate isWithinLastMinutes:10]) {
    NSLog(@"Skip sync (too recent)");
}
```

### UIKit Categories

#### UIColor+LIFTUPP
```objc
+ (UIColor *)liftuppColor;              // Default LIFTUPP blue
+ (UIColor *)liftuppSecondaryColor;
+ (UIColor *)liftUppGreenColor;
+ (UIColor *)liftUppBlueColor;
+ (UIColor *)liftuppNavyBlueColor;
+ (UIColor *)colorFromHexString:(NSString *)hexString;
```

**Example**:
```objc
self.view.backgroundColor = [UIColor liftuppColor];
UIColor *custom = [UIColor colorFromHexString:@"#3FA9F5"];
```

#### UIViewController+DSNavigationTitleSubtitle
```objc
- (void)setTitle:(NSString *)title 
        subtitle:(NSString *)subtitle 
        withFont:(UIFont *)font 
           color:(UIColor *)color;
```

**Example**:
```objc
[self setTitle:@"Clinical Assessment" 
      subtitle:@"Year 3 - Monday AM" 
      withFont:[UIFont systemFontOfSize:14] 
         color:[UIColor whiteColor]];
```

#### UIView+Borders
```objc
- (void)addTopBorder:(UIColor *)borderColor width:(CGFloat)width;
- (void)addBottomBorder:(UIColor *)borderColor width:(CGFloat)width;
- (void)addLeftBorder:(UIColor *)borderColor width:(CGFloat)width;
- (void)addRightBorder:(UIColor *)borderColor width:(CGFloat)width;
- (void)addAllBorder:(UIColor *)borderColor width:(CGFloat)width cornerRadius:(CGFloat)radius;
```

**Example**:
```objc
[tableView addTopBorder:[UIColor lightGrayColor] width:1.0];
[button addAllBorder:[UIColor liftuppColor] width:2.0 cornerRadius:8.0];
```

#### NSString+UUID
```objc
+ (NSString *)uuid;  // Generate UUID string
```

**Example**:
```objc
NSString *uuid = [NSString uuid];  // "A1B2C3D4-E5F6-..."
```

### Complete Category List

| Category | Purpose |
|----------|---------|
| **NSDate+DateArithmetic** | Add days/weeks/months |
| **NSDate+DateChecks** | Is today/yesterday, time between dates |
| **NSDate+InBetweenChecks** | Within last N minutes/hours |
| **NSDateFormatter+Reusable** | Cached date formatters |
| **NSString+DateFormatting** | Parse/format date strings |
| **NSString+UUID** | Generate UUIDs |
| **NSString+Hashes** | MD5, SHA1 hashing |
| **NSString+URLEncoding** | URL encoding/decoding |
| **NSString+NumberChecking** | Validate numeric strings |
| **NSArray+EasyAccess** | Safe array access |
| **NSDictionary+HCBasicTypes** | Type-safe dictionary accessors |
| **NSDictionary+MergedDictionaries** | Merge dictionaries |
| **UIColor+LIFTUPP** | Brand colors |
| **UIColor+HexStringConversion** | Hex color parsing |
| **UIView+Borders** | Add borders to views |
| **UIView+Additions** | Misc view utilities |
| **UIView+Shakes** | Shake animation |
| **UIViewController+DSNavigationTitleSubtitle** | Title with subtitle |
| **UIViewController+DSButtons** | Common button actions |
| **UIViewController+DSDelayedBlocks** | Delayed execution |
| **UITableView+DSStyle** | Institution-specific styling |
| **UITableView+EasyNibRegistration** | Register cell nibs |
| **NSManagedObjectContext+DSModelAdditions** | Context helpers |
| **NSEntityDescription+DSUserInfo** | Entity metadata |
| **NSPropertyDescription+DSUserInfo** | Property metadata |
| **NSFetchedResultsController+Subscripting** | frc[index] syntax |

## Custom UI Components

### DSView - Base View

**File**: `liftupp-app-library/DS Views/DSView.h`

**Purpose**: Base class for custom views with setup hook.

```objc
@interface DSView : UIView

- (void)setupView;  // Override in subclasses

@end

@implementation DSView

- (instancetype)initWithFrame:(CGRect)frame {
    if (self = [super initWithFrame:frame]) {
        [self setupView];
    }
    return self;
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    if (self = [super initWithCoder:coder]) {
        [self setupView];
    }
    return self;
}

- (void)setupView {
    // Override in subclasses
}

@end
```

**Usage**:
```objc
@interface MyCustomView : DSView
@end

@implementation MyCustomView

- (void)setupView {
    [super setupView];
    self.backgroundColor = [UIColor whiteColor];
    // Add subviews, constraints, etc.
}

@end
```

### DSRoundedButton - Styled Button

**File**: `liftupp-app-library/DS Views/DSRoundedButton.h`

**Purpose**: Pre-styled button with rounded corners and borders.

```objc
@interface DSRoundedButton : UIButton
@end

@implementation DSRoundedButton

- (void)awakeFromNib {
    [super awakeFromNib];
    
    self.layer.cornerRadius = 8.0;
    self.layer.borderWidth = 2.0;
    self.layer.borderColor = [UIColor liftuppColor].CGColor;
    self.titleLabel.font = [UIFont boldSystemFontOfSize:16];
    
    // Selected state
    if (self.selected) {
        self.backgroundColor = [UIColor liftUppGreenColor];
    }
}

- (void)setSelected:(BOOL)selected {
    [super setSelected:selected];
    if (selected) {
        self.backgroundColor = [UIColor liftUppGreenColor];
    } else {
        self.backgroundColor = [UIColor clearColor];
    }
}

@end
```

**Usage**: Just use `DSRoundedButton` in Interface Builder or code.

### DSLiftuppSyncViewController - Sync Base VC

**File**: `liftupp-app-library/View Controllers/Main Clinic View Controllers/DSLiftuppSyncViewController.h`

**Purpose**: Base view controller with built-in sync functionality.

```objc
typedef void(^DSSyncCallbackBlock)(void);

@interface DSLiftuppSyncViewController : UIViewController

@property (nonatomic, copy) DSSyncCallbackBlock startSyncBlock;
@property (nonatomic, copy) DSSyncCallbackBlock successfulEndSyncBlock;
@property (nonatomic, copy) DSSyncCallbackBlock offlineEndSyncBlock;
@property (nonatomic, copy) DSSyncCallbackBlock failedEndSyncBlock;
@property (nonatomic, copy) DSSyncCallbackBlock endSyncBlock;

- (void)sync;

@end
```

**Usage**:
```objc
@interface MyViewController : DSLiftuppSyncViewController
@end

@implementation MyViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.successfulEndSyncBlock = ^{
        NSLog(@"Sync succeeded, refresh UI");
        [self.tableView reloadData];
    };
    
    self.offlineEndSyncBlock = ^{
        NSLog(@"Device is offline");
        [self showOfflineAlert];
    };
}

- (void)refreshButtonTapped {
    [self sync];  // Triggers sync with progress HUD
}

@end
```

## Usage Patterns

### Pattern 1: Fetching Entities

```objc
// Fetch all
NSArray *students = [DSStudent fetchAll];

// Fetch with predicate
NSPredicate *pred = [NSPredicate predicateWithFormat:@"year == %@", @"Year 3"];
NSArray *year3 = [DSStudent fetchAllWithPredicate:pred];

// Fetch by ID
DSStudent *student = [DSStudent managedObjectWithId:@"101"];

// Fetch in specific context
NSArray *students = [DSStudent fetchAllInManagedObjectContext:syncContext];
```

### Pattern 2: Creating Entities

```objc
// Create in main context
DSStudent *student = [DSStudent insert];
student.forename = @"John";
student.surname = @"Smith";
[[DSModel sharedModel].managedObjectContext save:nil];

// Create from dictionary
NSDictionary *dict = @{@"dsid": @"101", @"forename": @"Emma"};
DSStudent *emma = [DSStudent managedObjectWithDictionary:dict];
```

### Pattern 3: Performing Sync

```objc
[[DSModel sharedModel] startSyncWithProgressChangeCallback:^(float progress) {
    self.progressView.progress = progress;
} stageChangeCallback:^(DSModelSyncProgressType stage) {
    self.statusLabel.text = [self stringForStage:stage];
} completion:^(DSSyncSuccessType success) {
    if (success == DSSyncSuccessTypeSuccess) {
        [self.tableView reloadData];
    }
}];
```

### Pattern 4: Date Formatting

```objc
// Use cached formatters
NSDateFormatter *formatter = [DSModel sharedModel].dateTimeFormatter;
NSString *dateStr = [formatter stringFromDate:[NSDate date]];

// Or use category
NSString *uuid = [NSString uuid];
```

### Pattern 5: UI Customization

```objc
// Set institution colors
self.view.backgroundColor = [[DSSettingsManager sharedSettingsManager] mainColor];

// Add borders
[self.headerView addBottomBorder:[UIColor lightGrayColor] width:1.0];

// Rounded button (automatic via DSRoundedButton class)
```

---

**For related documentation, see:**
- [Architecture Overview](architecture-overview.md)
- [Data Model Reference](data-model.md)
- [Sync Architecture](sync-architecture.md)

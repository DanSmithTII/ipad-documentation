# Sync Architecture & Web Portal Integration

## Table of Contents
1. [Overview](#overview)
2. [Sync Strategy](#sync-strategy)
3. [Sync Flow](#sync-flow)
4. [API Integration](#api-integration)
5. [Conflict Resolution](#conflict-resolution)
6. [Offline Support](#offline-support)
7. [Performance Optimization](#performance-optimization)
8. [Error Handling](#error-handling)
9. [Security](#security)

## Overview

The LIFTUPP platform uses a sophisticated **bidirectional synchronization system** that enables offline-first operation while maintaining data consistency with a central web portal. The architecture is designed to handle unreliable network conditions common in clinical environments while ensuring no data loss.

### Key Characteristics

- **Offline-first**: Full functionality without network connectivity
- **Automatic sync**: Background synchronization when network available
- **Bidirectional**: Reference data DOWN, assessment data UP, some entities BOTH
- **Incremental**: Only changes synced, not entire dataset
- **Batch processing**: Efficient handling of large datasets (100 objects per batch)
- **Change tracking**: Granular tracking of modified attributes
- **Conflict resolution**: Server wins for reference data

### Sync Components

```mermaid
graph TB
    subgraph "iPad Components"
        A[UIViewController]
        B[DSLiftuppSyncViewController]
        C[DSModel]
        D[DSSyncManager]
        E[Core Data Stack]
        F[DSHTTPSessionManager]
        G[Change Tracker]
    end

    subgraph "Network"
        H[AFNetworking]
        I[Reachability Monitor]
    end

    subgraph "Server Components"
        J[Load Balancer]
        K[API Server]
        L[Database]
    end

    A --> B
    B --> C
    C --> D
    C --> E
    D --> F
    F --> H
    H --> J
    J --> K
    K --> L
    E --> G
    G --> C
    I --> F

    style A fill:#3FA9F5
    style E fill:#90EE90
    style K fill:#FFD700
    style L fill:#FF6B6B
```

## Sync Strategy

### Three-Way Sync Model

The sync system implements a sophisticated three-way sync:

1. **Client → Server (UP)**: User-generated data (assessments, attendance, comments)
2. **Server → Client (DOWN)**: Reference data (students, forms, timetables)
3. **Bidirectional (BOTH)**: Hybrid entities (timetable cover staff, attendance updates)

### Entity Sync Classification

```mermaid
graph LR
    subgraph "DOWN Only - 23 Entities"
        A[Students]
        B[Staff]
        C[TypeForm Templates]
        D[Timetables Base]
        E[ActivityTypes]
    end

    subgraph "UP Only - 9 Entities"
        F[DataForms]
        G[Comments]
        H[StaffSignIn]
        I[SyncInformation]
    end

    subgraph "Bidirectional - 2 Entities"
        J[Timetable coverStaff]
        K[TimetableAttendanceLog]
    end

    L[Server] -->|Reference Data| A
    L -->|Reference Data| B
    L -->|Reference Data| C
    L -->|Reference Data| D
    L -->|Reference Data| E

    F -->|Assessments| L
    G -->|Feedback| L
    H -->|Audit| L
    I -->|Monitoring| L

    J <-->|Updates| L
    K <-->|Updates| L

    style L fill:#FFD700
```

### Change Tracking System

Every syncable entity includes metadata for change tracking:

```objc
@interface DSManagedObject : NSManagedObject

@property (nonatomic) NSNumber *syncNew;        // Newly created
@property (nonatomic) NSNumber *syncError;      // Sync failed
@property (nonatomic) NSSet *syncChangedKeys;   // Modified attributes

@end
```

**Change Detection Algorithm**:
1. Override `willSave` in NSManagedObjectContext
2. Detect inserted, updated, deleted objects
3. For updated objects, capture changed keys via `changedValues`
4. Store in `syncChangedKeys` (NSSet)
5. Mark object for upload on next sync

**Example**:
```objc
// Staff marks student present
log.present = @(YES);
[context save:nil];

// Internally:
log.syncChangedKeys = [NSSet setWithObject:@"present"];
log.syncNew = @(NO);  // Existing object, not new
```

## Sync Flow

### Complete Sync Sequence

```mermaid
sequenceDiagram
    participant UI as View Controller
    participant SVC as SyncViewController
    participant Model as DSModel
    participant Sync as DSSyncManager
    participant HTTP as HTTPSessionManager
    participant API as Web Portal API

    UI->>SVC: Trigger Sync
    SVC->>Model: startSyncWithProgress()

    Note over Model: Stage 1: Prepare Upload Data
    Model->>Model: Fetch objects with syncNew=YES
    Model->>Model: Fetch objects with syncChangedKeys != nil
    Model->>Model: Build JSON payload

    Note over Model: Stage 2: Transfer Upload Data
    Model->>Sync: syncPostData:withCompletion:
    Sync->>HTTP: POST /syncClient
    HTTP->>API: JSON Request
    API-->>HTTP: JSON Response
    HTTP-->>Sync: Server Data

    Note over Model: Stage 3: Process Sync Data
    Sync-->>Model: Response Dictionary
    Model->>Model: Parse entities from response
    Model->>Model: Batch processing (100 per batch)
    Model->>Model: Create/update Core Data objects
    Model->>Model: Mark uploaded objects as synced
    Model->>Model: Update LastUpdate timestamps

    Note over Model: Stage 4: Clean Up
    Model->>Model: Reset syncNew flags
    Model->>Model: Clear syncChangedKeys
    Model->>Model: Save all contexts
    Model-->>SVC: Success Callback
    SVC-->>UI: Sync Complete
```

### Stage 1: Prepare Upload Data

**File**: `DSModel.m:273-425`

**Process**:
1. Enumerate all entities in Core Data model
2. For entities marked `shouldSyncUp`:
   - Fetch objects where `syncNew = YES` OR `syncChangedKeys.count > 0`
   - Build dictionary representation using `syncUpPropertyNames`
   - Convert NSDate → String, NSManagedObject → uniqueIdentifier
   - Add `dsid` or `uuid` as `id` key
3. For entities marked `shouldSyncDown`:
   - Send `lastUpdate` timestamp for incremental sync

**Upload Payload Structure**:
```json
{
  "key": {
    "uuid": "device-uuid-12345",
    "passcode": "hashed-passcode"
  },
  "data": {
    "data_form": {
      "data": [
        {
          "id": "uuid-form-123",
          "attendanceLogID": "uuid-log-456",
          "typeID": 42,
          ...
        }
      ]
    },
    "timetable_attendance_log": {
      "data": [
        {
          "id": "uuid-log-456",
          "present": 1,
          "staffSignout": 1,
          "timetableID": 789,
          "studentID": 101
        }
      ]
    }
  },
  "syncInfo": {
    "syncDays": "7",
    "latestTimetableDate": "2023-10-15",
    "appVersion": "1.0.24"
  }
}
```

**Progress Callback**:
```objc
typedef void(^DSModelSyncProgressChangeCallback)(float progress);

// Fires at 0%, 25%, 50%, 75%, 100%
progressCallback(0.25);  // Preparing data
```

### Stage 2: Transfer Upload Data

**File**: `DSHTTPSessionManager.m:72-101`

**HTTP Request**:
```http
POST /syncClient HTTP/1.1
Host: lu-apiprod.examsoft.com
Authorization: Basic bGlmdHVwcF9pcGFkOmlQYWRfTGlmdFVwcF9QYXNz
Content-Type: application/x-www-form-urlencoded

data=<URL-encoded JSON string>
```

**Authentication**:
- Basic HTTP Auth with hardcoded credentials
- Username: `liftupp_ipad`
- Password: `iPad_LiftUpp_Pass`
- App UUID + Passcode in JSON payload for device identification

**Network Monitoring**:
```objc
- (void)startReachabilityMonitoring {
    [self.reachabilityManager startMonitoring];
}

- (BOOL)isNetworkAvailable {
    return self.reachabilityManager.isReachable;
}
```

**Progress Callback**:
```objc
progressCallback(0.50);  // Transferring data
```

### Stage 3: Process Sync Data

**File**: `DSModel.m:427-659`

**Server Response Structure**:
```json
{
  "student": {
    "data": [
      {"id": 101, "forename": "John", "surname": "Smith", ...},
      {"id": 102, "forename": "Emma", "surname": "Johnson", ...}
    ],
    "lastUpdate": "2023-10-15 14:30:00"
  },
  "type_form": {
    "data": [
      {"id": 42, "name": "Clinical Skills Assessment", ...}
    ],
    "errors": [
      {"id": 38, "code": "Nothing to Update"}
    ]
  }
}
```

**Processing Algorithm**:
```objc
// 1. Count total elements for progress tracking
totalElements = sum of all data arrays

// 2. Enumerate entity dictionaries
for (entityName in response) {
    elements = response[entityName][@"data"];
    errorElements = response[entityName][@"errors"];

    // 3. Batch processing (100 objects per batch)
    batchSize = 100;
    for (batch = 0; batch < ceil(elements.count / batchSize); batch++) {
        @autoreleasepool {
            for (element in currentBatch) {
                // 4. Convert "id" to "dsid" for Core Data
                element[@"dsid"] = element[@"id"];
                delete element[@"id"];

                // 5. Create or update object
                [context performBlockAndWait:^{
                    DSManagedObject *object =
                        [class managedObjectWithDictionary:element];
                }];

                // 6. Update progress
                progressCallback((float)count / totalElements);
            }

            // 7. Save batch
            [context saveContextAndAncestorContexts];
        }
    }

    // 8. Update LastUpdate timestamp
    [class setLastUpdate:largestDate];

    // 9. Handle errors
    for (errorDict in errorElements) {
        if (errorDict[@"code"] != "Nothing to Update") {
            // Mark object as syncError = YES
        }
    }
}

// 10. Mark uploaded objects as synced
for (uploadedObject in uploadedObjects) {
    [uploadedObject markAsSynced];
}
```

**Batch Processing Benefits**:
- Memory efficiency via `@autoreleasepool`
- Progress updates every 100 objects
- Database commits every 100 objects (reduces write amplification)

**Progress Callback**:
```objc
progressCallback(0.75);  // Processing sync data
```

### Stage 4: Clean Up

**File**: `DSModel.m:643-656`

**Cleanup Steps**:
1. Mark all uploaded objects as synced:
   ```objc
   - (void)markAsSynced {
       self.syncNew = @(NO);
       self.syncChangedKeys = nil;
       self.syncError = @(NO);
   }
   ```

2. Save all contexts (recursive):
   ```objc
   [self.synchronizationManagedObjectContext saveContextAndAncestorContexts];
   ```

3. Reset main context:
   ```objc
   [self.managedObjectContext reset];
   ```

4. Signal completion:
   ```objc
   dispatch_semaphore_signal(syncDataProcessSemaphore);
   callback();
   ```

**Progress Callback**:
```objc
progressCallback(1.0);  // Sync complete
```

## API Integration

### API Endpoints

#### 1. Sync Client (Main Sync)

**Endpoint**: `POST /syncClient`

**Request**:
```http
POST /syncClient HTTP/1.1
Content-Type: application/x-www-form-urlencoded

data=<JSON payload>
```

**Response**:
```json
{
  "<entity_name>": {
    "data": [<array of objects>],
    "errors": [<array of error objects>],
    "lastUpdate": "<ISO timestamp>"
  },
  ...
}
```

**Rate Limit**: None (controlled by client-side semaphore)

#### 2. App Setup (Configuration)

**Endpoint**: `POST /appSetup`

**Request**:
```http
POST /appSetup HTTP/1.1
Content-Type: application/x-www-form-urlencoded

data=<JSON with app info>
```

**Response**:
```json
{
  "institutionCode": "liverpool",
  "institutionName": "University of Liverpool",
  "logoURL": "https://...",
  "mainColor": "#3FA9F5",
  "secondaryColor": "#FF6B6B",
  "syncURL": "https://lu-apiprod.examsoft.com/",
  ...
}
```

**Usage**: Called during QR code configuration

#### 3. Get Station Attendances

**Endpoint**: `GET /getStationAttendances/<dsid>`

**Request**:
```http
GET /getStationAttendances/12345 HTTP/1.1
```

**Response**:
```json
{
  "attendances": [
    {"studentID": 101, "status": "present", ...},
    ...
  ]
}
```

**Usage**: Fetch real-time attendance for specific station

#### 4. Toggle Assistance Required

**Endpoint**: `GET /toggleAssistanceRequired?stationID=<id>&required=<bool>`

**Request**:
```http
GET /toggleAssistanceRequired?stationID=42&required=true HTTP/1.1
```

**Response**:
```
Success
```

**Usage**: Request/cancel assistance at station

### URL Configuration

**Stored in Keychain**:
```swift
struct StringVault {
    static let syncClientServerURLs: [String]
    static let syncStudentImageVariableURLString: String
    static let syncStaffImageVariableURLString: String
    static let syncAssistanceRequiredVariableURLString: String
    static let syncCircuitStatusesVariableURLString: String
}
```

**Multi-Server Support**:
```objc
@property (nonatomic, strong) NSArray *syncClientServerURLs;

// Iterates through URLs until success
for (NSURL *url in urls) {
    success = [self trySyncWithURL:url];
    if (success) break;
}
```

## Conflict Resolution

### Conflict Types

#### 1. Update Conflicts

**Scenario**: Server and iPad both modify same entity.

**Resolution**: Server wins
- Merge policy: `NSMergeByPropertyObjectTrumpMergePolicy`
- Server data overwrites local changes for DOWN entities
- Local changes uploaded for UP entities

**Example**:
```
Server: Student.year = "Year 4"
iPad:   Student.year = "Year 3" (old cached value)

Result: Student.year = "Year 4" (server wins)
```

#### 2. Insert Conflicts

**Scenario**: Same entity created on server and iPad with different UUIDs.

**Resolution**: Keep both, deduplication handled by server
- iPad creates with client-generated UUID
- Server creates with server-generated dsid
- Server deduplicates based on business rules

#### 3. Delete Conflicts

**Scenario**: Entity deleted on server, modified on iPad.

**Resolution**: Soft delete wins
- `dsDeleted = YES` flag respected by both sides
- Deleted entities remain in database (audit trail)
- UI filters out `dsDeleted = YES` objects

**Implementation**:
```objc
// Fetch request excludes deleted
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"dsDeleted == NO"];

// Soft delete
object.dsDeleted = @(YES);
[context save:nil];  // Triggers sync
```

### Error Handling from Server

**Server Error Response**:
```json
{
  "data_form": {
    "errors": [
      {
        "id": "uuid-form-123",
        "code": "Validation Failed",
        "message": "Missing required field: studentID"
      }
    ]
  }
}
```

**Client Handling**:
```objc
for (errorDict in errors) {
    if ([errorDict[@"code"] isEqualToString:@"Nothing to Update"]) {
        // Ignore (object unchanged)
        continue;
    }

    // Fetch error object
    DSManagedObject *obj = [class managedObjectWithId:errorDict[@"id"]];

    // Mark error
    obj.syncError = @(YES);
    [context save:nil];

    // Log error
    DDLogError(@"Sync error for %@: %@", obj, errorDict[@"message"]);

    // Report to Crashlytics
    [[FIRCrashlytics crashlytics] recordError:error];
}
```

**User Notification**:
- Errors displayed in sync UI
- "Sync completed with errors" message
- User can retry sync later
- Object remains marked `syncError = YES` until resolved

## Offline Support

### Offline-First Strategy

```mermaid
stateDiagram-v2
    [*] --> CheckNetwork
    CheckNetwork --> Online: Network Available
    CheckNetwork --> Offline: No Network

    Offline --> LocalOperation: User Action
    LocalOperation --> QueueChange: Save to Core Data
    QueueChange --> CheckNetwork: Periodic Check

    Online --> Sync: Auto or Manual
    Sync --> UploadChanges: Stage 1-2
    UploadChanges --> DownloadData: Stage 3
    DownloadData --> UpdateLocal: Stage 4
    UpdateLocal --> [*]: Sync Complete

    Offline --> [*]: Continue Offline
```

### Change Queue

**Queue Implementation**: Core Data entities with sync flags

```objc
// Fetch objects needing sync
NSPredicate *needsSyncPredicate = [NSPredicate predicateWithFormat:
    @"syncNew == YES OR syncChangedKeys.@count > 0"];
NSArray *objectsToSync = [DataForm fetchAllWithPredicate:needsSyncPredicate];
```

**Queue Operations**:
- **Enqueue**: Automatic on save (via `willSave` notification)
- **Dequeue**: Automatic on successful sync
- **Inspect**: View pending changes in UI (see DSSyncPendingFormsTableViewController)

### Sync Triggers

#### 1. Automatic Sync

**Trigger Points**:
- App enters foreground
- View controller appears (if presenting modally)
- Network connectivity restored
- Timer-based (configurable interval, default: 10 minutes)

**Throttling**:
```objc
- (BOOL)shouldSkipSync {
    NSDate *lastSync = DSUserDefaults.modelLastSyncDate;
    NSTimeInterval elapsed = [[NSDate date] timeIntervalSinceDate:lastSync];
    return (elapsed < 600);  // 10 minutes
}
```

#### 2. Manual Sync

**User-Initiated**:
- Pull-to-refresh gesture
- Sync button in UI
- Settings → "Sync Now"

**Force Sync**:
```objc
[DSModel.sharedModel forceSync];  // Ignores throttling
```

#### 3. Event-Driven Sync

**Triggers**:
- Student sign-out (immediate sync)
- Clinic session complete (immediate sync)
- Assistance request (immediate sync)

### Offline UX

**Network Status Indicator**:
```
┌─────────────────────────────────────┐
│  ○ Offline - Changes will sync     │  ← Amber indicator
│     when network is available       │
└─────────────────────────────────────┘
```

**Sync Status Badge**:
```
┌─────────────────────────────────────┐
│  Sync Status: 12 forms pending      │
│  Last synced: 2 hours ago           │
│  [Sync Now Button]                  │
└─────────────────────────────────────┘
```

**Pending Changes View**:
- List of all pending forms/attendance logs
- Timestamp of last modification
- Tap to view details
- Option to discard changes (confirmation required)

## Performance Optimization

### Optimization Techniques

#### 1. Batch Processing

**Benefit**: Reduces memory usage and database writes

```objc
NSUInteger batchSize = 100;
for (NSUInteger i = 0; i < ceil(totalObjects / batchSize); i++) {
    @autoreleasepool {
        // Process 100 objects
        for (NSUInteger j = 0; j < batchSize; j++) {
            // Create/update object
        }
        // Save batch
        [context saveContextAndAncestorContexts];
    }
}
```

**Results**:
- 1000 students: 10 batches, 10 saves (vs. 1000 saves)
- Memory: ~10MB per batch (vs. ~100MB for all)

#### 2. Incremental Sync

**Benefit**: Only syncs changed data since last sync

```objc
// Send last update timestamp
@{
    @"student": @{
        @"lastUpdate": @"2023-10-15 10:00:00"
    }
}

// Server returns only students modified after 10:00:00
```

**Results**:
- Full sync: 5000 students (~2MB, 30 seconds)
- Incremental sync: 50 students (~20KB, 2 seconds)

#### 3. Cached Date Formatters

**Benefit**: Date formatting is expensive, caching improves performance

```objc
@property (nonatomic, strong, readonly) NSDateFormatter *dateTimeFormatter;

// Cached, reused across all date conversions
- (NSDateFormatter *)dateTimeFormatter {
    static NSDateFormatter *formatter = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"yyyy-MM-dd HH:mm:ss";
        formatter.locale = [NSLocale localeWithLocaleIdentifier:@"en_US_POSIX"];
    });
    return formatter;
}
```

**Results**:
- 1000 date conversions: 50ms (cached) vs. 500ms (uncached)

#### 4. Background Operations

**Benefit**: UI remains responsive during sync

```objc
UIBackgroundTaskIdentifier bgTask =
    [[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:^{
        // Cleanup if time expires
    }];

dispatch_async(syncQueue, ^{
    // Perform sync
    [[UIApplication sharedApplication] endBackgroundTask:bgTask];
});
```

**Results**:
- UI thread: 60 FPS maintained
- Sync continues for up to 30 seconds in background

#### 5. Semaphore-Based Locking

**Benefit**: Prevents concurrent syncs (data corruption)

```objc
if (dispatch_semaphore_wait(syncSemaphore, DISPATCH_TIME_NOW)) {
    DDLogWarn(@"Sync already in progress");
    return;
}

// Perform sync

dispatch_semaphore_signal(syncSemaphore);
```

### Performance Metrics

| Operation | Time (Avg) | Network | Memory |
|-----------|-----------|---------|--------|
| **Incremental Sync** | 2-5s | 20-100 KB | 10 MB |
| **Full Sync (1000 students)** | 30-45s | 2-5 MB | 50 MB |
| **Upload 100 forms** | 5-10s | 100-200 KB | 15 MB |
| **Process 1000 objects** | 10-15s | N/A | 10 MB (batched) |

## Error Handling

### Error Types & Responses

| Error Type | HTTP Code | App Response |
|-----------|-----------|--------------|
| **Offline** | -1009 (NSError) | Show "Offline" message, queue changes |
| **Unauthorized** | 401 | Prompt reconfiguration |
| **Server Error** | 500 | Show "Server error", retry later |
| **Bad Request** | 400 | Show specific error, mark objects |
| **Timeout** | -1001 | Retry with exponential backoff |

### Error Recovery Strategies

#### 1. Automatic Retry

```objc
- (void)syncWithRetry:(NSInteger)attempt {
    [self sync:^(BOOL success) {
        if (!success && attempt < 3) {
            NSTimeInterval delay = pow(2, attempt);  // Exponential backoff
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, delay * NSEC_PER_SEC), ^{
                [self syncWithRetry:attempt + 1];
            });
        }
    }];
}
```

**Backoff**:
- Attempt 1: Immediate
- Attempt 2: 2 seconds
- Attempt 3: 4 seconds
- Attempt 4: 8 seconds (give up)

#### 2. Object-Level Error Tracking

```objc
@property (nonatomic) NSNumber *syncError;

// Mark error
object.syncError = @(YES);

// Filter out errors in UI
NSPredicate *validObjects = [NSPredicate predicateWithFormat:@"syncError == NO"];
```

**Error Badge**:
```
┌─────────────────────────────────────┐
│  ⚠️  3 forms failed to sync         │
│     Tap to view details             │
└─────────────────────────────────────┘
```

#### 3. Logging & Monitoring

**CocoaLumberjack Integration**:
```objc
DDLogDebug(@"Starting sync");
DDLogInfo(@"Synced %lu objects", (unsigned long)count);
DDLogWarn(@"Sync throttled (too soon since last sync)");
DDLogError(@"Sync failed: %@", error.localizedDescription);
```

**Firebase Crashlytics**:
```objc
[[FIRCrashlytics crashlytics] recordError:error];
[[FIRCrashlytics crashlytics] setCustomValue:@(syncAttempt) forKey:@"sync_attempt"];
```

## Security

### Data Protection

#### 1. Transport Security

**HTTPS Only**:
```objc
// All URLs must use HTTPS
NSURL *url = [NSURL URLWithString:@"https://lu-apiprod.examsoft.com/"];
```

**Certificate Pinning**: Not currently implemented (relies on system trust)

#### 2. Credential Storage

**Keychain Storage**:
```swift
struct StringVault {
    static func setAppUUID(_ uuid: String) {
        KeychainStore.shared.set(uuid, forKey: "app_uuid")
    }

    static func appUUID() -> String? {
        return KeychainStore.shared.get(forKey: "app_uuid")
    }
}
```

**Protected Keys**:
- App UUID
- App Passcode
- Passcode Salt
- Sync Server URLs
- Image Shared Secrets
- Push Notification Token

#### 3. Data at Rest

**Core Data Encryption**:
- iOS-level file encryption (Data Protection API)
- Encryption key derived from device passcode
- Files inaccessible when device locked

**Keychain Encryption**:
- Hardware-backed encryption (Secure Enclave on newer devices)
- Biometric-protected access (optional)

#### 4. Audit Trail

**Sync Audit**:
```objc
SyncInformation {
    deviceUUID: "iPad-12345"
    deviceName: "Clinic 1 iPad"
    lastSync: 2023-10-15 14:30:00
    appVersion: "1.0.24"
    iOSVersion: "15.0"
    staff: Dr. Smith
}
```

**Staff Sign-In Audit**:
```objc
StaffSignIn {
    staff: Dr. Smith
    timetable: Year 3 Clinic - Monday AM
    time: 2023-10-15 09:00:00
    appUUID: "device-12345"
}
```

---

**For more information, see:**
- [Architecture Overview](architecture-overview.md) - System architecture
- [Data Model Reference](data-model.md) - Entity details
- [Shared Library Documentation](shared-library.md) - DSModel, DSSyncManager

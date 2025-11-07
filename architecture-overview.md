# Architecture Overview

## Table of Contents
1. [System Overview](#system-overview)
2. [Application Architecture](#application-architecture)
3. [Technology Stack](#technology-stack)
4. [Core Components](#core-components)
5. [Data Flow](#data-flow)
6. [Deployment Architecture](#deployment-architecture)

## System Overview

The LIFTUPP platform is an enterprise iPad solution for conducting OSCE (Objective Structured Clinical Examination) assessments in medical and dental education. The system enables real-time clinical assessment, student tracking, and comprehensive feedback management.

### High-Level Architecture

```mermaid
graph TB
    subgraph "Presentation Layer"
        A[iPad UI - UIKit]
        B[View Controllers]
        C[Custom UI Components]
    end

    subgraph "Business Logic Layer"
        D[Assessment Manager]
        E[Attendance Manager]
        F[Form Generator]
        G[Timer Manager]
    end

    subgraph "Data Layer"
        H[Core Data Stack]
        I[Sync Manager]
        J[Settings Manager]
    end

    subgraph "Infrastructure Layer"
        K[Network Manager - AFNetworking]
        L[Keychain Storage]
        M[Image Cache]
        N[Logging - CocoaLumberjack]
    end

    subgraph "External Systems"
        O[Web Portal API]
        P[Institution Servers]
    end

    A --> B
    B --> C
    B --> D
    D --> E
    D --> F
    D --> G
    E --> H
    F --> H
    G --> H
    H --> I
    I --> K
    J --> L
    K --> O
    O --> P

    style A fill:#3FA9F5
    style H fill:#90EE90
    style K fill:#FFD700
    style O fill:#FF6B6B
    
Application Architecture
------------------------

### Three-Tier Architecture

#### 1\. Presentation Tier

*   **Master-Detail Pattern** - iPad-optimized navigation
    
*   **Custom UI Components** - Reusable views (DSView, DSRoundedButton, DSTabView)
    
*   **Storyboard-based UI** - MainStoryBoard.storyboard defines UI structure
    
*   **Category Extensions** - 60+ UIKit extensions for enhanced functionality
    

#### 2\. Business Logic Tier

*   **DSModel** - Singleton coordinator for Core Data and sync operations
    
*   **DSSyncManager** - Handles bidirectional synchronization
    
*   **DSSettingsManager** - Multi-institution configuration
    
*   **DSHTTPSessionManager** - Network request abstraction
    
*   **Timer System** - DSCountDownTimer, DSStationTimer, DSStopwatch
    

#### 3\. Data Tier

*   **Core Data** - 38 entities modeling clinical assessments
    
*   **Keychain** - Secure credential storage via StringVault (Swift)
    
*   **File System** - Image caching and institution logos
    
*   **NSUserDefaults** - User preferences and app state
    

### Core Data Architecture
graph LR
    subgraph "Context Hierarchy"
        A[Main Queue Context]
        B[Private Sync Context]
        C[Saving Context]
        D[SQLite Persistent Store]
    end

    A -->|Parent| B
    B -->|Parent| C
    C -->|Writes to| D

    style A fill:#3FA9F5
    style B fill:#90EE90
    style C fill:#FFD700
    style D fill:#FF6B6B

**Context Responsibilities:**

*   **Main Queue Context**: UI operations, immediate user interactions
    
*   **Private Sync Context**: Background synchronization, data processing
    
*   **Saving Context**: Final persistence to SQLite store
    

### Module Architecture
graph TB
    subgraph "OSCE App Module"
        A[App Delegate]
        B[Root View Controller]
        C[Clinic View Controllers]
        D[Marking Matrix]
        E[Timers]
        F[Supporting Files]
    end

    subgraph "Shared Library Module - liftupp-app-library"
        G[Data Model]
        H[View Controllers]
        I[Categories - 60+ Extensions]
        J[Custom Views]
        K[Network Layer]
        L[App Configuration]
    end

    subgraph "Model Module"
        M[Core Data Model]
        N[Entity Classes - 38 entities]
        O[KeychainStore - Swift]
        P[StringVault - Swift]
    end

    A --> B
    B --> C
    C --> D
    C --> E
    A --> G
    C --> H
    G --> M
    G --> K
    H --> J
    G --> O

    style A fill:#3FA9F5
    style G fill:#90EE90
    style M fill:#FFD700
    
Technology Stack
----------------

### Development Environment

*   **Xcode**: iOS development IDE
    
*   **Language Mix**:
    
    *   Objective-C (88+ files, primary language)
        
    *   Swift (KeychainStore, StringVault for secure storage)
        
*   **Dependency Management**: CocoaPods
    
*   **Version Control**: Git (Bitbucket/GitHub)
    

### Frameworks & Libraries

#### Third-Party Dependencies

| Library | Version | Purpose | |---------|---------|---------| | AFNetworking | 4.0.1 | HTTP networking and API communication | | CocoaLumberjack | 3.8.1 | Logging framework with multiple log levels | | TPKeyboardAvoiding | 1.3.5 | Keyboard management for forms | | MBProgressHUD | Embedded | Progress indicators during sync |

#### iOS Frameworks

*   **UIKit** - User interface
    
*   **Core Data** - Object graph and persistence
    
*   **Foundation** - Core utilities
    
*   **Security** - Keychain services
    
*   **CFNetwork** - Low-level networking
    
*   **Firebase Crashlytics** - Crash reporting
    

### Platform Requirements

*   **Minimum iOS**: 12.0
    
*   **Target Device**: iPad only
    
*   **Orientation**: Landscape only
    
*   **Bundle ID**: com.liftupp.osce
    

Core Components
---------------

### 1\. Data Persistence Layer

graph TB
    subgraph "Persistence Components"
        A[DSModel Singleton]
        B[DSManagedObject Base Class]
        C[DSManagedObjectContext Extended Context]
        D[Entity Metadata System]
    end

    subgraph "38 Core Data Entities"
        E[Timetable / AttendanceLog]
        F[TypeForm / DataForm]
        G[Student / Staff / User]
        H[ActivityType / ClinicGroup]
        I[Comments / Feedback]
    end

    subgraph "Storage Backends"
        J[SQLite Database]
        K[iOS Keychain]
        L[NSUserDefaults]
        M[File System Cache]
    end

    A --> B
    B --> C
    C --> D
    D --> E
    D --> F
    D --> G
    D --> H
    D --> I
    E --> J
    F --> J
    G --> J
    H --> J
    I --> J
    A --> K
    A --> L
    A --> M

    style A fill:#3FA9F5
    style J fill:#FF6B6B

**DSModel** - Core Data Manager

*   Three-context architecture for thread safety
    
*   Automatic saving and merge policies
    
*   Cached date formatters for performance
    
*   Background task support for extended operations
    

**DSManagedObject** - Base Entity

*   CRUD operations: +insert, +fetchAll, +managedObjectWithId:
    
*   Automatic type conversion (String ↔ NSDate, NSNumber)
    
*   Dictionary-based object creation from JSON
    
*   Sync change tracking via syncChangedKeys
    

**Entity Metadata System**

*   Custom userInfo keys for sync configuration
    
*   uk.ac.liv.liftupp.coredata.sync - Sync enabled flag
    
*   uk.ac.liv.liftupp.coredata.syncpoint - Server endpoint identifier
    
*   Computed properties: shouldSyncUp, shouldSyncDown, primaryKeyName
    

### 2\. Synchronization Layer

sequenceDiagram
    participant App as iPad App
    participant SM as DSSyncManager
    participant DM as DSModel
    participant HTTP as HTTPSessionManager
    participant API as Web Portal API

    App->>SM: Trigger Sync
    SM->>DM: Prepare Upload Data
    DM->>DM: Collect Changed Objects
    DM->>DM: Build JSON Payload
    DM-->>SM: Upload Dictionary
    SM->>HTTP: POST /syncClient
    HTTP->>API: JSON Request
    API-->>HTTP: JSON Response
    HTTP-->>SM: Server Data
    SM->>DM: Process Sync Data
    DM->>DM: Parse Response
    DM->>DM: Update Core Data (batch 100)
    DM->>DM: Mark Objects as Synced
    DM-->>SM: Sync Complete
    SM-->>App: Success Callback

**Sync Stages:**

1.  **Preparing Upload Data** - Collect modified objects, build JSON
    
2.  **Transferring Upload Data** - POST to server via AFNetworking
    
3.  **Processing Sync Data** - Parse response, update Core Data
    
4.  **Cleaning Old Data** - Mark objects as synced, update timestamps
    

**Conflict Resolution:**

*   Merge policy: NSMergeByPropertyObjectTrumpMergePolicy
    
*   Server data wins for bidirectional entities
    
*   Error tracking via syncError flag
    
*   Failed syncs retry on next sync attempt
    

### 3\. Network Layer
graph TB
    subgraph "Network Components"
        A[DSHTTPSessionManager]
        B[AFNetworking]
        C[Reachability Monitor]
    end

    subgraph "API Endpoints"
        D[/syncClient - Main Sync]
        E[/appSetup - Configuration]
        F[/getStationAttendances - GET]
        G[/toggleAssistanceRequired - GET]
    end

    subgraph "Request/Response"
        H[JSON Serialization]
        I[Basic HTTP Auth]
        J[Error Handling]
    end

    subgraph "Offline Support"
        K[Change Queue]
        L[Last Sync Tracking]
        M[Sync Skip Logic]
    end

    A --> B
    A --> C
    A --> D
    A --> E
    A --> F
    A --> G
    B --> H
    B --> I
    B --> J
    A --> K
    K --> L
    L --> M

    style A fill:#3FA9F5
    style B fill:#90EE90
    style D fill:#FFD700
    
**Authentication:**

*   Basic HTTP Auth with hardcoded credentials
    
*   Username: liftupp\_ipad
    
*   Password: iPad\_LiftUpp\_Pass
    
*   App UUID + Passcode in JSON payload for device identification
    

**Offline Support:**

*   Reachability monitoring (www.google.com)
    
*   Queued changes stored in Core Data
    
*   Automatic sync on network restoration
    
*   Sync skip if synced within 10 minutes
    

### 4\. Configuration & Multi-Tenancy
graph TB
    subgraph "Configuration System"
        A[DSSettingsManager Singleton]
        B[QR Code Scanner]
        C[Institution Profiles]
    end

    subgraph "Institution Data"
        D[Code: liverpool, manchester, etc.]
        E[Name & Type]
        F[Brand Colors - mainColor, secondaryColor]
        G[Logo & Images]
        H[API URLs]
    end

    subgraph "Storage"
        I[Keychain - Sensitive]
        J[UserDefaults - Non-Sensitive]
        K[File System - Images]
    end

    subgraph "13+ Supported Institutions"
        L[Liverpool]
        M[Manchester]
        N[QMUL]
        O[King's College]
        P[Belfast]
        Q[Scotland: Aberdeen, Dundee, Glasgow]
        R[Cardiff, Portsmouth, Leicester]
    end

    A --> B
    B --> C
    C --> D
    C --> E
    C --> F
    C --> G
    C --> H
    D --> I
    E --> J
    F --> J
    G --> K
    H --> I
    A --> L
    A --> M
    A --> N
    A --> O
    A --> P
    A --> Q
    A --> R

    style A fill:#3FA9F5
    style I fill:#FF6B6B
    
**Configuration Flow:**

1.  User scans QR code at institution
    
2.  App fetches configuration JSON from URL
    
3.  DSSettingsManager.configureAppWithJSON() processes config
    
4.  Institution code, colors, logos, API URLs stored
    
5.  App UI refreshed with institution branding
    
6.  Device locked to institution (cannot reconfigure without reset)
    

Data Flow
---------

### Assessment Creation Flow
sequenceDiagram
    participant User as Staff User
    participant UI as View Controller
    participant TT as Timetable
    participant AL as AttendanceLog
    participant TF as TypeForm
    participant DF as DataForm

    User->>UI: Select Timetable
    UI->>TT: Load Timetable
    TT->>AL: Get AttendanceLogs
    AL->>TF: Get Available Forms
    TF->>DF: generateFormWithAttendanceLog()
    DF->>DF: Create DataSections
    DF->>DF: Create DataQuestions
    DF-->>AL: Assign DataForm
    AL-->>UI: Display Assessment Form
    User->>UI: Enter Ratings/Feedback
    UI->>DF: Update DataQuestions
    DF->>DF: Mark syncChangedKeys
    User->>UI: Submit Form
    UI->>DF: Save to Core Data
    DF->>DM: Context Save
    DM->>Sync: Trigger Sync
    Sync->>API: Upload DataForm
    
### Student Rotation Flow
stateDiagram-v2
    [*] --> CheckIn: Staff Opens Clinic
    CheckIn --> AttendanceMarked: Mark Present/Absent
    AttendanceMarked --> Station1: Assign to Station
    Station1 --> Assessment1: Staff Assesses
    Assessment1 --> Timer1: Station Timer Running
    Timer1 --> Rotation: Timer Expires
    Rotation --> Station2: Move to Next Station
    Station2 --> Assessment2: Staff Assesses
    Assessment2 --> SignOut: Complete All Stations
    SignOut --> Feedback: Provide Feedback
    Feedback --> [*]: Session Complete
    
### Deployment Architecture
### iPad Deployment
graph TB
    subgraph "Institution Infrastructure"
        A[Institution Network]
        B[WiFi Access Points]
    end

    subgraph "iPad Fleet"
        C[iPad Device 1]
        D[iPad Device 2]
        E[iPad Device N]
    end

    subgraph "Backend Systems"
        F[Load Balancer]
        G[API Server 1]
        H[API Server 2]
        I[Database Server]
    end

    subgraph "Configuration"
        J[QR Code Config]
        K[Institution Profile]
    end

    A --> B
    B --> C
    B --> D
    B --> E
    C --> F
    D --> F
    E --> F
    F --> G
    F --> H
    G --> I
    H --> I
    J --> K
    K --> C
    K --> D
    K --> E

    style C fill:#3FA9F5
    style D fill:#3FA9F5
    style E fill:#3FA9F5
    style I fill:#FF6B6B

### Server Endpoints

**Production Servers:**

*   https://lu-apiprod.examsoft.com/ - Main production API
    
*   https://liftupp.codehaus.es/ - Examsoft live server
    
*   https://liftupp-eu.qa-examsoft.com/ - QA/testing server
    

**Local Development:**

*   http://nick.local.liftupp.com/ - Nick's dev environment
    
*   http://adam.local.liftupp.com/ - Adam's dev environment
    

### Build Configurations

| Configuration | Server URL | Purpose | |--------------|------------|---------| | **Live** | lu-apiprod.examsoft.com | Production deployment | | **Debug Live** | lu-apiprod.examsoft.com | Production with debug logging | | **Examsoft Live** | liftupp.codehaus.es | Examsoft testing | | **Debug** | liftupp-eu.qa-examsoft.com | Development/QA | | **Debug MobDev** | liftupp-eu.qa-examsoft.com | Mobile dev team | | **Debug Nick** | nick.local.liftupp.com | Developer local | | **Debug Dan** | liftupp-eu.qa-examsoft.com | Developer testing | | **Debug Adam** | adam.local.liftupp.com | Developer local |

Design Patterns & Best Practices
--------------------------------

### Architectural Patterns Used

1.  **Singleton Pattern**
    
    *   DSModel, DSSettingsManager, DSSyncManager, DSHTTPSessionManager
        
    *   Thread-safe initialization with dispatch\_once
        
2.  **Repository Pattern**
    
    *   DSManagedObject provides static methods for entity access
        
    *   Abstracts Core Data fetch requests
        
3.  **Observer Pattern**
    
    *   Core Data context change notifications
        
    *   Automatic sync change tracking
        
4.  **Delegation Pattern**
    
    *   View controller callbacks for sync state changes
        
    *   Custom delegates for UI events
        
5.  **Category Pattern (Objective-C)**
    
    *   60+ category files extending Foundation/UIKit
        
    *   Code organization and reusability
        
6.  **Master-Detail Pattern**
    
    *   iPad-optimized navigation
        
    *   List of students → Individual assessment detail
        

### Thread Safety Strategy
graph LR
    A[Main Thread] -->|performBlockAndWait| B[Main Context]
    C[Background Thread] -->|performBlockAndWait| D[Sync Context]
    B -->|Parent-Child| D
    D -->|Parent-Child| E[Saving Context]
    E -->|Writes| F[Persistent Store]

    style A fill:#3FA9F5
    style C fill:#90EE90
    style F fill:#FF6B6B
    
**Rules:**

*   UI operations: Always on Main Queue Context
    
*   Sync operations: Always on Private Sync Context
    
*   Context access: Always use performBlockAndWait: or performBlock:
    
*   Never pass NSManagedObject between threads directly
    
*   Use objectID for cross-context references
    

### Error Handling

**Network Errors:**

*   Error code -1009: Offline (special handling)
    
*   Other errors: Log to Crashlytics, show alert
    
*   Retry logic: Background task with semaphore
    

**Sync Errors:**

*   Server returns errors array
    
*   Objects marked with syncError = YES
    
*   Error displayed to user, retry on next sync
    

**Core Data Errors:**

*   Logged via CocoaLumberjack
    
*   Recorded to Firebase Crashlytics
    
*   Merge conflicts: Server wins
    

Performance Optimizations
-------------------------

1.  **Batch Processing** - Sync processes 100 objects per batch
    
2.  **Cached Date Formatters** - Reusable NSDateFormatter instances
    
3.  **Autorelease Pools** - Memory management during sync
    
4.  **Background Tasks** - Extended execution time for sync
    
5.  **Image Caching** - Disk-based cache for student/staff photos
    
6.  **Fetch Request Optimization** - Proper predicates and sort descriptors
    

Security Architecture
---------------------

### Security Layers
graph TB
    subgraph "Application Security"
        A[Keychain Storage]
        B[App UUID + Passcode]
        C[Basic HTTP Auth]
    end

    subgraph "Data Security"
        D[Core Data Encryption - iOS]
        E[Keychain Encryption]
        F[HTTPS Transport]
    end

    subgraph "Access Control"
        G[Staff Authentication]
        H[QR Code Configuration Lock]
        I[Session Management]
    end

    A --> B
    B --> C
    C --> F
    E --> A
    D --> E
    G --> H
    H --> I

    style A fill:#FF6B6B
    style F fill:#90EE90
    
**Sensitive Data Storage:**

*   App UUID/Passcode → Keychain (via StringVault)
    
*   API URLs → Keychain
    
*   Staff credentials → Keychain
    
*   Push tokens → Keychain
    
*   Image secrets → Keychain
    

**Data Protection:**

*   iOS file encryption (Data Protection API)
    
*   Core Data backed by encrypted SQLite
    
*   HTTPS for all network communication
    

**Next Steps:**

*   Review [Data Model Reference](data-model.md) for entity details
    
*   Study [Sync Architecture](sync-architecture.md) for synchronization specifics
    
*   Explore [Shared Library Documentation](shared-library.md) for code reuse

# LIFTUPP iPad Apps - Complete Documentation Suite

This repository contains comprehensive documentation for the LIFTUPP (Liverpool iPad For Teaching University Professional Practice) platform, which consists of two related iPad applications for clinical assessment in dental and medical education.

## Repository Overview

- **[osce-ios](https://github.com/DanSmithTII/osce-ios)** - OSCE assessment app for staff/assessors
- **[liftupp-for-ipad](https://github.com/DanSmithTII/liftupp-for-ipad)** - Complementary teaching app
- Both apps share the **liftupp-app-library** common codebase

## Documentation Structure

### For Product Owners & Stakeholders
- **[Product Owner Guide](product-owner-guide.md)** - Features, user journeys, and value proposition

### For Technical Teams
- **[Architecture Overview](architecture-overview.md)** - High-level system architecture with diagrams
- **[Data Model Reference](data-model.md)** - Complete Core Data model documentation
- **[Sync Architecture](sync-architecture.md)** - Synchronization and web portal integration
- **[Shared Library Documentation](shared-library.md)** - liftupp-app-library detailed reference
- **[Technical Guide](technical-guide.md)** - Development setup and guidelines

## Quick Start

### For Product Owners
Start with the [Product Owner Guide](product-owner-guide.md) to understand:
- What the apps do and why they exist
- Key features and workflows
- User personas and journeys
- Business value and benefits

### For Developers
Begin with the [Architecture Overview](architecture-overview.md), then:
1. Review the [Data Model Reference](data-model.md) to understand data structures
2. Study the [Sync Architecture](sync-architecture.md) for backend integration
3. Explore the [Shared Library Documentation](shared-library.md) for reusable components
4. Consult the [Technical Guide](technical-guide.md) for development practices

## Technology Stack

- **Languages**: Objective-C (primary), Swift (security/modern features)
- **Platform**: iOS 12.0+, iPad only, landscape orientation
- **Persistence**: Core Data (SQLite-backed)
- **Networking**: AFNetworking 4.0
- **Logging**: CocoaLumberjack 3.8
- **Analytics**: Firebase Crashlytics
- **Security**: iOS Keychain for sensitive data

## System Architecture

```mermaid
graph TB
    subgraph "iPad Apps"
        A[OSCE iOS App]
        B[LiftUpp for iPad]
    end

    subgraph "Shared Code"
        C[liftupp-app-library]
    end

    subgraph "Backend"
        D[Web Portal API]
        E[Institution Servers]
    end

    A --> C
    B --> C
    C --> D
    D --> E

    style A fill:#3FA9F5
    style B fill:#3FA9F5
    style C fill:#90EE90
    style D fill:#FFD700
    
Key Features
------------

### Clinical Assessment

*   **Station-based assessments** - Multiple assessment stations in circuits
    
*   **Form-based evaluation** - Customizable assessment forms and marking matrices
    
*   **Real-time attendance** - Track student attendance and rotations
    
*   **Timer management** - Countdown timers for station durations
    

### Data Management

*   **Offline-first design** - Full functionality without internet
    
*   **Bidirectional sync** - Automatic synchronization with web portal
    
*   **Multi-institution support** - 13+ UK institutions configured
    
*   **Secure storage** - Keychain-based credential management
    

### User Experience

*   **Master-detail interface** - iPad-optimized workflows
    
*   **Assistance requests** - Real-time help system during assessments
    
*   **Comments and feedback** - Rich annotation system
    
*   **Institution branding** - Customizable colors and logos
    

Project Status
--------------

*   **Current Version**: 1.0.24 (osce-ios)
    
*   **Active Institutions**: Liverpool, Manchester, QMUL, King's, Belfast, Scotland (Aberdeen, Dundee, Glasgow), Cardiff, Portsmouth, Leicester
    
*   **Platform**: Production use across UK dental and medical schools
    
*   **Maintenance**: Active development and support
    

Contributing
------------

This documentation is maintained to support ongoing development and institutional deployment. For updates or corrections, please submit pull requests.

License
-------

© The University of Liverpool. All rights reserved.    

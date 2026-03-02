# Telemetry Plugin Architecture

## Overview

The Telemetry plugin is a Thunder (WPEFramework) plugin that provides telemetry data collection, reporting, and management capabilities for RDK devices. It enables the collection of device metrics, application events, and system telemetry data, which can be uploaded to backend servers for analysis and monitoring.

## System Architecture

### Component Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    Thunder Framework                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Telemetry Plugin                         │  │
│  │  ┌─────────────────┐  ┌─────────────────────────┐   │  │
│  │  │   Telemetry     │  │  TelemetryImplementation │   │  │
│  │  │  (JSON-RPC)     │  │   (COM-RPC Interface)    │   │  │
│  │  └────────┬────────┘  └──────────┬──────────────┘   │  │
│  │           │                       │                   │  │
│  │           └───────────┬───────────┘                   │  │
│  │                       │                               │  │
│  │           ┌───────────▼───────────────┐              │  │
│  │           │   Event Handling & Data   │              │  │
│  │           │   Collection Management   │              │  │
│  │           └───────────┬───────────────┘              │  │
│  └───────────────────────┼───────────────────────────────┘  │
└────────────────────────┼─────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
    │  RFC    │    │  RBUS   │    │  T2     │
    │  APIs   │    │  Events │    │ Service │
    └─────────┘    └─────────┘    └─────────┘
```

## Core Components

### 1. Telemetry Plugin Interface (`Telemetry.cpp/h`)
- Implements Thunder's `PluginHost::IPlugin` and `PluginHost::JSONRPC` interfaces
- Handles plugin lifecycle (Initialize, Deinitialize, activation states)
- Exposes JSON-RPC methods for external communication
- Manages plugin configuration and state transitions

### 2. Telemetry Implementation (`TelemetryImplementation.cpp/h`)
- Implements `Exchange::ITelemetry` and `Exchange::IConfiguration` interfaces
- Core business logic for telemetry operations
- Manages telemetry report profiles and their lifecycle
- Handles event logging and data collection
- Integrates with system services (PowerManager, UserSettings)

### 3. External Dependencies Integration

#### RFC (Remote Feature Control)
- Retrieves telemetry configuration parameters
- Manages report profile definitions via RFC values
- Key RFC parameters:
  - `Device.X_RDKCENTRAL-COM_T2.ReportProfiles`
  - `Device.DeviceInfo.X_RDKCENTRAL-COM_RFC.Feature.Telemetry.FTUEReport.Enable`

#### RBUS (RDK Message Bus)
- Enables real-time communication with T2 service
- Publishes telemetry upload requests
- Subscribes to privacy mode changes
- Key RBUS elements:
  - `Device.X_RDKCENTRAL-COM_T2.UploadDCMReport` (trigger uploads)
  - `Device.X_RDKCENTRAL-COM_T2.AbortDCMReport` (abort uploads)
  - `Device.X_RDKCENTRAL-COM_Privacy.PrivacyMode` (privacy events)

## Data Flow

### Report Profile Management
1. Plugin initialization loads default profiles from `/etc/t2profiles/default.json`
2. RFC configuration is queried for additional report profiles
3. Profile data is cached in persistent storage at `/opt/.t2reportprofiles/`
4. Profile status changes (STARTED/COMPLETE) are tracked and persisted

### Event Logging Flow
1. External applications call `logApplicationEvent` JSON-RPC method
2. Plugin validates event parameters (name, value, max 512 bytes)
3. Event data is sent to T2 telemetry service via RBUS
4. Telemetry service processes and aggregates events for reporting

### Report Upload Flow
1. Client initiates upload via `uploadReport` JSON-RPC method or scheduled trigger
2. Plugin checks telemetry opt-out status (`/opt/tmtryoptout`)
3. RBUS message sent to T2 service with upload request
4. T2 service collects metrics and uploads to backend servers

## Inter-Plugin Communication

The Telemetry plugin integrates with other Thunder plugins using COM-RPC:

### PowerManager Integration
- Registers for power state change notifications
- Implements `IPowerManager::IModeChangedNotification` interface
- Tracks device power transitions for telemetry context

### UserSettings Integration (Optional)
- Queries privacy mode settings when RBUS is enabled
- Registers for privacy mode change notifications
- Implements `IUserSettings::INotification` interface
- Respects user privacy preferences for telemetry collection

## Threading and Synchronization

- Uses `Core::CriticalSection` (`_adminLock`) for thread-safe operations
- Notification callbacks are protected by mutex locks
- Multiple notification handlers can be registered concurrently
- Asynchronous event processing to prevent blocking Thunder main thread

## Configuration Management

### Static Configuration
- Plugin configuration file: `Telemetry.config`
- Compile-time settings: persistent folder path, default profiles location
- CMake options: `PLUGIN_T2_PERSISTENT_FOLDER`, `PLUGIN_DEFAULT_PROFILES_FILE`

### Runtime Configuration
- RFC-based dynamic configuration
- Report profile definitions loaded at runtime
- Profile status persisted across reboots

## Security Considerations

- Security token validation (can be disabled via `DISABLE_SECURITY_TOKEN` flag)
- Opt-out mechanism for user privacy (`/opt/tmtryoptout`)
- Privacy mode integration with UserSettings
- Input validation for all JSON-RPC parameters (max length checks)

## Error Handling

- COM-RPC interface acquisitions include null checks
- RBUS initialization failures are logged and handled gracefully
- JSON-RPC responses include appropriate error codes
- File I/O operations include error checking for profile persistence

## Performance Characteristics

- Lightweight JSON-RPC interface for minimal overhead
- Asynchronous event processing prevents blocking
- Profile data caching reduces filesystem I/O
- RBUS message bus provides low-latency communication with T2 service
- Out-of-process (OOP) execution option available for isolation and stability

## Extensibility

The plugin architecture supports:
- Multiple concurrent notification handlers
- Custom report profile definitions via RFC
- Extension points for additional telemetry data sources
- Plugin interface versioning (currently v1.2.2)

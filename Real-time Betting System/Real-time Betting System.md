# Real-time Betting System

<cite>
**Referenced Files in This Document**
- [server.js](file://server/server.js)
- [index.js](file://server/index.js)
- [socketHandler.js](file://server/socket/socketHandler.js)
- [betController.js](file://server/controllers/bet/betController.js)
- [betRoute.js](file://server/routes/bet/betRoute.js)
- [betModel.js](file://server/models/betModel.js)
- [main.jsx](file://client/src/main.jsx)
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx)
- [LiveBettingInterface.jsx](file://client/src/components/Bet/LiveBettingInterface.jsx)
- [LiveStream.jsx](file://client/src/components/Bet/LiveStream.jsx)
- [index.js](file://client/src/store/user/match-and-bet-slice/index.js)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Conclusion](#conclusion)

## Introduction
This document provides comprehensive documentation for the real-time betting system, focusing on the Socket.IO implementation for live match updates, odds changes, and bet notifications. It explains the room-based architecture for match-specific communications, the event broadcasting mechanism, and the live betting interface components including odds display, bet placement forms, and real-time statistics. The document also covers WebSocket connection handling, reconnection logic, error recovery strategies, data synchronization between frontend and backend, state consistency, conflict resolution, live streaming integration, and performance optimization techniques for handling multiple concurrent connections and high-frequency updates.

## Project Structure
The system follows a clear separation of concerns:
- Backend: Express server with Socket.IO for real-time communication, MongoDB for persistence, and REST APIs for CRUD operations.
- Frontend: React application with Redux Toolkit for state management, Socket.IO client for real-time updates, and integrated live streaming.

```mermaid
graph TB
subgraph "Client"
A[React App]
B[Redux Store]
C[SocketContext]
D[LiveBettingPage]
E[LiveBettingInterface]
F[LiveStream]
end
subgraph "Server"
G[Express Server]
H[Socket.IO Server]
I[Bet Controller]
J[Match Routes]
K[Socket Handler]
L[MongoDB]
end
A --> |HTTP| G
A --> |Socket.IO| H
G --> |REST| I
I --> |MongoDB| L
H --> |Rooms| K
K --> |Broadcast| A
D --> |Socket Events| C
E --> |Socket Events| C
F --> |Video Streaming| A
```

**Diagram sources**
- [server.js](file://server/server.js#L1-L92)
- [index.js](file://server/index.js#L1-L150)
- [socketHandler.js](file://server/socket/socketHandler.js#L1-L101)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L1-L943)
- [LiveBettingInterface.jsx](file://client/src/components/Bet/LiveBettingInterface.jsx#L1-L439)
- [LiveStream.jsx](file://client/src/components/Bet/LiveStream.jsx#L1-L462)

**Section sources**
- [server.js](file://server/server.js#L1-L92)
- [index.js](file://server/index.js#L1-L150)
- [main.jsx](file://client/src/main.jsx#L1-L20)

## Core Components
The real-time betting system consists of several core components:

### Backend Components
- **Socket Handler**: Manages Socket.IO connections, rooms, and event broadcasting
- **Bet Controller**: Handles bet placement, validation, and real-time notifications
- **REST Routes**: Expose endpoints for match data, bet history, and user operations
- **Database Models**: Define schemas for matches, bets, and user data

### Frontend Components
- **Socket Context**: Provides global Socket.IO connection with reconnection logic
- **Live Betting Page**: Orchestrates match data fetching, room management, and event listeners
- **Live Betting Interface**: Displays real-time odds, handles bet placement, and shows live statistics
- **Live Stream**: Integrates various video platforms for match viewing

**Section sources**
- [socketHandler.js](file://server/socket/socketHandler.js#L1-L101)
- [betController.js](file://server/controllers/bet/betController.js#L1-L125)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L1-L943)
- [LiveBettingInterface.jsx](file://client/src/components/Bet/LiveBettingInterface.jsx#L1-L439)
- [LiveStream.jsx](file://client/src/components/Bet/LiveStream.jsx#L1-L462)

## Architecture Overview
The system implements a room-based architecture for efficient real-time communication:

```mermaid
sequenceDiagram
participant Client as "Client App"
participant Socket as "SocketContext"
participant Server as "Socket Handler"
participant Controller as "Bet Controller"
participant DB as "MongoDB"
Client->>Socket : Connect to Socket.IO
Socket->>Server : Establish WebSocket connection
Server->>Server : Initialize rooms
Client->>Server : join-match(matchId)
Server->>Server : Join match room
Client->>Server : join-event(eventId)
Server->>Server : Join event room
Client->>Controller : placeBet(data)
Controller->>DB : Save bet record
Controller->>Server : Emit bet-placed
Server->>Server : Broadcast to match room
Server-->>Client : new-bet event
Client->>Server : match-update
Server->>Server : Broadcast match status
Server-->>Client : match-update event
```

**Diagram sources**
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx#L1-L62)
- [socketHandler.js](file://server/socket/socketHandler.js#L1-L101)
- [betController.js](file://server/controllers/bet/betController.js#L43-L106)

The architecture supports:
- **Match-specific rooms**: Isolated communication per match
- **Event rooms**: Broadcasting to all matches within an event
- **Admin rooms**: Specialized notifications for administrators
- **User-specific rooms**: Personal bet history and notifications

## Detailed Component Analysis

### Socket.IO Implementation
The backend Socket.IO implementation manages multiple room types and handles real-time events efficiently.

#### Room Management
The system creates specialized rooms for different communication scopes:

```mermaid
graph LR
A["Match Rooms"] --> A1["match-{matchId}"]
B["Event Rooms"] --> B1["event-{eventId}"]
C["Admin Room"] --> C1["admin_room"]
D["User Rooms"] --> D1["user-{userId}"]
A1 --> E["Match-specific Updates"]
B1 --> F["Event-wide Notifications"]
C1 --> G["Admin Operations"]
D1 --> H["Personal Bet History"]
```

**Diagram sources**
- [socketHandler.js](file://server/socket/socketHandler.js#L9-L56)

#### Event Broadcasting Mechanism
The system implements targeted broadcasting to minimize unnecessary network traffic:

```mermaid
flowchart TD
A["New Bet Placed"] --> B{"Get IO Instance"}
B --> C{"IO Available?"}
C -->|Yes| D["Emit to match-{matchId}"]
C -->|Yes| E["Emit to admin-room"]
C -->|No| F["Log Error"]
D --> G["Update Local State"]
E --> H["Admin Notification"]
G --> I["Client Receives new-bet"]
H --> I
```

**Diagram sources**
- [betController.js](file://server/controllers/bet/betController.js#L79-L96)
- [socketHandler.js](file://server/socket/socketHandler.js#L58-L72)

**Section sources**
- [socketHandler.js](file://server/socket/socketHandler.js#L1-L101)
- [betController.js](file://server/controllers/bet/betController.js#L1-L125)

### WebSocket Connection Handling
The frontend implements robust connection management with automatic reconnection:

#### Connection Lifecycle
```mermaid
stateDiagram-v2
[*] --> Connecting
Connecting --> Connected : connect event
Connecting --> Disconnected : connect_error
Connected --> Reconnecting : disconnect
Reconnecting --> Connected : reconnect
Reconnecting --> Disconnected : reconnect_failed
Connected --> Disconnected : disconnect
Disconnected --> Connecting : reconnect
```

**Diagram sources**
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx#L18-L54)

#### Reconnection Logic
The system implements exponential backoff with configurable attempts:
- Maximum 5 reconnection attempts
- Base delay of 1 second with exponential increase
- Maximum delay cap of 5 seconds
- Supports both WebSocket and polling transports

**Section sources**
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx#L1-L62)

### Live Betting Interface Components
The betting interface provides comprehensive real-time functionality:

#### Real-time Statistics
The interface maintains live betting statistics with incremental updates:

```mermaid
flowchart TD
A[Initial Load] --> B[Fetch Match Bets]
B --> C[Calculate Initial Stats]
C --> D[Render UI]
E[Socket Event] --> F[Validate Match ID]
F --> G[Check Duplicate]
G --> |Duplicate| H[Ignore Event]
G --> |New| I[Add to Live Bets]
I --> J[Update Statistics]
J --> K[Recalculate Unique Bettors]
K --> L[Update UI Incrementally]
M[Manual Refresh] --> N[Fetch Latest Bets]
N --> O[Replace Live Bets List]
O --> P[Recalculate All Stats]
```

**Diagram sources**
- [LiveBettingInterface.jsx](file://client/src/components/Bet/LiveBettingInterface.jsx#L75-L169)

#### Bet Placement Form
The form validates user input and handles various bet types:

```mermaid
flowchart TD
A[User Click Place Bet] --> B[Validate Selection]
B --> C{Bird Selected?}
C --> |No| D[Show Error: Select Bird]
C --> |Yes| E[Validate Amount]
E --> F{Valid Amount?}
F --> |No| G[Show Error: Invalid Amount]
F --> |Yes| H[Check Match Status]
H --> I{Match Active?}
I --> |No| J[Show Error: Betting Closed]
I --> |Yes| K[Check User Balance]
K --> L{Sufficient Funds?}
L --> |No| M[Show Error: Insufficient Balance]
L --> |Yes| N[Place Bet Request]
N --> O[Show Loading State]
O --> P[Bet Success]
P --> Q[Reset Form & Update UI]
P --> R[Show Success Toast]
P --> S[Refresh Match Data]
P --> T[Update User Balance]
```

**Diagram sources**
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L420-L517)

**Section sources**
- [LiveBettingInterface.jsx](file://client/src/components/Bet/LiveBettingInterface.jsx#L1-L439)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L1-L943)

### Live Streaming Integration
The system integrates multiple video streaming platforms:

#### Platform Support Matrix
```mermaid
graph TB
A[Video URL Processing] --> B{Detect Platform}
B --> C[Youtube]
B --> D[Vimeo]
B --> E[Twitch]
B --> F[Facebook]
B --> G[Dailymotion]
B --> H[Google Drive]
B --> I[Streamable]
B --> J[Direct Video]
B --> K[HLS Stream]
C --> L[Embed Template + Params]
D --> L
E --> L
F --> L
G --> L
H --> M[Direct File URL]
I --> L
J --> M
K --> N[HLS.js Player]
```

**Diagram sources**
- [LiveStream.jsx](file://client/src/components/Bet/LiveStream.jsx#L101-L218)

#### Streaming Features
- **Multi-platform support**: YouTube, Vimeo, Twitch, Facebook, Dailymotion, Google Drive, Streamable
- **HLS streaming**: Native Safari support with HLS.js polyfill for other browsers
- **Auto-play management**: Intelligent autoplay handling with mute controls
- **Error recovery**: Automatic retry mechanisms and user-initiated retries
- **Responsive design**: Adaptive video player with fullscreen capability

**Section sources**
- [LiveStream.jsx](file://client/src/components/Bet/LiveStream.jsx#L1-L462)

### Data Synchronization and State Management
The system ensures consistency between frontend and backend through multiple synchronization points:

#### Frontend State Management
```mermaid
sequenceDiagram
participant Redux as "Redux Store"
participant Page as "LiveBettingPage"
participant Interface as "LiveBettingInterface"
participant Socket as "SocketContext"
participant API as "REST API"
Page->>API : Fetch Match Data
API-->>Page : Match Information
Page->>Redux : Dispatch Actions
Redux-->>Page : Updated State
Socket->>Socket : Listen for Events
Socket-->>Page : Real-time Updates
Page->>Redux : Update State
Interface->>API : Fetch Live Bets
API-->>Interface : Bet History
Interface->>Redux : Update Live Bets
Page->>API : Place Bet
API-->>Page : Bet Confirmation
Page->>Redux : Update Balance & History
```

**Diagram sources**
- [index.js](file://client/src/store/user/match-and-bet-slice/index.js#L1-L127)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L1-L943)

#### Conflict Resolution Strategies
- **Processed Bet Tracking**: Prevents duplicate processing of the same bet event
- **Match ID Validation**: Ensures events are only processed for the current match
- **State Synchronization**: Manual refresh complements real-time updates
- **Local Storage Backup**: Maintains user bet history and close updates locally

**Section sources**
- [LiveBettingInterface.jsx](file://client/src/components/Bet/LiveBettingInterface.jsx#L35-L169)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L208-L408)

## Dependency Analysis
The system exhibits clean dependency management with clear separation of concerns:

```mermaid
graph TD
A[Client Main] --> B[SocketContext]
A --> C[Redux Store]
A --> D[Router]
B --> E[Socket.IO Client]
C --> F[Redux Toolkit]
D --> G[React Router]
H[Server Main] --> I[Express App]
H --> J[Socket.IO Server]
I --> K[REST Routes]
K --> L[Bet Controller]
K --> M[Match Routes]
L --> N[MongoDB Models]
M --> N
J --> O[Socket Handler]
O --> P[Room Management]
O --> Q[Event Broadcasting]
```

**Diagram sources**
- [main.jsx](file://client/src/main.jsx#L1-L20)
- [server.js](file://server/server.js#L1-L92)
- [index.js](file://server/index.js#L1-L150)

**Section sources**
- [main.jsx](file://client/src/main.jsx#L1-L20)
- [server.js](file://server/server.js#L1-L92)
- [index.js](file://server/index.js#L1-L150)

## Performance Considerations
The system implements several optimization techniques for handling high-frequency updates and multiple concurrent connections:

### Backend Optimizations
- **Connection Pooling**: Socket.IO server configured with optimal buffer sizes
- **Room-based Broadcasting**: Minimizes broadcast overhead by targeting specific rooms
- **Database Indexing**: Strategic indexes on frequently queried fields (createdAt, matchId, status)
- **Memory Management**: Proper cleanup of event listeners and room memberships

### Frontend Optimizations
- **Efficient State Updates**: Incremental updates to betting statistics instead of full recalculations
- **Debounced Updates**: Prevents excessive re-renders during rapid bet placements
- **Virtual Scrolling**: Efficient rendering of large bet history lists
- **Lazy Loading**: Conditional loading of HLS.js library only when needed

### Network Optimization
- **Transport Selection**: Automatic fallback between WebSocket and polling based on network conditions
- **Compression**: Configurable compression for reduced bandwidth usage
- **Heartbeat Monitoring**: Ping/pong mechanism for connection health verification

## Troubleshooting Guide

### Common Connection Issues
1. **Connection Drops**: Verify reconnection attempts are configured correctly
2. **Room Join Failures**: Check match IDs are properly formatted and exist in database
3. **Event Delivery**: Ensure proper event names match between frontend and backend

### Error Recovery Strategies
```mermaid
flowchart TD
A[Error Occurs] --> B{Type of Error?}
B --> C[Network Error]
B --> D[Validation Error]
B --> E[Database Error]
C --> F[Attempt Reconnection]
F --> G{Reconnection Success?}
G --> |Yes| H[Resume Normal Operation]
G --> |No| I[Show Connection Error]
D --> J[Display User-Friendly Message]
D --> K[Prevent Invalid Operations]
E --> L[Retry Operation]
L --> M{Operation Success?}
M --> |Yes| N[Continue Execution]
M --> |No| O[Log Error & Notify Admin]
```

### Debugging Tools
- **Server Logs**: Monitor connection events, room joins/leaves, and error messages
- **Client Console**: Track Socket.IO events and connection states
- **Database Queries**: Verify bet records and match status updates
- **Network Tab**: Inspect WebSocket frames and HTTP requests

**Section sources**
- [socketHandler.js](file://server/socket/socketHandler.js#L74-L88)
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx#L39-L47)

## Conclusion
The real-time betting system demonstrates robust implementation of Socket.IO for live sports betting applications. The room-based architecture ensures efficient communication isolation, while comprehensive reconnection logic provides resilience against network interruptions. The integration of live streaming enhances the user experience by combining betting functionality with match viewing capabilities.

Key strengths of the implementation include:
- **Scalable Architecture**: Room-based communication minimizes broadcast overhead
- **Robust Error Handling**: Comprehensive reconnection and recovery mechanisms
- **Performance Optimization**: Efficient state management and selective updates
- **Multi-platform Support**: Flexible video streaming integration
- **User Experience**: Real-time feedback and responsive interface

The system provides a solid foundation for real-time betting applications with clear extensibility points for additional features such as odds updates, advanced statistics, and enhanced administrative capabilities.
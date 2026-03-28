# Bet Model

<cite>
**Referenced Files in This Document**
- [betModel.js](file://server/models/betModel.js)
- [matchModel.js](file://server/models/matchModel.js)
- [authModel.js](file://server/models/authModel.js)
- [betController.js](file://server/controllers/bet/betController.js)
- [matchController.js](file://server/controllers/admin/matchController.js)
- [betRoute.js](file://server/routes/bet/betRoute.js)
- [socketHandler.js](file://server/socket/socketHandler.js)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx)
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx)
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
This document provides comprehensive documentation for the Bet Model schema and its ecosystem. It covers bet placement fields, relationships with authentication and match models, bet type classification (Straight, Lay90, Call90), bet outcome tracking, result settlement logic, profit/loss calculations, status management, settlement processing, historical record keeping, field validation, indexing for query performance, and real-time bet notifications via sockets. It also includes practical examples of bet placement, odds updates, result processing, and settlement workflows.

## Project Structure
The Bet Model resides in the server models and integrates with controllers, routes, and socket handlers. The client-side pages consume real-time updates and trigger bet placement actions.

```mermaid
graph TB
subgraph "Server"
A["Models<br/>betModel.js<br/>matchModel.js<br/>authModel.js"]
B["Controllers<br/>betController.js<br/>matchController.js"]
C["Routes<br/>betRoute.js"]
D["Socket<br/>socketHandler.js"]
end
subgraph "Client"
E["LiveBettingPage.jsx"]
F["SocketContext.jsx"]
end
E --> |HTTP requests| B
B --> |MongoDB| A
B --> |Socket events| D
D --> |Real-time updates| E
F --> |Socket client| D
```

**Diagram sources**
- [betModel.js](file://server/models/betModel.js#L1-L24)
- [matchModel.js](file://server/models/matchModel.js#L1-L101)
- [authModel.js](file://server/models/authModel.js#L1-L40)
- [betController.js](file://server/controllers/bet/betController.js#L1-L125)
- [matchController.js](file://server/controllers/admin/matchController.js#L1-L1188)
- [betRoute.js](file://server/routes/bet/betRoute.js#L1-L11)
- [socketHandler.js](file://server/socket/socketHandler.js#L1-L101)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L400-L599)
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx#L1-L62)

**Section sources**
- [betModel.js](file://server/models/betModel.js#L1-L24)
- [matchModel.js](file://server/models/matchModel.js#L1-L101)
- [authModel.js](file://server/models/authModel.js#L1-L40)
- [betController.js](file://server/controllers/bet/betController.js#L1-L125)
- [matchController.js](file://server/controllers/admin/matchController.js#L1-L1188)
- [betRoute.js](file://server/routes/bet/betRoute.js#L1-L11)
- [socketHandler.js](file://server/socket/socketHandler.js#L1-L101)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L400-L599)
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx#L1-L62)

## Core Components
- Bet Model: Defines the bet schema, relationships, and indexes.
- Match Model: Stores match metadata, status, and settlement results.
- Auth Model: Provides user identity and balance used during bet placement.
- Bet Controller: Handles bet placement, validation, user balance deduction, and real-time notifications.
- Admin Match Controller: Manages settlement, updates match status, and emits settlement events.
- Socket Handler: Initializes socket rooms and emits real-time events for clients.
- Client Pages: Validate inputs, submit bets, and render real-time updates.

Key schema highlights:
- Fields: user, matchId, matchTitle, selectedBird, amount, type, status, isRefunded, winnings, losing, refundedAmount, winningBird, actualAmount, timestamps.
- Enumerations: type includes Straight, Lay90, Call90; status includes Pending, Won, Lost, Refunded, Tie, Cancelled.
- Indexes: createdAt, (matchId, status), and others for performance.

**Section sources**
- [betModel.js](file://server/models/betModel.js#L3-L23)
- [matchModel.js](file://server/models/matchModel.js#L17-L96)
- [authModel.js](file://server/models/authModel.js#L3-L37)
- [betController.js](file://server/controllers/bet/betController.js#L42-L106)
- [matchController.js](file://server/controllers/admin/matchController.js#L1120-L1165)

## Architecture Overview
The Bet Model participates in a request-response flow for placing bets and a real-time event-driven flow for live updates. Settlement is handled by the admin controller, which updates match status and emits settlement events.

```mermaid
sequenceDiagram
participant Client as "Client App"
participant Page as "LiveBettingPage.jsx"
participant Ctrl as "betController.js"
participant Auth as "authModel.js"
participant Match as "matchModel.js"
participant Bet as "betModel.js"
participant Sock as "socketHandler.js"
Client->>Page : "User selects bet and amount"
Page->>Ctrl : "POST /bet/create"
Ctrl->>Auth : "Validate user and balance"
Ctrl->>Match : "Validate match active"
Ctrl->>Bet : "Create bet document"
Ctrl->>Sock : "Emit 'new-bet' to match room"
Sock-->>Page : "Receive 'new-bet' event"
Page-->>Client : "Render updated bet feed"
```

**Diagram sources**
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L420-L517)
- [betController.js](file://server/controllers/bet/betController.js#L42-L106)
- [authModel.js](file://server/models/authModel.js#L1-L40)
- [matchModel.js](file://server/models/matchModel.js#L1-L101)
- [betModel.js](file://server/models/betModel.js#L1-L24)
- [socketHandler.js](file://server/socket/socketHandler.js#L58-L72)

## Detailed Component Analysis

### Bet Model Schema
The Bet Model defines the core structure for bets, including references to users and matches, bet type classification, and financial outcome fields.

```mermaid
erDiagram
AUTH {
ObjectId _id PK
string name
string email UK
string phone UK
number balance
}
MATCH {
ObjectId _id PK
ObjectId topLevelMatch FK
string BirdA
string BirdB
string section
number round
string status
string winningBird
}
BET {
ObjectId _id PK
ObjectId user FK
ObjectId matchId FK
string matchTitle
string selectedBird
number amount
string type
string status
boolean isRefunded
number winnings
number losing
number refundedAmount
string winningBird
number actualAmount
date createdAt
date updatedAt
}
AUTH ||--o{ BET : "owns"
MATCH ||--o{ BET : "contains"
```

- Relationships:
  - user references Auth.
  - matchId references Match.
- Enumerations:
  - type: Straight, Lay90, Call90.
  - status: Pending, Won, Lost, Refunded, Tie, Cancelled.
- Financial fields:
  - amount: staked amount.
  - actualAmount: settled matched amount.
  - winnings: total win credited.
  - losing: total loss debited.
  - refundedAmount: refund amount processed.
- Timestamps:
  - createdAt, updatedAt managed automatically.

**Diagram sources**
- [betModel.js](file://server/models/betModel.js#L3-L23)
- [authModel.js](file://server/models/authModel.js#L3-L37)
- [matchModel.js](file://server/models/matchModel.js#L17-L34)

**Section sources**
- [betModel.js](file://server/models/betModel.js#L3-L23)

### Bet Placement Workflow
This workflow covers validation, balance checks, bet creation, and real-time notification.

```mermaid
sequenceDiagram
participant UI as "LiveBettingPage.jsx"
participant Ctrl as "betController.js"
participant Auth as "authModel.js"
participant Match as "matchModel.js"
participant Bet as "betModel.js"
participant Sock as "socketHandler.js"
UI->>Ctrl : "placeBet(userId, matchId, selectedBird, amount, type)"
Ctrl->>Ctrl : "Validate required fields"
Ctrl->>Auth : "Find user and check balance"
Ctrl->>Match : "Find match and check status"
Ctrl->>Ctrl : "Round amount to 2 decimals"
Ctrl->>Auth : "Deduct amount from user.balance"
Ctrl->>Bet : "Create Bet document"
Ctrl->>Sock : "Emit 'new-bet' to 'match-{matchId}'"
Sock-->>UI : "Clients receive bet update"
```

Validation and constraints:
- Required fields: matchId, selectedBird, amount, type, userId.
- Amount must be greater than zero.
- User must exist and have sufficient balance.
- Match must be Active.

Real-time notification:
- Emits to the match-specific room only.
- Uses socket rooms to avoid broadcasting to unrelated clients.

**Diagram sources**
- [betController.js](file://server/controllers/bet/betController.js#L42-L106)
- [authModel.js](file://server/models/authModel.js#L1-L40)
- [matchModel.js](file://server/models/matchModel.js#L1-L101)
- [socketHandler.js](file://server/socket/socketHandler.js#L58-L72)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L420-L517)

**Section sources**
- [betController.js](file://server/controllers/bet/betController.js#L42-L106)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L420-L517)

### Bet Outcome Tracking and Settlement Logic
Settlement is performed by the admin controller upon match closure. It updates bet statuses, credits/debits balances, and emits settlement events.

```mermaid
flowchart TD
Start(["Admin triggers settle match"]) --> Load["Load match and closeResults"]
Load --> Unmatched["Refund unmatched amounts to users"]
Unmatched --> Pairs{"Lay90/Call90 pairs?"}
Pairs --> |Yes| LayCall["Apply Lay90/Call90 rules:<br/>81% net for Lay, 90% net for Call"]
Pairs --> |No| Straight["Apply Straight rules:<br/>Win/Loss per pair"]
LayCall --> UpdateUsers["Update user balances and bet fields"]
Straight --> UpdateUsers
UpdateUsers --> UpdateBets["Update bet.status, winnings, losing,<br/>refundedAmount, actualAmount"]
UpdateBets --> UpdateMatch["Update match.winningBird and status"]
UpdateMatch --> Emit["Emit 'match-settled' and bet history updates"]
Emit --> End(["Complete"])
```

Settlement specifics:
- Lay90/Call90:
  - Lay risk is matched; Lay receives 81% net plus stake; Call receives 90% net plus stake.
  - Winner status set accordingly; actualAmount derived from closeResults.
- Straight:
  - Winner receives gross return plus stake; loser’s stake remains as loss.
- Tie/Cancelled:
  - Bets refunded; status reflects Tie or Cancelled; actualAmount reflects refunded amount.

**Diagram sources**
- [matchController.js](file://server/controllers/admin/matchController.js#L785-L1165)

**Section sources**
- [matchController.js](file://server/controllers/admin/matchController.js#L785-L1165)

### Profit/Loss Calculation and Historical Records
Profit/loss fields:
- winnings: cumulative credited wins per bet.
- losing: cumulative debits for losses per bet.
- refundedAmount: refunded portion of a bet.
- actualAmount: the matched amount used for settlement computations.

Historical records:
- Bet documents persist with timestamps and status changes.
- Settlement updates are broadcast to users via socket events and stored in bet documents.

**Section sources**
- [betModel.js](file://server/models/betModel.js#L11-L17)
- [matchController.js](file://server/controllers/admin/matchController.js#L980-L1121)

### Real-Time Bet Notifications and Socket Integration
Socket rooms:
- Clients join match-specific rooms to receive targeted updates.
- Admin and global rooms receive administrative and event-wide updates.

Client-side integration:
- Socket provider establishes persistent connections.
- Pages subscribe to rooms and listen for events to update UI in real time.

```mermaid
sequenceDiagram
participant Client as "LiveBettingPage.jsx"
participant SockCtx as "SocketContext.jsx"
participant Sock as "socketHandler.js"
Client->>SockCtx : "Initialize socket"
SockCtx-->>Client : "Provide socket instance"
Client->>Sock : "Join 'match-{matchId}' room"
Sock-->>Client : "Receive 'new-bet' updates"
Client->>Sock : "Leave room on unmount"
```

**Diagram sources**
- [socketHandler.js](file://server/socket/socketHandler.js#L6-L40)
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx#L14-L54)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L400-L408)

**Section sources**
- [socketHandler.js](file://server/socket/socketHandler.js#L6-L40)
- [SocketContext.jsx](file://client/src/context/SocketContext.jsx#L14-L54)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L400-L408)

### Field Validation and Indexing
Field validation:
- Required fields enforced at controller level.
- Amount validated to be positive.
- User balance checked before deduction.
- Match status validated to be Active.

Indexing for query performance:
- Bet model indexes:
  - createdAt: descending for recent-first queries.
  - (matchId, status): composite index for efficient filtering by match and status.
- Match model indexes:
  - (topLevelMatch, section, round): for grouping rounds within sections.
  - status, createdAt: for status-based queries.
- Auth model indexes:
  - email, name, role, createdAt: for user lookups and analytics.

**Section sources**
- [betModel.js](file://server/models/betModel.js#L21-L22)
- [matchModel.js](file://server/models/matchModel.js#L94-L96)
- [authModel.js](file://server/models/authModel.js#L33-L37)
- [betController.js](file://server/controllers/bet/betController.js#L46-L58)

### Examples

#### Example 1: Bet Placement
- Input: userId, matchId, selectedBird, amount, type.
- Validation: fields present, amount > 0, user exists, user balance sufficient, match Active.
- Action: deduct amount from user.balance, create Bet document, emit 'new-bet' to match room.
- Output: success payload with bet details.

**Section sources**
- [betController.js](file://server/controllers/bet/betController.js#L42-L106)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L420-L517)

#### Example 2: Odds Updates and Live Feed
- Client joins match room and listens for 'new-bet'.
- On each bet placement, clients receive the latest bet payload and update the UI.

**Section sources**
- [socketHandler.js](file://server/socket/socketHandler.js#L58-L72)
- [LiveBettingPage.jsx](file://client/src/Pages/Bet/LiveBettingPage.jsx#L400-L408)

#### Example 3: Result Processing and Settlement
- Admin settles match with a winningBird or tie/cancelled.
- System refunds unmatched, applies Lay90/Call90 or Straight rules, updates user balances and bet fields, and emits settlement events.

**Section sources**
- [matchController.js](file://server/controllers/admin/matchController.js#L785-L1165)

## Dependency Analysis
The Bet Model depends on Auth and Match models. Controllers orchestrate validation, persistence, and real-time updates. Routes expose endpoints for bet operations. Socket handlers manage rooms and event broadcasting.

```mermaid
graph TB
Bet["Bet Model (betModel.js)"]
Auth["Auth Model (authModel.js)"]
Match["Match Model (matchModel.js)"]
BCtrl["Bet Controller (betController.js)"]
MCtrl["Admin Match Controller (matchController.js)"]
Route["Bet Routes (betRoute.js)"]
Sock["Socket Handler (socketHandler.js)"]
BCtrl --> Bet
BCtrl --> Auth
BCtrl --> Match
BCtrl --> Sock
MCtrl --> Bet
MCtrl --> Auth
MCtrl --> Match
MCtrl --> Sock
Route --> BCtrl
```

**Diagram sources**
- [betModel.js](file://server/models/betModel.js#L1-L24)
- [authModel.js](file://server/models/authModel.js#L1-L40)
- [matchModel.js](file://server/models/matchModel.js#L1-L101)
- [betController.js](file://server/controllers/bet/betController.js#L1-L125)
- [matchController.js](file://server/controllers/admin/matchController.js#L1-L1188)
- [betRoute.js](file://server/routes/bet/betRoute.js#L1-L11)
- [socketHandler.js](file://server/socket/socketHandler.js#L1-L101)

**Section sources**
- [betModel.js](file://server/models/betModel.js#L1-L24)
- [authModel.js](file://server/models/authModel.js#L1-L40)
- [matchModel.js](file://server/models/matchModel.js#L1-L101)
- [betController.js](file://server/controllers/bet/betController.js#L1-L125)
- [matchController.js](file://server/controllers/admin/matchController.js#L1-L1188)
- [betRoute.js](file://server/routes/bet/betRoute.js#L1-L11)
- [socketHandler.js](file://server/socket/socketHandler.js#L1-L101)

## Performance Considerations
- Use indexes on frequently queried fields:
  - Bet: createdAt, (matchId, status).
  - Match: (topLevelMatch, section, round), status, createdAt.
  - Auth: email, name, role, createdAt.
- Round amounts to two decimal places to prevent precision drift.
- Emit socket events only to relevant rooms to reduce bandwidth.
- Batch UI updates after receiving settlement events to minimize re-renders.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and resolutions:
- Missing required fields: Ensure matchId, selectedBird, amount, type, userId are provided.
- Insufficient funds: Verify user.balance >= amount before bet placement.
- Match not Active: Confirm match.status is Active before accepting bets.
- Socket emit errors: Check socket initialization and room membership.
- Settlement discrepancies: Validate closeResults and bet.actualAmount alignment.

**Section sources**
- [betController.js](file://server/controllers/bet/betController.js#L46-L58)
- [socketHandler.js](file://server/socket/socketHandler.js#L93-L98)
- [matchController.js](file://server/controllers/admin/matchController.js#L785-L1165)

## Conclusion
The Bet Model schema provides a robust foundation for capturing bets, linking them to users and matches, and enabling real-time updates. Settlement logic supports both Straight and Lay90/Call90 bet types, with clear profit/loss tracking and refund mechanisms. Proper indexing and room-based socket communication ensure scalable and responsive performance. The documented workflows and examples offer practical guidance for implementing and maintaining the betting system.
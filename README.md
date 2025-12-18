# Tween Mini-App Communication Protocol (TMCP)

**TRFC:** 1001  
**Category:** Proposed Standard  
**Date:** December 2025  
**Authors:** Ezeani Emmanuel
**Handle:** @mona:tween.im

---

## Abstract

This document specifies the Tween Mini-App Communication Protocol (TMCP), a comprehensive protocol for secure communication between instant messaging applications and third-party mini-applications. Built as an isolated Application Service layer on the Matrix protocol, TMCP provides authentication, authorization, and wallet-based payment processing without modifying Matrix/Synapse core code. The protocol enables a super-app ecosystem with integrated wallet services, instant peer-to-peer transfers, mini-app payments, and social commerce within a closed federation environment.

---

## Status of This Memo

This document specifies a Proposed Standard protocol for the Internet community, and requests discussion and suggestions for improvements. Distribution of this memo is unlimited.

---

## Copyright Notice

Copyright (c) 2025 Tween IM. All rights reserved.

---

## Table of Contents

1. [Introduction](#1-introduction)  
2. [Conventions and Terminology](#2-conventions-and-terminology)  
3. [Protocol Architecture](#3-protocol-architecture)  
4. [Identity and Authentication](#4-identity-and-authentication)  
5. [Authorization Framework](#5-authorization-framework)  
6. [Wallet Integration Layer](#6-wallet-integration-layer)  
7. [Payment Protocol](#7-payment-protocol)  
8. [Event System](#8-event-system)  
9. [Mini-App Lifecycle](#9-mini-app-lifecycle)  
10. [Communication Verbs](#10-communication-verbs)  
11. [Security Considerations](#11-security-considerations)  
12. [Error Handling](#12-error-handling)  
13. [Federation Considerations](#13-federation-considerations)  
14. [IANA Considerations](#14-iana-considerations)  
15. [References](#15-references)  
16. [Appendices](#16-appendices)

---

## 1. Introduction

### 1.1 Motivation

Modern instant messaging platforms increasingly serve as super-apps that integrate communication, commerce, and financial services. This specification defines a protocol that enables such functionality while maintaining protocol isolation from the underlying communication infrastructure.

The Tween Mini-App Communication Protocol (TMCP) addresses the following requirements:

- **Protocol Isolation**: Extensions to Matrix without core modifications
- **Wallet-Centric Architecture**: Integrated financial services as first-class citizens
- **Peer-to-Peer Transactions**: Direct value transfer between users within conversations
- **Mini-Application Ecosystem**: Third-party application integration with standardized APIs
- **Closed Federation**: Internal server infrastructure with centralized wallet management

### 1.2 Design Goals

**MUST Requirements:**
- Zero modification to Matrix/Synapse core protocol
- OAuth 2.0 + PKCE compliance for authentication
- Strong cryptographic signing for payment transactions
- Matrix Application Service API compatibility
- Real-time bidirectional communication
- Idempotent payment processing

**SHOULD Requirements:**
- Sub-200ms API response times for non-payment operations
- Sub-3s settlement time for peer-to-peer transfers
- Horizontal scalability across internal server instances
- Backwards compatibility for protocol updates

### 1.3 Scope

This specification defines:
- Mini-application registration and lifecycle management
- OAuth 2.0 authentication and authorization flows
- Wallet API for balance queries and peer-to-peer transfers
- Payment authorization protocol for mini-app transactions
- Event-driven communication patterns using Matrix events
- Security mechanisms for payment and data protection

This specification does NOT define:
- Wallet backend implementation details
- Matrix core protocol modifications
- Client user interface requirements
- External banking system integration specifics

---

## 2. Conventions and Terminology

### 2.1 Requirements Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 [RFC2119].

### 2.2 Matrix Protocol Terms

**Homeserver**  
A Matrix server instance responsible for maintaining user state and federating events. In TMCP deployments, homeservers exist within a closed federation environment.

**User ID**  
Matrix user identifier in the format `@localpart:domain`. Example: `@alice:tween.example`

**Room**  
A persistent conversation context where events are shared between participants.

**Event**  
A JSON object representing an action, message, or state change in the Matrix ecosystem.

**Application Service (AS)**  
A server-side extension mechanism defined by the Matrix Application Service API that enables third-party services to integrate with a homeserver without modifying its core code.

### 2.3 TMCP-Specific Terms

**Mini-App (MA)**  
A third-party application running within the Tween client environment. Mini-Apps execute in sandboxed contexts and communicate with the host application via standardized APIs.

**Mini-App ID**  
Unique identifier for a registered mini-app, format: `ma_` followed by alphanumeric characters. Example: `ma_shop_001`

**TMCP Server**  
Application Service implementation that handles mini-app protocol operations including authentication, payment processing, and event routing.

**Tween Wallet**  
Integrated wallet service for storing digital currency balances and processing financial transactions.

**Wallet ID**  
User wallet identifier, format: `tw_` followed by alphanumeric characters. Example: `tw_user_12345`

**P2P Transfer**  
Peer-to-peer direct value transfer between user wallets within chat conversations.

**TEP Token (TMCP Extension Protocol Token)**  
JWT-based access token issued by the TMCP Server for mini-app authentication, distinct from Matrix access tokens.

---

## 3. Protocol Architecture

### 3.1 System Components

TMCP operates as an isolated layer that extends Matrix capabilities without modifying its core. The architecture consists of four primary components:

```
┌─────────────────────────────────────────────────────────┐
│                 TWEEN CLIENT APPLICATION                 │
│  ┌──────────────┐         ┌──────────────────────┐    │
│  │ Matrix SDK   │         │ TMCP Bridge          │    │
│  │ (Element)    │◄───────►│ (Mini-App Runtime)   │    │
│  └──────────────┘         └──────────────────────┘    │
└────────────┬──────────────────────┬───────────────────┘
             │                      │
             │ Matrix Client-       │ TMCP Protocol
             │ Server API           │ (JSON-RPC 2.0)
             │                      │
             ↓                      ↓
┌──────────────────┐     ┌──────────────────────────┐
│ Matrix Homeserver│◄───►│   TMCP Server            │
│ (Synapse)        │     │   (Application Service)  │
└──────────────────┘     └──────────────────────────┘
        │                          │
        │ Matrix                   ├──→ OAuth 2.0 Service
        │ Application              ├──→ Payment Processor
        │ Service API              ├──→ Mini-App Registry
        │                          └──→ Event Router
        │
        ↓
┌──────────────────┐     ┌──────────────────────────┐
│ Matrix Event     │     │   Tween Wallet Service   │
│ Store (DAG)      │     │   (gRPC/REST)            │
└──────────────────┘     └──────────────────────────┘
```

#### 3.1.1 Tween Client

The client application is a forked version of Element that implements the TMCP Bridge. Key responsibilities:

- **Matrix SDK Integration**: Standard Matrix client-server communication
- **TMCP Bridge**: WebView/iframe sandbox for mini-app execution
- **Hardware Security**: Leverages Secure Enclave (iOS) or TEE (Android) for payment signing
- **Event Rendering**: Custom rendering for TMCP-specific Matrix events

#### 3.1.2 TMCP Server (Application Service)

Server-side component that implements the Matrix Application Service API and provides TMCP-specific functionality:

**Registration with Homeserver:**
```yaml
# Application Service Configuration
id: tween-miniapps
url: https://tmcp.internal.example.com
as_token: <APPLICATION_SERVICE_TOKEN>
hs_token: <HOMESERVER_TOKEN>
sender_localpart: _tmcp
namespaces:
  users:
    - exclusive: true
      regex: "@_tmcp_.*"
    - exclusive: true
      regex: "@ma_.*"
  aliases:
    - exclusive: true
      regex: "#_tmcp_.*"
  rooms: []
rate_limited: false
```

**Functional Modules:**

1. **OAuth Service** - Issues TEP tokens, manages scopes
2. **Payment Processor** - Coordinates wallet transactions
3. **Mini-App Registry** - Stores application metadata
4. **Event Router** - Routes Matrix events to mini-apps
5. **Webhook Manager** - Dispatches notifications to mini-apps

#### 3.1.3 Matrix Homeserver

Standard Synapse homeserver with Application Service support. Responsibilities:

- Event persistence and ordering
- Room state management
- Federation (disabled in closed federation mode)
- Access control and authentication

#### 3.1.4 Tween Wallet Service

Separate service managing financial operations:

- Balance management
- Transaction ledger
- Payment settlement
- External gateway integration (bank APIs, payment processors)

### 3.2 Communication Patterns

#### 3.2.1 Client-to-Server Communication

**Matrix Protocol Path:**
```
Client → Matrix Client-Server API → Homeserver → Event Store
```

**TMCP Protocol Path:**
```
Client (Mini-App) → TMCP Bridge → TMCP Server → Wallet/Registry
```

#### 3.2.2 Event Flow for Payment Transaction

```
User initiates payment in Mini-App
     ↓
Mini-App calls TEP Bridge API
     ↓
Client displays payment confirmation UI
     ↓
User authorizes with biometric/PIN
     ↓
Client signs transaction with hardware key
     ↓
Signed transaction sent to TMCP Server
     ↓
TMCP Server validates signature
     ↓
TMCP Server coordinates with Wallet Service
     ↓
Wallet Service executes transfer
     ↓
TMCP Server creates Matrix event (m.tween.payment.completed)
     ↓
Homeserver persists event and distributes to room participants
     ↓
Client renders payment receipt
     ↓
Mini-App receives webhook notification
```

### 3.3 Protocol Layers

**Layer 1: Transport**
- HTTPS/TLS 1.3 (REQUIRED)
- WebSocket for real-time bidirectional communication
- Matrix federation protocol (closed federation only)

**Layer 2: Authentication**
- OAuth 2.0 with PKCE for mini-app authorization
- Matrix access tokens for client-server communication
- JWT (TEP tokens) for mini-app session management
- Hardware-backed signing for payments

**Layer 3: Application**
- JSON-RPC 2.0 for TMCP Bridge communication
- RESTful APIs for server-side operations
- Matrix custom events (m.tween.*) for state and messaging

**Layer 4: Security**
- End-to-end encryption (Matrix Olm/Megolm) for sensitive events
- HMAC-SHA256 for webhook signatures
- Request signing for payment authorization
- Content Security Policy for mini-app sandboxing

---

## 4. Identity and Authentication

### 4.1 Authentication Architecture

TMCP implements a dual-token system that separates Matrix identity from mini-app authorization:

1. **Matrix Access Token**: Used for Matrix client-server API calls
2. **TEP Token**: Used for mini-app-specific operations

This separation ensures mini-apps never access the user's primary Matrix credentials or encryption keys.

### 4.2 Mini-App Authentication Flow

#### 4.2.1 Authorization Code Flow with PKCE

TMCP implements OAuth 2.0 Authorization Code Flow with Proof Key for Code Exchange (PKCE) as defined in RFC 7636 [RFC7636].

**Step 1: Generate Code Verifier and Challenge**

The client MUST generate a code verifier and code challenge:

```javascript
// Generate code verifier (43-128 characters)
const codeVerifier = base64url(randomBytes(32));

// Generate code challenge
const codeChallenge = base64url(sha256(codeVerifier));
```

**Step 2: Authorization Request**

```http
GET /oauth2/authorize?
    response_type=code&
    client_id=ma_shop_001&
    redirect_uri=https://miniapp.example.com/callback&
    scope=user:read+wallet:pay&
    state=random_state_string&
    code_challenge=BASE64URL(SHA256(code_verifier))&
    code_challenge_method=S256
    
Host: tmcp.example.com
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| response_type | Yes | MUST be "code" |
| client_id | Yes | Mini-App ID |
| redirect_uri | Yes | HTTPS callback URL |
| scope | Yes | Space-separated scope list |
| state | Yes | CSRF protection token |
| code_challenge | Yes | Base64URL encoded SHA256 hash |
| code_challenge_method | Yes | MUST be "S256" |

**Step 3: User Consent**

The TMCP Server redirects to the client application, which displays a native consent screen showing:

```json
{
  "miniapp": {
    "id": "ma_shop_001",
    "name": "Shopping Assistant",
    "developer": "Example Corp",
    "icon_url": "https://cdn.example.com/icon.png",
    "verified": true
  },
  "requested_scopes": [
    {
      "scope": "user:read",
      "description": "Access your profile information",
      "sensitivity": "low"
    },
    {
      "scope": "wallet:pay",
      "description": "Process payments from your wallet",
      "sensitivity": "high",
      "note": "You'll confirm each payment"
    }
  ]
}
```

**Step 4: Authorization Response**

Upon user approval:

```http
HTTP/1.1 302 Found
Location: https://miniapp.example.com/callback?
    code=auth_abc123xyz&
    state=random_state_string
```

**Step 5: Token Request**

The mini-app backend exchanges the authorization code for tokens:

```http
POST /oauth2/token HTTP/1.1
Host: tmcp.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=auth_abc123xyz&
redirect_uri=https://miniapp.example.com/callback&
client_id=ma_shop_001&
client_secret=<CLIENT_SECRET>&
code_verifier=<CODE_VERIFIER>
```

**Step 6: Token Response**

```json
{
  "access_token": "tep.eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "rt_abc123...",
  "scope": "user:read wallet:pay",
  "user_id": "@alice:tween.example",
  "wallet_id": "tw_user_12345"
}
```

### 4.3 TEP Token Structure

TEP tokens are JSON Web Tokens (JWT) as defined in RFC 7519 [RFC7519].

**Header:**
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "tmcp-2024-12"
}
```

**Payload:**
```json
{
  "iss": "https://tmcp.example.com",
  "sub": "@alice:tween.example",
  "aud": "ma_shop_001",
  "exp": 1704067200,
  "iat": 1704063600,
  "jti": "unique-token-id",
  "scope": "user:read wallet:pay",
  "wallet_id": "tw_user_12345",
  "session_id": "session_xyz789",
  "miniapp_context": {
    "launch_source": "chat_bubble",
    "room_id": "!abc123:tween.example"
  }
}
```

**Claims:**

| Claim | Description |
|-------|-------------|
| iss | Issuer (TMCP Server URL) |
| sub | Subject (Matrix User ID) |
| aud | Audience (Mini-App ID) |
| exp | Expiration time (Unix timestamp) |
| iat | Issued at (Unix timestamp) |
| jti | JWT ID (unique identifier) |
| scope | Granted scopes |
| wallet_id | User's wallet identifier |
| session_id | Session identifier |
| miniapp_context | Launch context information |

### 4.4 Token Refresh

```http
POST /oauth2/token HTTP/1.1
Host: tmcp.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=rt_abc123...&
client_id=ma_shop_001&
client_secret=<CLIENT_SECRET>
```

Response:
```json
{
  "access_token": "tep.newtoken...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "rt_newtoken...",
  "scope": "user:read wallet:pay"
}
```

**Token Rotation:**
- Refresh tokens MUST be single-use
- Old refresh token MUST be invalidated upon successful rotation
- Access token expiration: 1 hour (RECOMMENDED)
- Refresh token expiration: 30 days (RECOMMENDED)

### 4.5 Matrix Integration

TMCP creates virtual Matrix users for mini-app bots to send messages:

```http
POST /_matrix/client/v3/login HTTP/1.1
Authorization: Bearer <APP_SERVICE_TOKEN>
Content-Type: application/json

{
  "type": "m.login.application_service",
  "identifier": {
    "type": "m.id.user",
    "user": "@ma_shop_001_bot:tween.example"
  }
}
```

This allows mini-apps to:
- Send rich messages to rooms
- Participate in conversations as automated agents
- Create room state events for payment confirmations

---

## 5. Authorization Framework

### 5.1 Scope Definitions

Scopes define the permissions granted to mini-apps. Each scope MUST be explicitly requested during authorization and approved by the user.

**Scope Naming Convention:**
```
<category>:<action>[:<resource>]
```

**Standard Scopes:**

| Scope | Description | Sensitivity | User Approval Required |
|-------|-------------|-------------|------------------------|
| `user:read` | Read basic profile (name, avatar) | Low | Yes |
| `user:read:extended` | Read extended profile (status, bio) | Medium | Yes |
| `user:read:contacts` | Read friend list | High | Yes |
| `wallet:balance` | Read wallet balance | High | Yes |
| `wallet:pay` | Process payments | Critical | Yes (per transaction) |
| `wallet:history` | Read transaction history | High | Yes |
| `messaging:send` | Send messages to rooms | High | Yes |
| `messaging:read` | Read message history | High | Yes |
| `storage:read` | Read mini-app storage | Low | No |
| `storage:write` | Write to mini-app storage | Low | No |

### 5.2 Scope Validation

The TMCP Server MUST validate that all requested scopes are:
1. Syntactically valid
2. Registered for the requesting mini-app
3. Not escalated beyond initial registration

### 5.3 Permission Revocation

Users MAY revoke permissions at any time. When permissions are revoked:

1. TMCP Server MUST invalidate all TEP tokens for that mini-app/user pair
2. A Matrix state event MUST be created documenting the revocation:

```json
{
  "type": "m.room.tween.authorization",
  "state_key": "ma_shop_001",
  "content": {
    "authorized": false,
    "revoked_at": 1704067200,
    "revoked_scopes": ["wallet:pay"],
    "reason": "user_initiated"
  }
}
```

3. A webhook notification MUST be sent to the mini-app

---

## 6. Wallet Integration Layer

### 6.1 Wallet Architecture

The Tween Wallet Service operates independently from the TMCP Server and Matrix Homeserver:

```
TMCP Server ←→ gRPC/REST ←→ Wallet Service ←→ External Gateways
```

### 6.2 Wallet API Endpoints

#### 6.2.1 Get Balance

```http
GET /wallet/v1/balance HTTP/1.1
Host: tmcp.example.com
Authorization: Bearer <TEP_TOKEN>
```

**Response:**
```json
{
  "wallet_id": "tw_user_12345",
  "user_id": "@alice:tween.example",
  "balance": {
    "available": 50000.00,
    "pending": 1500.00,
    "currency": "USD"
  },
  "limits": {
    "daily_limit": 100000.00,
    "daily_used": 25000.00,
    "transaction_limit": 50000.00
  },
  "status": "active"
}
```

#### 6.2.2 Transaction History

```http
GET /wallet/v1/transactions?limit=50&offset=0 HTTP/1.1
Host: tmcp.example.com
Authorization: Bearer <TEP_TOKEN>
```

**Response:**
```json
{
  "transactions": [
    {
      "txn_id": "txn_abc123",
      "type": "p2p_received",
      "amount": 5000.00,
      "currency": "USD",
      "from": {
        "user_id": "@bob:tween.example",
        "display_name": "Bob"
      },
      "status": "completed",
      "note": "Lunch money",
      "timestamp": "2024-12-18T12:00:00Z",
      "room_id": "!chat:tween.example"
    }
  ],
  "pagination": {
    "total": 245,
    "limit": 50,
    "offset": 0,
    "has_more": true
  }
}
```

---

## 7. Payment Protocol

### 7.1 Payment State Machine

Payments transition through well-defined states:

```
INITIATED → AUTHORIZED → PROCESSING → COMPLETED
              ↓              ↓            
          EXPIRED        FAILED      
              ↓              ↓            
          CANCELLED ←───────┘
```

### 7.2 Peer-to-Peer Transfer

#### 7.2.1 Initiate Transfer

```http
POST /wallet/v1/p2p/initiate HTTP/1.1
Host: tmcp.example.com
Authorization: Bearer <TEP_TOKEN>
Content-Type: application/json

{
  "recipient": "@bob:tween.example",
  "amount": 5000.00,
  "currency": "USD",
  "note": "Lunch money",
  "idempotency_key": "unique-uuid-here"
}
```

**Idempotency Requirements:**
- Clients MUST include a unique idempotency key
- Servers MUST cache keys for 24 hours minimum
- Duplicate requests MUST return original response

**Response:**
```json
{
  "transfer_id": "p2p_abc123",
  "status": "completed",
  "amount": 5000.00,
  "sender": {
    "user_id": "@alice:tween.example",
    "wallet_id": "tw_user_12345"
  },
  "recipient": {
    "user_id": "@bob:tween.example",
    "wallet_id": "tw_user_67890"
  },
  "timestamp": "2024-12-18T14:30:00Z",
  "event_id": "$event_abc123:tween.example"
}
```

#### 7.2.2 Matrix Event for P2P Transfer

The TMCP Server MUST create a Matrix event documenting the transfer:

```json
{
  "type": "m.tween.wallet.p2p",
  "content": {
    "msgtype": "m.tween.money",
    "body": "💸 Sent $5,000.00",
    "transfer_id": "p2p_abc123",
    "amount": 5000.00,
    "currency": "USD",
    "note": "Lunch money",
    "sender": {
      "user_id": "@alice:tween.example"
    },
    "recipient": {
      "user_id": "@bob:tween.example"
    },
    "status": "completed",
    "timestamp": "2024-12-18T14:30:00Z"
  },
  "room_id": "!chat:tween.example",
  "sender": "@alice:tween.example"
}
```

### 7.3 Mini-App Payment Flow

#### 7.3.1 Payment Request

```http
POST /api/v1/payments/request HTTP/1.1
Host: tmcp.example.com
Authorization: Bearer <TEP_TOKEN>
Content-Type: application/json

{
  "amount": 15000.00,
  "currency": "USD",
  "description": "Order #12345",
  "merchant_order_id": "ORDER-2024-12345",
  "items": [
    {
      "item_id": "prod_123",
      "name": "Product Name",
      "quantity": 2,
      "unit_price": 7500.00
    }
  ],
  "callback_url": "https://miniapp.example.com/webhooks/payment",
  "idempotency_key": "unique-uuid-here"
}
```

**Response:**
```json
{
  "payment_id": "pay_abc123",
  "status": "pending_authorization",
  "amount": 15000.00,
  "currency": "USD",
  "merchant": {
    "miniapp_id": "ma_shop_001",
    "name": "Shopping Assistant",
    "wallet_id": "tw_merchant_001"
  },
  "authorization_required": true,
  "expires_at": "2024-12-18T14:35:00Z",
  "created_at": "2024-12-18T14:30:00Z"
}
```

#### 7.3.2 Payment Authorization

The client displays a native payment confirmation UI. User authorizes using:
- Biometric authentication (fingerprint, face recognition)
- PIN code
- Hardware security module

**Authorization Signature:**

```javascript
// Client-side signing
const paymentHash = sha256(
  `${payment_id}:${amount}:${currency}:${timestamp}`
);

const signature = sign(paymentHash, hardwarePrivateKey);
```

**Submit Authorization:**

```http
POST /api/v1/payments/{payment_id}/authorize HTTP/1.1
Host: tmcp.example.com
Authorization: Bearer <TEP_TOKEN>
Content-Type: application/json

{
  "signature": "base64_encoded_signature",
  "device_id": "device_xyz789",
  "timestamp": "2024-12-18T14:30:15Z"
}
```

#### 7.3.3 Payment Completion

**Response:**
```json
{
  "payment_id": "pay_abc123",
  "status": "completed",
  "txn_id": "txn_def456",
  "amount": 15000.00,
  "payer": {
    "user_id": "@alice:tween.example",
    "wallet_id": "tw_user_12345"
  },
  "merchant": {
    "miniapp_id": "ma_shop_001",
    "wallet_id": "tw_merchant_001"
  },
  "completed_at": "2024-12-18T14:30:20Z"
}
```

**Matrix Event:**

```json
{
  "type": "m.tween.payment.completed",
  "content": {
    "msgtype": "m.tween.payment",
    "body": "Payment of $15,000.00 completed",
    "payment_id": "pay_abc123",
    "txn_id": "txn_def456",
    "amount": 15000.00,
    "merchant": {
      "miniapp_id": "ma_shop_001",
      "name": "Shopping Assistant"
    },
    "status": "completed"
  }
}
```

### 7.4 Refunds

```http
POST /api/v1/payments/{payment_id}/refund HTTP/1.1
Host: tmcp.example.com
Authorization: Bearer <TEP_TOKEN>
Content-Type: application/json

{
  "amount": 15000.00,
  "reason": "customer_request",
  "notes": "User requested refund"
}
```

---

## 8. Event System

### 8.1 Custom Matrix Event Types

TMCP defines custom Matrix event types in the `m.tween.*` namespace.

#### 8.1.1 Mini-App Launch Event

```json
{
  "type": "m.tween.miniapp.launch",
  "content": {
    "miniapp_id": "ma_shop_001",
    "launch_source": "chat_bubble",
    "launch_params": {
      "product_id": "prod_123"
    },
    "session_id": "session_xyz789"
  },
  "sender": "@alice:tween.example"
}
```

#### 8.1.2 Payment Events

**Payment Request:**
```json
{
  "type": "m.tween.payment.request",
  "content": {
    "miniapp_id": "ma_shop_001",
    "payment": {
      "payment_id": "pay_abc123",
      "amount": 15000.00,
      "currency": "USD",
      "description": "Order #12345"
    }
  }
}
```

**Payment Completed:**
```json
{
  "type": "m.tween.payment.completed",
  "content": {
    "payment_id": "pay_abc123",
    "txn_id": "txn_def456",
    "status": "completed",
    "amount": 15000.00
  }
}
```

#### 8.1.3 Rich Message Cards

```json
{
  "type": "m.room.message",
  "content": {
    "msgtype": "m.tween.card",
    "miniapp_id": "ma_shop_001",
    "card": {
      "type": "product",
      "title": "Product Name",
      "description": "Product description",
      "image": "mxc://tween.example/image123",
      "price": {
        "amount": 7500.00,
        "currency": "USD"
      },
      "actions": [
        {
          "type": "button",
          "label": "Buy Now",
          "action": "miniapp.open",
          "params": {
            "miniapp_id": "ma_shop_001",
            "path": "/product/123"
          }
        }
      ]
    }
  }
}
```

### 8.2 Event Processing

#### 8.2.1 Application Service Transaction

The Matrix Homeserver sends events to the TMCP Server via the Application Service API:

```http
PUT /_matrix/app/v1/transactions/{txnId} HTTP/1.1
Authorization: Bearer <HS_TOKEN>
Content-Type: application/json

{
  "events": [
    {
      "type": "m.tween.payment.request",
      "content": {...},
      "sender": "@alice:tween.example",
      "room_id": "!chat:tween.example",
      "event_id": "$event_abc123:tween.example"
    }
  ]
}
```

**Response:**
```json
{
  "success": true
}
```

---

## 9. Mini-App Lifecycle

### 9.1 Registration

#### 9.1.1 Registration Request

```http
POST /mini-apps/v1/register HTTP/1.1
Host: tmcp.example.com
Authorization: Bearer <DEVELOPER_TOKEN>
Content-Type: application/json

{
  "name": "Shopping Assistant",
  "short_name": "ShopAssist",
  "description": "AI-powered shopping recommendations",
  "category": "shopping",
  "developer": {
    "company_name": "Example Corp",
    "email": "dev@example.com",
    "website": "https://example.com"
  },
  "technical": {
    "entry_url": "https://miniapp.example.com",
    "redirect_uris": [
      "https://miniapp.example.com/oauth/callback"
    ],
    "webhook_url": "https://api.example.com/webhooks/tween",
    "scopes_requested": [
      "user:read",
      "wallet:pay"
    ]
  },
  "branding": {
    "icon_url": "https://cdn.example.com/icon.png",
    "primary_color": "#FF6B00"
  }
}
```

**Response:**
```json
{
  "miniapp_id": "ma_shop_001",
  "status": "pending_review",
  "credentials": {
    "client_id": "ma_shop_001",
    "client_secret": "secret_abc123",
    "webhook_secret": "whsec_def456"
  },
  "created_at": "2024-12-18T14:30:00Z"
}
```

### 9.2 Lifecycle States

```
DRAFT → SUBMITTED → UNDER_REVIEW → APPROVED → ACTIVE
                         ↓
                    REJECTED
```

---

## 10. Communication Verbs

### 10.1 JSON-RPC 2.0 Bridge

Communication between mini-apps and the host application uses JSON-RPC 2.0 [RFC4627].

**Request Format:**
```json
{
  "jsonrpc": "2.0",
  "method": "tween.wallet.pay",
  "params": {
    "amount": 5000.00,
    "description": "Product purchase"
  },
  "id": 1
}
```

**Response Format:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "payment_id": "pay_abc123",
    "status": "completed"
  },
  "id": 1
}
```

### 10.2 Standard Methods

| Method | Direction | Description |
|--------|-----------|-------------|
| `tween.auth.getUserInfo` | MA → Host | Retrieve user profile |
| `tween.wallet.getBalance` | MA → Host | Get wallet balance |
| `tween.wallet.pay` | MA → Host | Initiate payment |
| `tween.messaging.sendCard` | MA → Host | Send rich message card |
| `tween.storage.get` | MA → Host | Read storage |
| `tween.storage.set` | MA → Host | Write storage |
| `tween.lifecycle.onShow` | Host → MA | Mini-app shown |
| `tween.lifecycle.onHide` | Host → MA | Mini-app hidden |

---

## 11. Security Considerations

### 11.1 Transport Security

- TLS 1.3 REQUIRED for all communications
- Certificate pinning RECOMMENDED for mobile clients
- HSTS with `max-age` >= 31536000 REQUIRED

### 11.2 Authentication Security

**Token Security:**
- Access tokens: 1 hour validity (RECOMMENDED)
- Refresh tokens: 30 days validity (RECOMMENDED)
- Tokens MUST be stored in secure storage (Keychain/KeyStore)
- Tokens MUST NOT be logged

**PKCE Requirements:**
- `code_challenge_method` MUST be `S256`
- Minimum `code_verifier` entropy: 256 bits

### 11.3 Payment Security

**Transaction Signing:**
- All payment authorizations MUST be signed
- Signatures MUST use hardware-backed keys when available
- Signature algorithm: ECDSA P-256 or RSA-2048 minimum

**Idempotency:**
- All payment requests MUST include idempotency keys
- Servers MUST cache idempotency keys for 24 hours minimum

### 11.4 Rate Limiting

| Operation | Limit | Window |
|-----------|-------|--------|
| Token generation | 10 | 1 minute |
| Payment requests | 20 | 1 minute |
| API calls | 100 | 1 minute |
| Webhook delivery | 1000 | 1 hour |

---

## 12. Error Handling

### 12.1 Error Response Format

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Wallet balance too low",
    "details": {
      "required_amount": 15000.00,
      "available_balance": 8000.00
    },
    "timestamp": "2024-12-18T14:30:00Z",
    "request_id": "req_abc123"
  }
}
```

### 12.2 Standard Error Codes

| Code | HTTP Status | Description | Retry |
|------|-------------|-------------|-------|
| `INVALID_TOKEN` | 401 | Invalid or expired token | No |
| `INSUFFICIENT_PERMISSIONS` | 403 | Missing required scope | No |
| `INSUFFICIENT_FUNDS` | 402 | Low wallet balance | No |
| `PAYMENT_FAILED` | 400 | Payment processing error | Yes |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests | Yes |
| `MINIAPP_NOT_FOUND` | 404 | Mini-app not registered | No |
| `INVALID_SIGNATURE` | 401 | Invalid payment signature | No |
| `DUPLICATE_TRANSACTION` | 409 | Idempotency key conflict | No |

---

## 13. Federation Considerations

### 13.1 Closed Federation Model

TMCP deployments typically operate in closed federation mode:

- No external Matrix server federation
- All homeservers within controlled infrastructure
- Shared wallet backend
- Centralized TMCP Server instances

### 13.2 Multi-Server Deployment

For horizontal scaling, multiple instances can be deployed:

```
Load Balancer
     ↓
┌────────────────┐  ┌────────────────┐
│ TMCP Server 1  │  │ TMCP Server 2  │
└────────────────┘  └────────────────┘
         ↓                   ↓
    ┌────────────────────────────┐
    │  Shared Wallet Backend     │
    └────────────────────────────┘
```

Session affinity NOT required due to stateless design.

---

## 14. IANA Considerations

### 14.1 Matrix Event Type Registration

Request registration for the `m.tween.*` namespace:

- `m.tween.miniapp.*`
- `m.tween.wallet.*`
- `m.tween.payment.*`

### 14.2 OAuth Scope Registration

Request registration of TMCP-specific scopes:

- `user:read`
- `user:read:extended`
- `wallet:balance`
- `wallet:pay`
- `messaging:send`

---

## 15. References

### 15.1 Normative References

**[RFC2119]** Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.

**[RFC6749]** Hardt, D., "The OAuth 2.0 Authorization Framework", RFC 6749, October 2012.

**[RFC7636]** Sakimura, N., Bradley, J., and N. Agarwal, "Proof Key for Code Exchange by OAuth Public Clients", RFC 7636, September 2015.

**[RFC7519]** Jones, M., Bradley, J., and N. Sakimura, "JSON Web Token (JWT)", RFC 7519, May 2015.

**[RFC4627]** Crockford, D., "The application/json Media Type for JavaScript Object Notation (JSON)", RFC 4627, July 2006.

**[Matrix-Spec]** The Matrix.org Foundation, "Matrix Specification v1.12", https://spec.matrix.org/v1.12/

**[Matrix-AS]** The Matrix.org Foundation, "Matrix Application Service API", https://spec.matrix.org/v1.12/application-service-api/

### 15.2 Informative References

**[Matrix-CS]** The Matrix.org Foundation, "Matrix Client-Server API", https://spec.matrix.org/v1.12/client-server-api/

**[JSON-RPC]** "JSON-RPC 2.0 Specification", https://www.jsonrpc.org/specification

---

## 16. Appendices

### Appendix A: Complete Protocol Flow Example

**Scenario:** User purchases item from mini-app

1. User opens mini-app from chat
2. Mini-app requests `tween.auth.getUserInfo`
3. Client returns cached user info
4. User adds item to cart, clicks "Buy"
5. Mini-app calls `tween.wallet.pay`
6. Client displays payment confirmation UI
7. User authorizes with biometric
8. Client signs payment with hardware key
9. Client sends signed payment to TMCP Server
10. TMCP Server validates signature
11. TMCP Server coordinates with Wallet Service
12. Wallet Service executes transfer
13. TMCP Server creates Matrix event
14. Homeserver distributes event to room
15. Client renders payment receipt
16. TMCP Server sends webhook to mini-app
17. Mini-app processes order

### Appendix B: SDK Interface Definitions

**TypeScript Interface:**
```typescript
interface TweenSDK {
  auth: {
    getUserInfo(): Promise<UserInfo>;
    requestPermissions(scopes: string[]): Promise<boolean>;
  };
  
  wallet: {
    getBalance(): Promise<WalletBalance>;
    requestPayment(params: PaymentRequest): Promise<PaymentResult>;
  };
  
  messaging: {
    sendCard(params: CardParams): Promise<EventId>;
  };
  
  storage: {
    get(key: string): Promise<string | null>;
    set(key: string, value: string): Promise<void>;
  };
}
```

### Appendix C: Webhook Signature Verification

**Python Example:**
```python
import hmac
import hashlib

def verify_webhook(payload, signature, secret):
    expected = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

---

**End of RFC 9001**
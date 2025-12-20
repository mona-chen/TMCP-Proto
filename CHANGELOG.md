# TMCP Protocol Changelog

All notable changes to Tween Mini-App Communication Protocol (TMCP) will be documented in this file.

## [1.3.0] - 2025-12-20

### Added
- **Section 7.2.3: Recipient Acceptance Protocol**
  - Added two-step confirmation pattern for P2P transfers
  - Defined acceptance flow with 24-hour window
  - Added accept/reject endpoints for recipients
  - Implemented auto-expiry with refund mechanism
  - Added Matrix events for pending acceptance and status updates

- **Section 7.5: Group Gift Distribution Protocol**
  - Added culturally relevant gamified gifting alternative
  - Defined individual and group gift creation flows
  - Implemented random and equal distribution algorithms
  - Added gift opening protocol with leaderboard
  - Created Matrix events for gift creation and opening

- **Section 6.4: External Account Linking & Funding**
  - Added support for linking bank accounts and cards
  - Implemented micro-deposit verification for accounts
  - Added funding protocol with status tracking
  - Defined external account management endpoints

- **Section 6.5: Wallet Withdrawal Protocol**
  - Added withdrawal initiation with approval workflow
  - Implemented tiered withdrawal limits by verification level
  - Added withdrawal status tracking and completion
  - Defined security requirements for large withdrawals

- **Section 11.5: Compliance Framework**
  - Added flexible compliance framework for regulatory requirements
  - Implemented tiered verification levels with corresponding limits
  - Added transaction monitoring and risk scoring
  - Created regulatory reporting capabilities
  - Defined sanctions screening protocol

- **Updated Section 7.1: Payment State Machine**
  - Added P2P transfer states with recipient acceptance
  - Added group gift states (created → active → opened)
  - Updated state transitions to reflect new flows

- **Updated Section 8.1: New Matrix Event Types**
  - Added m.tween.wallet.p2p.status for transfer updates
  - Added m.tween.gift and m.tween.gift.opened for group gifts
  - Added m.tween.wallet.invite for wallet invitations

- **Updated Section 10.2: New JSON-RPC Methods**
  - Added tween.wallet.sendGift for creating group gifts
  - Added tween.wallet.openGift for opening received gifts
  - Added tween.wallet.acceptTransfer and tween.wallet.rejectTransfer

- **Updated Section 12.2: Additional Error Codes**
  - Added RECIPIENT_NO_WALLET, RECIPIENT_ACCEPTANCE_REQUIRED
  - Added TRANSFER_EXPIRED, GIFT_EXPIRED
  - Added EXTERNAL_ACCOUNT_PENDING, WITHDRAWAL_LIMIT_EXCEEDED

- **Updated Section 6.2.1: Verification Tiers**
  - Added verification tier information to balance responses
  - Defined tier requirements and corresponding limits
  - Added upgrade path for enhanced features

## [1.2.0] - 2025-12-19

### Added
- **Section 7.4: Multi-Factor Authentication for Payments**
  - Added MFA challenge-response mechanism for payment authorization
  - Defined standard MFA method types (transaction_pin, biometric, totp)
  - Added device registration protocol for biometric MFA
  - Updated payment state machine to include MFA_REQUIRED state
  - Added Wallet Service MFA interface requirements

- **Section 10.3: Mini-App Storage System**
  - Added key-value storage protocol for mini-apps
  - Defined storage quotas (10MB per user/app, 1MB per key, 1000 keys)
  - Added offline storage support with conflict resolution
  - Implemented batch operations for efficiency
  - Added storage scopes (storage:read, storage:write) with auto-approval

- **Section 8.1.4: App Lifecycle Events**
  - Added Matrix events for app installation, updates, and uninstallation
  - Defined event formats for lifecycle tracking

- **Section 16: Official and Preinstalled Mini-Apps**
  - Added mini-app classification system (official, verified, community, beta)
  - Defined preinstallation manifest format and loading process
  - Added internal URL scheme (tween-internal://) for official apps
  - Implemented mini-app store protocol with discovery and installation
  - Added app ranking algorithm and trending detection
  - Defined privileged scopes for official apps
  - Added update management protocol with verification requirements
  - Modified OAuth flow for official apps to use PKCE with pre-approved basic scopes

- **Section 11.4.1: Rate Limiting Implementation Guidance**
  - Added required rate limit headers (X-RateLimit-*)
  - Defined token bucket/sliding window algorithm recommendation
  - Added 429 status code with retry_after header

- **Section 12.2: Additional Error Codes**
  - Added MFA_REQUIRED, MFA_LOCKED, INVALID_MFA_CREDENTIALS
  - Added STORAGE_QUOTA_EXCEEDED, APP_NOT_REMOVABLE, APP_NOT_FOUND, DEVICE_NOT_REGISTERED

### Security Enhancements
- Biometric attestation for MFA using device-bound keys
- Enhanced token security for official apps
- Audit logging requirements for privileged operations

### Implementation Guidance
- Added Section 16.10: OAuth Server Implementation with Keycloak
  - Comprehensive Keycloak realm configuration
  - Client registration process for mini-apps
  - Token service configuration with JWT signing
  - MFA service integration details

### Documentation
- Added Appendix D: Protocol Change Log for tracking evolution
- Updated Table of Contents to reflect new Section numbering

---

## [Unreleased] - Future

### Planned
- None currently

---

## Format

This changelog follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) format with modifications for TMCP protocol's specific needs.

### Types of Changes
- `Added` for new features
- `Changed` for modifications to existing features
- `Deprecated` for removed features
- `Security` for security-related changes
- `Documentation` for documentation updates
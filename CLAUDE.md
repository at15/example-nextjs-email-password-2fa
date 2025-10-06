# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Next.js example application demonstrating email/password authentication with two-factor authentication (2FA), built with SQLite. It's part of the Lucia Auth project's example collection.

**Key Features:**
- Email/password authentication with HaveIBeenPwned password checking
- Email verification via verification codes
- TOTP-based 2FA with recovery codes
- Password reset flow
- Login throttling and global rate limiting
- Session management with automatic renewal

## Setup & Commands

### Initial Setup
1. Create and initialize database:
```bash
sqlite3 sqlite.db < setup.sql
```

2. Generate and set encryption key in `.env`:
```bash
openssl rand --base64 16
# Add to .env: ENCRYPTION_KEY="<generated_key>"
```

### Common Commands
```bash
pnpm i              # Install dependencies
pnpm dev            # Run development server (localhost:3000)
pnpm build          # Build for production
pnpm start          # Start production server
pnpm lint           # Run ESLint
pnpm format         # Format code with Prettier
```

## Architecture

### Directory Structure

- **`app/`** - Next.js App Router pages and server actions
  - Route-specific `actions.ts` files contain server actions for that feature
  - Route-specific `components.tsx` files contain client components
  - Each auth flow (login, signup, 2fa, password reset, etc.) has its own directory
  - `app/actions.ts` - Global server actions (e.g., logout)

- **`lib/server/`** - Server-side core modules (do not import in client components)
  - `session.ts` - Session token management, validation, and cookie handling
  - `user.ts` - User CRUD operations
  - `db.ts` - SQLite database connection and adapter
  - `rate-limit.ts` - Rate limiting implementations (RefillingTokenBucket, ExpiringTokenBucket, Throttler)
  - `password.ts` - Password hashing with Argon2 and HaveIBeenPwned API integration
  - `email-verification.ts` - Email verification request management
  - `password-reset.ts` - Password reset session management
  - `2fa.ts` - TOTP and recovery code handling
  - `encryption.ts` - AES-128-GCM encryption/decryption for sensitive data (TOTP keys, recovery codes)
  - `email.ts` - Email sending (currently just console logging)
  - `request.ts` - Global rate limiting middleware
  - `utils.ts` - Utility functions (recovery code generation, etc.)

- **`middleware.ts`** - Next.js middleware for CSRF protection and session cookie renewal on GET requests

### Database Schema

See `setup.sql` for complete schema. Key tables:
- `user` - Stores email, username, password_hash, email_verified flag, encrypted totp_key and recovery_code
- `session` - User sessions with expires_at and two_factor_verified flag
- `email_verification_request` - Email verification codes
- `password_reset_session` - Password reset sessions with verification state

### Authentication Flow

1. **Session Management**: Token-based sessions using SHA-256 hashed tokens
   - Session tokens stored in httpOnly cookies
   - Auto-renewal when within 15 days of expiration
   - 30-day session lifetime
   - Sessions tracked with `two_factor_verified` flag for 2FA enforcement

2. **Two-Factor Authentication**:
   - Users with `totp_key` must verify 2FA after login to access protected routes
   - Session created with `twoFactorVerified: false` initially
   - Recovery codes can disable 2FA and regenerate a new recovery code
   - Rate limited via `ExpiringTokenBucket` (5 TOTP attempts/30min, 3 recovery code attempts/hour)

3. **Rate Limiting**: In-memory JavaScript `Map` storage (NOT production-ready)
   - Global POST rate limiting in `middleware.ts`
   - Per-feature rate limiting using token bucket and throttling algorithms
   - Login throttling with exponential backoff

### Security Patterns

- **Encryption**: Sensitive data (TOTP keys, recovery codes) encrypted with AES-128-GCM using `ENCRYPTION_KEY`
- **CSRF Protection**: Origin header validation in middleware for non-GET requests
- **Password Security**: Argon2 hashing + HaveIBeenPwned breach checking
- **Client IP**: Relies on `X-Forwarded-For` header (needs hardening for production)

### Important Implementation Notes

- Email sending is stubbed out (logs to console only)
- Rate limiting uses in-memory `Map` - will not scale or persist across restarts
- Some queries may need transactions with row locking for MySQL/Postgres to avoid race conditions
- User enumeration is considered acceptable by design
- Error handling is minimal (not production-ready)
- Significant code duplication in 2FA flows for simplicity
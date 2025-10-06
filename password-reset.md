# Password Reset Flow Documentation

## Database Schema

```sql
-- Users table
CREATE TABLE user (
    id INTEGER NOT NULL PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    username TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    email_verified INTEGER NOT NULL DEFAULT 0,
    totp_key BLOB,                          -- Encrypted TOTP secret
    recovery_code BLOB NOT NULL             -- Encrypted recovery code
);
CREATE INDEX email_index ON user(email);

-- Active sessions
CREATE TABLE session (
    id TEXT NOT NULL PRIMARY KEY,          -- SHA-256 hash of token
    user_id INTEGER NOT NULL REFERENCES user(id),
    expires_at INTEGER NOT NULL,           -- Unix timestamp
    two_factor_verified INTEGER NOT NULL DEFAULT 0
);

-- Email verification requests
CREATE TABLE email_verification_request (
    id TEXT NOT NULL PRIMARY KEY,          -- SHA-256 hash of token
    user_id INTEGER NOT NULL REFERENCES user(id),
    email TEXT NOT NULL,
    code TEXT NOT NULL,                    -- OTP code
    expires_at INTEGER NOT NULL            -- Unix timestamp
);

-- Password reset sessions
CREATE TABLE password_reset_session (
    id TEXT NOT NULL PRIMARY KEY,          -- SHA-256 hash of token
    user_id INTEGER NOT NULL REFERENCES user(id),
    email TEXT NOT NULL,
    code TEXT NOT NULL,                    -- OTP code
    expires_at INTEGER NOT NULL,           -- Unix timestamp
    email_verified INTEGER NOT NULL DEFAULT 0,
    two_factor_verified INTEGER NOT NULL DEFAULT 0
);
```

## Password Reset Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     PASSWORD RESET FLOW                                  │
└─────────────────────────────────────────────────────────────────────────┘

Step 1: Request Reset
┌──────────────────────────────────────────────────────────────────────┐
│ User submits email at /forgot-password                                │
│ app/forgot-password/actions.ts:forgotPasswordAction()                 │
└───────────────────┬──────────────────────────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────────────────────────┐
    │ Rate limit checks:                                │
    │ - 3 emails per IP per 60s (RefillingTokenBucket)  │
    │ - 3 emails per user per 60s                       │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ SQL: SELECT id, email, username, email_verified,  │
    │      IIF(totp_key IS NOT NULL, 1, 0)              │
    │      FROM user WHERE email = ?                    │
    │ lib/server/user.ts:85-100                         │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ SQL: DELETE FROM password_reset_session           │
    │      WHERE user_id = ?                            │
    │ lib/server/password-reset.ts:73-74                │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ Generate session token (20 random bytes)          │
    │ Generate OTP code (8 digits)                      │
    │                                                   │
    │ SQL: INSERT INTO password_reset_session           │
    │      (id, user_id, email, code, expires_at)       │
    │      VALUES (SHA256(token), ?, ?, ?, ?)           │
    │ lib/server/password-reset.ts:9-27                 │
    │                                                   │
    │ Expiry: 10 minutes                                │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ Send email with OTP code (console.log)            │
    │ Set cookie: password_reset_session=<token>        │
    │ Redirect to /reset-password/verify-email          │
    └───────────────────────────────────────────────────┘


Step 2: Verify Email
┌──────────────────────────────────────────────────────────────────────┐
│ User enters code at /reset-password/verify-email                      │
│ app/reset-password/verify-email/actions.ts                            │
│   :verifyPasswordResetEmailAction()                                   │
└───────────────────┬──────────────────────────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────────────────────────┐
    │ Read password_reset_session cookie                │
    │                                                   │
    │ SQL: SELECT prs.*, user.*                         │
    │      FROM password_reset_session prs              │
    │      INNER JOIN user ON user.id = prs.user_id     │
    │      WHERE prs.id = SHA256(token)                 │
    │ lib/server/password-reset.ts:30-63                │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ Rate limit: 5 attempts per user per 30min         │
    │ (ExpiringTokenBucket)                             │
    │                                                   │
    │ Compare submitted code with session.code          │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ SQL: UPDATE password_reset_session                │
    │      SET email_verified = 1 WHERE id = ?          │
    │ lib/server/password-reset.ts:65-66                │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ SQL: UPDATE user SET email_verified = 1           │
    │      WHERE id = ? AND email = ?                   │
    │ lib/server/user.ts:40-42                          │
    │                                                   │
    │ Redirect to /reset-password/2fa                   │
    └───────────────────────────────────────────────────┘


Step 3: Verify 2FA (if user has totp_key set)
┌──────────────────────────────────────────────────────────────────────┐
│ User enters TOTP or recovery code at /reset-password/2fa              │
│ app/reset-password/2fa/actions.ts                                     │
└───────────────────┬──────────────────────────────────────────────────┘
                    │
            ┌───────┴──────────┐
            │                  │
            ▼                  ▼
    ┌──────────────┐   ┌──────────────────┐
    │ TOTP Option  │   │ Recovery Code    │
    └──────┬───────┘   └────────┬─────────┘
           │                    │
           ▼                    ▼
    ┌──────────────┐   ┌───────────────────────────┐
    │ Rate limit:  │   │ Rate limit:                │
    │ 5 per 30min  │   │ 3 per 60min                │
    └──────┬───────┘   └────────┬──────────────────┘
           │                    │
           ▼                    ▼
    ┌──────────────────┐   ┌─────────────────────────┐
    │ SQL: SELECT      │   │ SQL: SELECT recovery_code│
    │   totp_key       │   │   FROM user WHERE id = ? │
    │   FROM user      │   │ lib/server/2fa.ts:11     │
    │   WHERE id = ?   │   │                          │
    │ user.ts:61-71    │   │ Decrypt & compare        │
    │                  │   │                          │
    │ Verify TOTP code │   │ If valid:                │
    └──────┬───────────┘   │ SQL: UPDATE session      │
           │               │   SET two_factor_verified│
           │               │     = 0 WHERE user_id = ?│
           │               │                          │
           │               │ SQL: UPDATE user SET     │
           │               │   totp_key = NULL,       │
           │               │   recovery_code = <new>  │
           │               │   WHERE id = ?           │
           │               │ lib/server/2fa.ts:23-29  │
           └───────────────┴─────────┬────────────────┘
                                     │
                                     ▼
                    ┌────────────────────────────────┐
                    │ SQL: UPDATE password_reset_    │
                    │   session SET                  │
                    │   two_factor_verified = 1      │
                    │   WHERE id = ?                 │
                    │ lib/server/password-reset.     │
                    │   ts:69-70                     │
                    │                                │
                    │ Redirect to /reset-password    │
                    └────────────────────────────────┘


Step 4: Set New Password
┌──────────────────────────────────────────────────────────────────────┐
│ User enters new password at /reset-password                           │
│ app/reset-password/actions.ts:resetPasswordAction()                   │
└───────────────────┬──────────────────────────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────────────────────────┐
    │ Validate password_reset_session from cookie       │
    │ Check: emailVerified = true                       │
    │ Check: if user.registered2FA then                 │
    │        twoFactorVerified = true                   │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ Verify password strength (HaveIBeenPwned API)     │
    │ lib/server/password.ts                            │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ SQL: DELETE FROM password_reset_session           │
    │      WHERE user_id = ?                            │
    │ lib/server/password-reset.ts:73                   │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ SQL: DELETE FROM session WHERE user_id = ?        │
    │ lib/server/session.ts:63                          │
    │ (Invalidates all existing user sessions)          │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ Hash new password with Argon2                     │
    │                                                   │
    │ SQL: UPDATE user SET password_hash = ?            │
    │      WHERE id = ?                                 │
    │ lib/server/user.ts:31-33                          │
    └───────────────────┬───────────────────────────────┘
                        │
                        ▼
    ┌───────────────────────────────────────────────────┐
    │ Generate new session token                        │
    │                                                   │
    │ SQL: INSERT INTO session                          │
    │      (id, user_id, expires_at,                    │
    │       two_factor_verified)                        │
    │      VALUES (SHA256(token), ?, ?,                 │
    │              <from_reset_session>)                │
    │ lib/server/session.ts:94-108                      │
    │                                                   │
    │ Set session cookie, expires in 30 days            │
    │ Delete password_reset_session cookie              │
    │ Redirect to /                                     │
    └───────────────────────────────────────────────────┘
```

## Key Files Involved

1. [app/forgot-password/actions.ts](app/forgot-password/actions.ts) - Initial reset request
2. [app/reset-password/verify-email/actions.ts](app/reset-password/verify-email/actions.ts) - Email verification
3. [app/reset-password/2fa/actions.ts](app/reset-password/2fa/actions.ts) - 2FA verification (TOTP or recovery code)
4. [app/reset-password/actions.ts](app/reset-password/actions.ts) - Final password update
5. [lib/server/password-reset.ts](lib/server/password-reset.ts) - Password reset session management
6. [lib/server/user.ts](lib/server/user.ts) - User data operations
7. [lib/server/session.ts](lib/server/session.ts) - Session management
8. [lib/server/2fa.ts](lib/server/2fa.ts) - 2FA operations
9. [lib/server/password.ts](lib/server/password.ts) - Password hashing/verification

## All SQL Queries in Password Reset Flow

### Step 1: Request Reset

1. **Get user by email**
   ```sql
   SELECT id, email, username, email_verified, IIF(totp_key IS NOT NULL, 1, 0)
   FROM user
   WHERE email = ?
   ```
   Location: [lib/server/user.ts:85-100](lib/server/user.ts#L85-L100)

2. **Delete old reset sessions**
   ```sql
   DELETE FROM password_reset_session
   WHERE user_id = ?
   ```
   Location: [lib/server/password-reset.ts:73-74](lib/server/password-reset.ts#L73-L74)

3. **Create reset session**
   ```sql
   INSERT INTO password_reset_session (id, user_id, email, code, expires_at)
   VALUES (?, ?, ?, ?, ?)
   ```
   Location: [lib/server/password-reset.ts:20-26](lib/server/password-reset.ts#L20-L26)
   - `id` = SHA-256 hash of random 20-byte token
   - `code` = 8-digit OTP
   - `expires_at` = Unix timestamp (current time + 10 minutes)

### Step 2: Verify Email

4. **Validate reset session**
   ```sql
   SELECT password_reset_session.id, password_reset_session.user_id,
          password_reset_session.email, password_reset_session.code,
          password_reset_session.expires_at, password_reset_session.email_verified,
          password_reset_session.two_factor_verified,
          user.id, user.email, user.username, user.email_verified,
          IIF(user.totp_key IS NOT NULL, 1, 0)
   FROM password_reset_session
   INNER JOIN user ON user.id = password_reset_session.user_id
   WHERE password_reset_session.id = ?
   ```
   Location: [lib/server/password-reset.ts:32-37](lib/server/password-reset.ts#L32-L37)

5. **Mark email verified in reset session**
   ```sql
   UPDATE password_reset_session
   SET email_verified = 1
   WHERE id = ?
   ```
   Location: [lib/server/password-reset.ts:65-66](lib/server/password-reset.ts#L65-L66)

6. **Mark user email as verified**
   ```sql
   UPDATE user
   SET email_verified = 1
   WHERE id = ? AND email = ?
   ```
   Location: [lib/server/user.ts:40-42](lib/server/user.ts#L40-L42)

### Step 3: Verify 2FA (if applicable)

7. **Get TOTP key**
   ```sql
   SELECT totp_key
   FROM user
   WHERE id = ?
   ```
   Location: [lib/server/user.ts:61-71](lib/server/user.ts#L61-L71)
   - Returns encrypted BLOB, decrypted using AES-128-GCM

8. **Get recovery code**
   ```sql
   SELECT recovery_code
   FROM user
   WHERE id = ?
   ```
   Location: [lib/server/2fa.ts:11](lib/server/2fa.ts#L11)
   - Returns encrypted BLOB, decrypted using AES-128-GCM

9. **Reset 2FA with recovery code** (if recovery code used)
   ```sql
   -- First, invalidate all user sessions
   UPDATE session
   SET two_factor_verified = 0
   WHERE user_id = ?

   -- Then disable 2FA and rotate recovery code
   UPDATE user
   SET recovery_code = ?, totp_key = NULL
   WHERE id = ? AND recovery_code = ?
   ```
   Location: [lib/server/2fa.ts:23-29](lib/server/2fa.ts#L23-L29)

10. **Mark 2FA verified in reset session**
    ```sql
    UPDATE password_reset_session
    SET two_factor_verified = 1
    WHERE id = ?
    ```
    Location: [lib/server/password-reset.ts:69-70](lib/server/password-reset.ts#L69-L70)

### Step 4: Set New Password

11. **Delete all password reset sessions**
    ```sql
    DELETE FROM password_reset_session
    WHERE user_id = ?
    ```
    Location: [lib/server/password-reset.ts:73](lib/server/password-reset.ts#L73)

12. **Invalidate all user sessions**
    ```sql
    DELETE FROM session
    WHERE user_id = ?
    ```
    Location: [lib/server/session.ts:63-64](lib/server/session.ts#L63-L64)

13. **Update password hash**
    ```sql
    UPDATE user
    SET password_hash = ?
    WHERE id = ?
    ```
    Location: [lib/server/user.ts:31-33](lib/server/user.ts#L31-L33)
    - Password hashed using Argon2

14. **Create new session**
    ```sql
    INSERT INTO session (id, user_id, expires_at, two_factor_verified)
    VALUES (?, ?, ?, ?)
    ```
    Location: [lib/server/session.ts:102-107](lib/server/session.ts#L102-L107)
    - `id` = SHA-256 hash of random 20-byte token
    - `expires_at` = Unix timestamp (current time + 30 days)
    - `two_factor_verified` = inherited from password reset session

## Security Features

### Rate Limiting

- **IP-based**: 3 reset emails per IP per 60 seconds (RefillingTokenBucket)
- **User-based**: 3 reset emails per user per 60 seconds (RefillingTokenBucket)
- **Email verification**: 5 code attempts per user per 30 minutes (ExpiringTokenBucket)
- **TOTP verification**: 5 code attempts per user per 30 minutes (ExpiringTokenBucket)
- **Recovery code verification**: 3 code attempts per user per 60 minutes (ExpiringTokenBucket)

### Session Security

- Password reset sessions expire after 10 minutes
- Sessions use SHA-256 hashed tokens stored in httpOnly cookies
- All existing user sessions are invalidated when password is changed
- Two-factor verification state is tracked throughout the reset flow

### Password Security

- Passwords are hashed using Argon2
- Password strength validated against HaveIBeenPwned API
- Cannot reuse existing password (though not explicitly checked in code)

### 2FA Handling

- Users with TOTP enabled must verify 2FA during password reset
- Recovery codes can disable TOTP and generate a new recovery code
- After using recovery code, all sessions are marked as not 2FA-verified
- Rate limiting prevents brute-force attacks on TOTP/recovery codes

### Data Encryption

- TOTP keys stored as AES-128-GCM encrypted BLOBs
- Recovery codes stored as AES-128-GCM encrypted BLOBs
- Encryption key loaded from `ENCRYPTION_KEY` environment variable

## Important Notes

- Email sending is stubbed (logs to console only) - see [lib/server/password-reset.ts:109-111](lib/server/password-reset.ts#L109-L111)
- Rate limiting uses in-memory `Map` - not suitable for production (no persistence across restarts, doesn't scale horizontally)
- Relies on `X-Forwarded-For` header for client IP - needs hardening for production
- User enumeration is possible (design decision, not a bug)
- Some queries may need transactions with `SELECT FOR UPDATE` when using MySQL/Postgres to avoid race conditions

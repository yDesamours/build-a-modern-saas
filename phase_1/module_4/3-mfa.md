# Module 4 - Day 5: Multi-Factor Authentication (MFA)

**Part of:** Module 4 - Authentication & Authorization  
**Duration:** Day 5 of 7

---

## 📚 Learning Objectives

By the end of this day, you will:

- Understand different MFA methods (TOTP, SMS, Email, Hardware tokens)
- Implement TOTP-based authentication (Google Authenticator, Authy)
- Set up SMS-based authentication
- Create backup/recovery codes
- Implement "remember this device" functionality
- Handle MFA enrollment and verification flows
- Build recovery mechanisms for lost MFA devices
- Implement MFA across Java, Node.js, and Go

---

## 🎯 Understanding Multi-Factor Authentication

### What is MFA?

**Multi-Factor Authentication** requires users to provide two or more verification factors to gain access.

**Three Authentication Factors:**

1. **Something you know** - Password, PIN
2. **Something you have** - Phone, hardware token, authenticator app
3. **Something you are** - Fingerprint, face recognition

**Common MFA Methods:**

| Method                        | Security   | UX        | Cost        | Best For                  |
| ----------------------------- | ---------- | --------- | ----------- | ------------------------- |
| **TOTP** (Authenticator apps) | High       | Good      | Free        | Most SaaS apps            |
| **SMS**                       | Medium     | Excellent | Low-Medium  | Consumer apps             |
| **Email**                     | Low-Medium | Good      | Free        | Low-security apps         |
| **Hardware tokens** (YubiKey) | Very High  | Good      | High        | Enterprise, high-security |
| **Push notifications**        | High       | Excellent | Medium      | Mobile-first apps         |
| **Biometrics**                | High       | Excellent | Medium-High | Mobile apps               |

---

## 📖 Section 1: TOTP (Time-based One-Time Password)

### 5.1 How TOTP Works

**TOTP Algorithm:**

```
1. Generate secret key (shared between server and authenticator app)
2. Current time divided by 30 seconds = time step
3. HMAC-SHA1(secret, time_step) = hash
4. Extract 6 digits from hash = OTP code
5. Code changes every 30 seconds
```

**Flow:**

```
Setup Phase:
1. Server generates secret key
2. Server displays QR code (contains secret)
3. User scans QR code with authenticator app
4. App stores secret and generates codes

Login Phase:
1. User enters username + password
2. Server requests MFA code
3. User opens authenticator app
4. User enters 6-digit code
5. Server validates code against secret
6. Access granted if valid
```

---

### 5.2 Database Schema for MFA

```sql
-- MFA secrets table
CREATE TABLE mfa_secrets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  secret VARCHAR(32) NOT NULL, -- Base32 encoded secret
  enabled BOOLEAN DEFAULT FALSE,
  verified_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Backup codes
CREATE TABLE mfa_backup_codes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  code VARCHAR(10) NOT NULL, -- 8-10 character code
  used BOOLEAN DEFAULT FALSE,
  used_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(user_id, code)
);

-- Trusted devices (remember this device)
CREATE TABLE trusted_devices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  device_token VARCHAR(64) UNIQUE NOT NULL,
  device_name VARCHAR(255),
  device_type VARCHAR(50), -- browser, mobile, desktop
  ip_address VARCHAR(45),
  user_agent TEXT,
  last_used TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  INDEX idx_device_token (device_token),
  INDEX idx_user_devices (user_id, expires_at)
);

-- MFA attempts log (rate limiting)
CREATE TABLE mfa_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  success BOOLEAN NOT NULL,
  ip_address VARCHAR(45),
  attempted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  INDEX idx_user_attempts (user_id, attempted_at)
);

-- Update users table
ALTER TABLE users ADD COLUMN mfa_enabled BOOLEAN DEFAULT FALSE;
ALTER TABLE users ADD COLUMN mfa_required BOOLEAN DEFAULT FALSE; -- Force MFA for certain users
```

---

### 5.3 Node.js/TypeScript TOTP Implementation

**Dependencies:**

```bash
npm install speakeasy qrcode
npm install --save-dev @types/speakeasy @types/qrcode
```

**MFA Service:**

```typescript
// src/services/mfa.service.ts
import speakeasy from "speakeasy";
import QRCode from "qrcode";
import crypto from "crypto";
import { pool } from "../config/database";

export class MFAService {
  /**
   * Generate MFA secret for user
   */
  static generateSecret(userEmail: string, appName: string = "My SaaS App") {
    const secret = speakeasy.generateSecret({
      name: `${appName} (${userEmail})`,
      issuer: appName,
      length: 32,
    });

    return {
      secret: secret.base32,
      otpauthUrl: secret.otpauth_url!,
    };
  }

  /**
   * Generate QR code for secret
   */
  static async generateQRCode(otpauthUrl: string): Promise<string> {
    try {
      return await QRCode.toDataURL(otpauthUrl);
    } catch (error) {
      throw new Error("Failed to generate QR code");
    }
  }

  /**
   * Verify TOTP code
   */
  static verifyToken(secret: string, token: string): boolean {
    return speakeasy.totp.verify({
      secret: secret,
      encoding: "base32",
      token: token,
      window: 2, // Allow 1 step before/after for clock drift (60 seconds total)
    });
  }

  /**
   * Save MFA secret for user (not yet enabled)
   */
  static async saveMFASecret(userId: string, secret: string): Promise<void> {
    const query = `
      INSERT INTO mfa_secrets (user_id, secret, enabled)
      VALUES ($1, $2, FALSE)
      ON CONFLICT (user_id) 
      DO UPDATE SET secret = $2, enabled = FALSE, updated_at = CURRENT_TIMESTAMP
    `;

    await pool.query(query, [userId, secret]);
  }

  /**
   * Enable MFA after verification
   */
  static async enableMFA(userId: string): Promise<void> {
    const client = await pool.connect();

    try {
      await client.query("BEGIN");

      // Enable MFA secret
      await client.query(
        `UPDATE mfa_secrets 
         SET enabled = TRUE, verified_at = CURRENT_TIMESTAMP 
         WHERE user_id = $1`,
        [userId]
      );

      // Update user
      await client.query("UPDATE users SET mfa_enabled = TRUE WHERE id = $1", [
        userId,
      ]);

      await client.query("COMMIT");
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Disable MFA
   */
  static async disableMFA(userId: string): Promise<void> {
    const client = await pool.connect();

    try {
      await client.query("BEGIN");

      // Delete MFA secret
      await client.query("DELETE FROM mfa_secrets WHERE user_id = $1", [
        userId,
      ]);

      // Delete backup codes
      await client.query("DELETE FROM mfa_backup_codes WHERE user_id = $1", [
        userId,
      ]);

      // Update user
      await client.query("UPDATE users SET mfa_enabled = FALSE WHERE id = $1", [
        userId,
      ]);

      await client.query("COMMIT");
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Get user's MFA secret
   */
  static async getMFASecret(userId: string): Promise<string | null> {
    const query =
      "SELECT secret FROM mfa_secrets WHERE user_id = $1 AND enabled = TRUE";
    const result = await pool.query(query, [userId]);

    return result.rows.length > 0 ? result.rows[0].secret : null;
  }

  /**
   * Check if user has MFA enabled
   */
  static async isMFAEnabled(userId: string): Promise<boolean> {
    const query = "SELECT mfa_enabled FROM users WHERE id = $1";
    const result = await pool.query(query, [userId]);

    return result.rows.length > 0 ? result.rows[0].mfa_enabled : false;
  }

  /**
   * Generate backup codes
   */
  static generateBackupCodes(count: number = 10): string[] {
    const codes: string[] = [];

    for (let i = 0; i < count; i++) {
      // Generate 8-character alphanumeric code
      const code = crypto.randomBytes(4).toString("hex").toUpperCase();
      codes.push(code);
    }

    return codes;
  }

  /**
   * Save backup codes
   */
  static async saveBackupCodes(userId: string, codes: string[]): Promise<void> {
    // Delete old codes
    await pool.query("DELETE FROM mfa_backup_codes WHERE user_id = $1", [
      userId,
    ]);

    // Insert new codes
    const values = codes.map((code) => `('${userId}', '${code}')`).join(",");
    const query = `
      INSERT INTO mfa_backup_codes (user_id, code)
      VALUES ${values}
    `;

    await pool.query(query);
  }

  /**
   * Verify backup code
   */
  static async verifyBackupCode(
    userId: string,
    code: string
  ): Promise<boolean> {
    const query = `
      UPDATE mfa_backup_codes
      SET used = TRUE, used_at = CURRENT_TIMESTAMP
      WHERE user_id = $1 AND code = $2 AND used = FALSE
      RETURNING id
    `;

    const result = await pool.query(query, [userId, code.toUpperCase()]);
    return result.rows.length > 0;
  }

  /**
   * Get remaining backup codes count
   */
  static async getRemainingBackupCodesCount(userId: string): Promise<number> {
    const query = `
      SELECT COUNT(*) as count
      FROM mfa_backup_codes
      WHERE user_id = $1 AND used = FALSE
    `;

    const result = await pool.query(query, [userId]);
    return parseInt(result.rows[0].count);
  }

  /**
   * Log MFA attempt
   */
  static async logAttempt(
    userId: string,
    success: boolean,
    ipAddress: string
  ): Promise<void> {
    const query = `
      INSERT INTO mfa_attempts (user_id, success, ip_address)
      VALUES ($1, $2, $3)
    `;

    await pool.query(query, [userId, success, ipAddress]);
  }

  /**
   * Check rate limiting (max 5 failed attempts in 15 minutes)
   */
  static async checkRateLimit(userId: string): Promise<boolean> {
    const query = `
      SELECT COUNT(*) as count
      FROM mfa_attempts
      WHERE user_id = $1 
        AND success = FALSE 
        AND attempted_at > CURRENT_TIMESTAMP - INTERVAL '15 minutes'
    `;

    const result = await pool.query(query, [userId]);
    const failedAttempts = parseInt(result.rows[0].count);

    return failedAttempts < 5;
  }
}
```

**MFA Controller:**

```typescript
// src/controllers/mfa.controller.ts
import { Request, Response } from "express";
import { MFAService } from "../services/mfa.service";
import { User } from "../models/user.model";

export class MFAController {
  /**
   * Step 1: Setup MFA - Generate secret and QR code
   */
  static async setupMFA(req: Request, res: Response) {
    try {
      const userId = req.user!.userId;
      const user = await User.findById(userId);

      if (!user) {
        return res.status(404).json({ error: "User not found" });
      }

      // Check if MFA already enabled
      if (await MFAService.isMFAEnabled(userId)) {
        return res.status(400).json({ error: "MFA already enabled" });
      }

      // Generate secret
      const { secret, otpauthUrl } = MFAService.generateSecret(user.email);

      // Generate QR code
      const qrCode = await MFAService.generateQRCode(otpauthUrl);

      // Save secret (not yet enabled)
      await MFAService.saveMFASecret(userId, secret);

      res.json({
        message: "MFA setup initiated",
        secret, // Show this as manual entry option
        qrCode, // Data URL for QR code image
      });
    } catch (error) {
      console.error("MFA setup error:", error);
      res.status(500).json({ error: "MFA setup failed" });
    }
  }

  /**
   * Step 2: Verify and enable MFA
   */
  static async verifyAndEnableMFA(req: Request, res: Response) {
    try {
      const userId = req.user!.userId;
      const { token } = req.body;

      if (!token || token.length !== 6) {
        return res.status(400).json({ error: "Invalid token format" });
      }

      // Get secret
      const result = await pool.query(
        "SELECT secret FROM mfa_secrets WHERE user_id = $1",
        [userId]
      );

      if (result.rows.length === 0) {
        return res
          .status(400)
          .json({ error: "MFA not set up. Call /setup first" });
      }

      const secret = result.rows[0].secret;

      // Verify token
      const isValid = MFAService.verifyToken(secret, token);

      if (!isValid) {
        return res.status(400).json({ error: "Invalid verification code" });
      }

      // Enable MFA
      await MFAService.enableMFA(userId);

      // Generate backup codes
      const backupCodes = MFAService.generateBackupCodes(10);
      await MFAService.saveBackupCodes(userId, backupCodes);

      res.json({
        message: "MFA enabled successfully",
        backupCodes, // Show once, user must save them
      });
    } catch (error) {
      console.error("MFA verification error:", error);
      res.status(500).json({ error: "MFA verification failed" });
    }
  }

  /**
   * Disable MFA
   */
  static async disableMFA(req: Request, res: Response) {
    try {
      const userId = req.user!.userId;
      const { password, token } = req.body;

      // Verify password
      const user = await User.findById(userId);
      const isValidPassword = await user!.verifyPassword(password);

      if (!isValidPassword) {
        return res.status(401).json({ error: "Invalid password" });
      }

      // Verify MFA token
      const secret = await MFAService.getMFASecret(userId);
      if (secret) {
        const isValidToken = MFAService.verifyToken(secret, token);
        if (!isValidToken) {
          return res.status(400).json({ error: "Invalid MFA code" });
        }
      }

      // Disable MFA
      await MFAService.disableMFA(userId);

      res.json({ message: "MFA disabled successfully" });
    } catch (error) {
      console.error("MFA disable error:", error);
      res.status(500).json({ error: "Failed to disable MFA" });
    }
  }

  /**
   * Verify MFA token during login
   */
  static async verifyMFAToken(req: Request, res: Response) {
    try {
      const { userId, token } = req.body;
      const ipAddress = req.ip;

      // Check rate limiting
      const canAttempt = await MFAService.checkRateLimit(userId);
      if (!canAttempt) {
        return res.status(429).json({
          error: "Too many failed attempts",
          retryAfter: 900, // 15 minutes in seconds
        });
      }

      // Get secret
      const secret = await MFAService.getMFASecret(userId);

      if (!secret) {
        return res.status(400).json({ error: "MFA not enabled" });
      }

      // Verify token
      const isValid = MFAService.verifyToken(secret, token);

      // Log attempt
      await MFAService.logAttempt(userId, isValid, ipAddress);

      if (!isValid) {
        return res.status(400).json({ error: "Invalid MFA code" });
      }

      res.json({
        message: "MFA verification successful",
        verified: true,
      });
    } catch (error) {
      console.error("MFA verification error:", error);
      res.status(500).json({ error: "MFA verification failed" });
    }
  }

  /**
   * Verify backup code
   */
  static async verifyBackupCode(req: Request, res: Response) {
    try {
      const { userId, code } = req.body;
      const ipAddress = req.ip;

      // Verify backup code
      const isValid = await MFAService.verifyBackupCode(userId, code);

      // Log attempt
      await MFAService.logAttempt(userId, isValid, ipAddress);

      if (!isValid) {
        return res.status(400).json({ error: "Invalid backup code" });
      }

      // Check remaining codes
      const remainingCodes = await MFAService.getRemainingBackupCodesCount(
        userId
      );

      res.json({
        message: "Backup code verified successfully",
        verified: true,
        remainingCodes,
        warning: remainingCodes <= 2 ? "Running low on backup codes" : null,
      });
    } catch (error) {
      console.error("Backup code verification error:", error);
      res.status(500).json({ error: "Backup code verification failed" });
    }
  }

  /**
   * Regenerate backup codes
   */
  static async regenerateBackupCodes(req: Request, res: Response) {
    try {
      const userId = req.user!.userId;
      const { token } = req.body;

      // Verify MFA token first
      const secret = await MFAService.getMFASecret(userId);
      if (secret) {
        const isValid = MFAService.verifyToken(secret, token);
        if (!isValid) {
          return res.status(400).json({ error: "Invalid MFA code" });
        }
      }

      // Generate new backup codes
      const backupCodes = MFAService.generateBackupCodes(10);
      await MFAService.saveBackupCodes(userId, backupCodes);

      res.json({
        message: "Backup codes regenerated",
        backupCodes,
      });
    } catch (error) {
      console.error("Backup code regeneration error:", error);
      res.status(500).json({ error: "Failed to regenerate backup codes" });
    }
  }

  /**
   * Get MFA status
   */
  static async getMFAStatus(req: Request, res: Response) {
    try {
      const userId = req.user!.userId;

      const mfaEnabled = await MFAService.isMFAEnabled(userId);
      const remainingBackupCodes = mfaEnabled
        ? await MFAService.getRemainingBackupCodesCount(userId)
        : 0;

      res.json({
        mfaEnabled,
        remainingBackupCodes,
      });
    } catch (error) {
      console.error("Get MFA status error:", error);
      res.status(500).json({ error: "Failed to get MFA status" });
    }
  }
}
```

**Updated Login Flow with MFA:**

```typescript
// src/controllers/auth.controller.ts (updated)
export class AuthController {
  static async login(req: Request, res: Response) {
    try {
      const { email, password, mfaToken, backupCode } = req.body;

      // Step 1: Verify credentials
      const user = await User.findByEmail(email);
      if (!user) {
        return res.status(401).json({ error: "Invalid credentials" });
      }

      const isValidPassword = await user.verifyPassword(password);
      if (!isValidPassword) {
        return res.status(401).json({ error: "Invalid credentials" });
      }

      // Step 2: Check if MFA is enabled
      const mfaEnabled = await MFAService.isMFAEnabled(user.id);

      if (mfaEnabled) {
        // MFA is required
        if (!mfaToken && !backupCode) {
          // First step: credentials verified, now need MFA
          return res.status(200).json({
            requiresMFA: true,
            userId: user.id, // Temporary ID for MFA verification
            message: "Enter MFA code or backup code",
          });
        }

        // Verify MFA token or backup code
        let mfaVerified = false;

        if (mfaToken) {
          const secret = await MFAService.getMFASecret(user.id);
          mfaVerified = MFAService.verifyToken(secret!, mfaToken);
          await MFAService.logAttempt(user.id, mfaVerified, req.ip);
        } else if (backupCode) {
          mfaVerified = await MFAService.verifyBackupCode(user.id, backupCode);
        }

        if (!mfaVerified) {
          return res.status(401).json({ error: "Invalid MFA code" });
        }
      }

      // Step 3: Generate tokens
      const { accessToken, refreshToken } = JWTService.generateTokenPair({
        userId: user.id,
        email: user.email,
        role: user.role,
      });

      // Update last login
      await User.updateLastLogin(user.id);

      res.json({
        message: "Login successful",
        user: {
          id: user.id,
          email: user.email,
          name: user.name,
          mfaEnabled,
        },
        accessToken,
        refreshToken,
      });
    } catch (error) {
      console.error("Login error:", error);
      res.status(500).json({ error: "Login failed" });
    }
  }
}
```

**MFA Routes:**

```typescript
// src/routes/mfa.routes.ts
import express from "express";
import { authenticateJWT } from "../middleware/jwt-auth.middleware";
import { MFAController } from "../controllers/mfa.controller";

const router = express.Router();

// All routes require authentication
router.use(authenticateJWT);

// MFA setup flow
router.post("/setup", MFAController.setupMFA);
router.post("/verify-enable", MFAController.verifyAndEnableMFA);
router.post("/disable", MFAController.disableMFA);

// MFA status
router.get("/status", MFAController.getMFAStatus);

// Backup codes
router.post("/backup-codes/regenerate", MFAController.regenerateBackupCodes);

export default router;

// Add to main app
app.use("/api/mfa", mfaRoutes);
```

---

### 5.4 Remember This Device

**Trust Device Service:**

```typescript
// src/services/trusted-device.service.ts
import crypto from "crypto";
import { pool } from "../config/database";

export class TrustedDeviceService {
  private static DEVICE_EXPIRY_DAYS = 30;

  /**
   * Generate device token
   */
  static generateDeviceToken(): string {
    return crypto.randomBytes(32).toString("hex");
  }

  /**
   * Trust a device
   */
  static async trustDevice(
    userId: string,
    deviceInfo: {
      name?: string;
      type?: string;
      ipAddress?: string;
      userAgent?: string;
    }
  ): Promise<string> {
    const deviceToken = this.generateDeviceToken();
    const expiresAt = new Date();
    expiresAt.setDate(expiresAt.getDate() + this.DEVICE_EXPIRY_DAYS);

    const query = `
      INSERT INTO trusted_devices 
        (user_id, device_token, device_name, device_type, ip_address, user_agent, expires_at)
      VALUES ($1, $2, $3, $4, $5, $6, $7)
    `;

    await pool.query(query, [
      userId,
      deviceToken,
      deviceInfo.name || "Unknown Device",
      deviceInfo.type || "browser",
      deviceInfo.ipAddress,
      deviceInfo.userAgent,
      expiresAt,
    ]);

    return deviceToken;
  }

  /**
   * Check if device is trusted
   */
  static async isDeviceTrusted(
    userId: string,
    deviceToken: string
  ): Promise<boolean> {
    const query = `
      SELECT id
      FROM trusted_devices
      WHERE user_id = $1 
        AND device_token = $2
        AND expires_at > CURRENT_TIMESTAMP
    `;

    const result = await pool.query(query, [userId, deviceToken]);

    if (result.rows.length > 0) {
      // Update last used
      await pool.query(
        "UPDATE trusted_devices SET last_used = CURRENT_TIMESTAMP WHERE id = $1",
        [result.rows[0].id]
      );
      return true;
    }

    return false;
  }

  /**
   * Get user's trusted devices
   */
  static async getTrustedDevices(userId: string) {
    const query = `
      SELECT 
        device_token,
        device_name,
        device_type,
        ip_address,
        last_used,
        expires_at,
        created_at
      FROM trusted_devices
      WHERE user_id = $1 AND expires_at > CURRENT_TIMESTAMP
      ORDER BY last_used DESC
    `;

    const result = await pool.query(query, [userId]);
    return result.rows;
  }

  /**
   * Revoke trusted device
   */
  static async revokeDevice(
    userId: string,
    deviceToken: string
  ): Promise<void> {
    const query =
      "DELETE FROM trusted_devices WHERE user_id = $1 AND device_token = $2";
    await pool.query(query, [userId, deviceToken]);
  }

  /**
   * Revoke all trusted devices
   */
  static async revokeAllDevices(userId: string): Promise<void> {
    const query = "DELETE FROM trusted_devices WHERE user_id = $1";
    await pool.query(query, [userId]);
  }
}
```

**Updated Login with Device Trust:**

```typescript
static async login(req: Request, res: Response) {
  // ... previous login code ...

  // After MFA verification
  if (mfaVerified) {
    const { trustDevice } = req.body;
    let deviceToken: string | undefined;

    if (trustDevice) {
      // Trust this device for 30 days
      deviceToken = await TrustedDeviceService.trustDevice(user.id, {
        name: req.headers['user-agent']?.substring(0, 100),
        type: 'browser',
        ipAddress: req.ip,
        userAgent: req.headers['user-agent']
      });
    }

    res.json({
      // ... previous response ...
      deviceToken // Return to client to store in cookie/localStorage
    });
  }
}

// Check trusted device before requiring MFA
static async login(req: Request, res: Response) {
  // ... credential verification ...

  const mfaEnabled = await MFAService.isMFAEnabled(user.id);

  if (mfaEnabled) {
    // Check if device is trusted
    const { deviceToken } = req.body;

    if (deviceToken) {
      const isTrusted = await TrustedDeviceService.isDeviceTrusted(user.id, deviceToken);
      if (isTrusted) {
        // Skip MFA for trusted device
        // Continue with token generation
      }
    }

    // Otherwise require MFA...
  }
}
```

---

This is a comprehensive start to Day 5. Would you like me to continue with:

1. **SMS-based MFA implementation**
2. **Java/Spring Boot MFA implementation**
3. **Go MFA implementation**
4. **Recovery flows for lost devices**
5. **Admin forcing MFA for users**

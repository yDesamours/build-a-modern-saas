# Module 4 - Day 6: Single Sign-On (SSO) & Enterprise Authentication

**Part of:** Module 4 - Authentication & Authorization  
**Duration:** Day 6 of 7

---

## 📚 Learning Objectives

By the end of this day, you will:

- Understand SAML 2.0 protocol and implementation
- Implement OAuth 2.0 for enterprise SSO
- Integrate with Azure AD (Microsoft)
- Integrate with Google Workspace
- Set up Okta integration
- Implement Just-In-Time (JIT) user provisioning
- Handle SCIM for user management
- Build a multi-tenant SSO system
- Understand enterprise SSO security best practices

---

## 🎯 Understanding Single Sign-On (SSO)

### What is SSO?

**Single Sign-On (SSO)** allows users to authenticate once and access multiple applications without re-entering credentials.

**Benefits for SaaS:**

- **Better UX:** Users login once for all apps
- **Enterprise Sales:** Required for B2B customers
- **Security:** Centralized authentication control
- **Compliance:** Meet enterprise security requirements

**Common SSO Protocols:**

| Protocol           | Use Case                | Complexity | Adoption           |
| ------------------ | ----------------------- | ---------- | ------------------ |
| **SAML 2.0**       | Enterprise SSO          | High       | Very High          |
| **OAuth 2.0**      | Modern apps, API access | Medium     | Very High          |
| **OpenID Connect** | Modern SSO              | Medium     | High               |
| **LDAP**           | Legacy enterprise       | Low        | Medium (declining) |

---

## 📖 Section 1: SAML 2.0 Implementation

### 6.1 Understanding SAML 2.0

**SAML (Security Assertion Markup Language)** is an XML-based standard for exchanging authentication and authorization data.

**SAML Flow:**

```
1. User accesses SaaS app (Service Provider)
   ↓
2. App redirects to company IdP (Identity Provider)
   ↓
3. User authenticates at IdP (e.g., Okta, Azure AD)
   ↓
4. IdP sends SAML assertion back to app
   ↓
5. App validates assertion
   ↓
6. User is logged in
```

**Key SAML Concepts:**

- **Service Provider (SP):** Your SaaS application
- **Identity Provider (IdP):** Customer's auth system (Okta, Azure AD)
- **SAML Assertion:** XML document with user identity
- **SSO URL:** Where IdP sends the assertion
- **Entity ID:** Unique identifier for SP/IdP
- **Metadata:** XML document describing SP/IdP configuration

---

### 6.2 Database Schema for SSO

```sql
-- SSO configurations (per tenant/organization)
CREATE TABLE sso_configurations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,

  -- Provider info
  provider_type VARCHAR(50) NOT NULL, -- saml, oidc, oauth2
  provider_name VARCHAR(100), -- "Okta", "Azure AD", etc.

  -- SAML specific
  saml_entity_id VARCHAR(500),
  saml_sso_url VARCHAR(500),
  saml_slo_url VARCHAR(500), -- Single Logout URL
  saml_x509_cert TEXT, -- IdP certificate

  -- OAuth/OIDC specific
  oauth_client_id VARCHAR(255),
  oauth_client_secret VARCHAR(255),
  oauth_authorization_url VARCHAR(500),
  oauth_token_url VARCHAR(500),
  oauth_userinfo_url VARCHAR(500),

  -- General settings
  enabled BOOLEAN DEFAULT TRUE,
  default_role VARCHAR(50) DEFAULT 'user',
  jit_provisioning BOOLEAN DEFAULT TRUE, -- Just-In-Time user creation

  -- Metadata
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by UUID REFERENCES users(id),

  UNIQUE(tenant_id)
);

-- SSO sessions
CREATE TABLE sso_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
  sso_config_id UUID REFERENCES sso_configurations(id) ON DELETE CASCADE,

  session_index VARCHAR(255), -- SAML SessionIndex
  name_id VARCHAR(255), -- SAML NameID

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP NOT NULL,
  last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- SSO user mappings (external ID to internal user)
CREATE TABLE sso_user_mappings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,

  external_id VARCHAR(255) NOT NULL, -- ID from IdP
  provider_type VARCHAR(50) NOT NULL,

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(tenant_id, external_id, provider_type)
);

-- Indexes
CREATE INDEX idx_sso_config_tenant ON sso_configurations(tenant_id);
CREATE INDEX idx_sso_sessions_user ON sso_sessions(user_id);
CREATE INDEX idx_sso_sessions_expires ON sso_sessions(expires_at);
CREATE INDEX idx_sso_mappings_external ON sso_user_mappings(tenant_id, external_id);
```

---

### 6.3 Node.js SAML Implementation

**Dependencies:**

```bash
npm install passport passport-saml express-session
npm install --save-dev @types/passport @types/passport-saml
```

**SAML Configuration:**

```typescript
// src/config/saml.ts
import { Strategy as SamlStrategy, Profile } from "passport-saml";
import passport from "passport";
import { pool } from "./database";
import { User } from "../models/user.model";

interface SAMLConfig {
  entryPoint: string;
  issuer: string;
  callbackUrl: string;
  cert: string;
  identifierFormat?: string;
}

export class SAMLConfigService {
  /**
   * Get SAML configuration for tenant
   */
  static async getTenantSAMLConfig(
    tenantId: string
  ): Promise<SAMLConfig | null> {
    const query = `
      SELECT 
        saml_sso_url as entryPoint,
        saml_entity_id as issuer,
        saml_x509_cert as cert
      FROM sso_configurations
      WHERE tenant_id = $1 AND provider_type = 'saml' AND enabled = TRUE
    `;

    const result = await pool.query(query, [tenantId]);

    if (result.rows.length === 0) {
      return null;
    }

    const config = result.rows[0];

    return {
      entryPoint: config.entrypoint,
      issuer: `https://myapp.com/saml/metadata/${tenantId}`,
      callbackUrl: `https://myapp.com/saml/callback/${tenantId}`,
      cert: config.cert,
      identifierFormat:
        "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress",
    };
  }

  /**
   * Create SAML strategy for tenant
   */
  static async createSAMLStrategy(
    tenantId: string
  ): Promise<SamlStrategy | null> {
    const config = await this.getTenantSAMLConfig(tenantId);

    if (!config) {
      return null;
    }

    return new SamlStrategy(config, async (profile: Profile, done: any) => {
      try {
        // Extract user info from SAML assertion
        const email = profile.email || profile.nameID;
        const firstName = profile.givenName || profile["firstName"];
        const lastName = profile.surname || profile["lastName"];

        // Find or create user
        let user = await User.findByEmail(email);

        if (!user) {
          // JIT Provisioning - create user on first login
          user = await User.create({
            email,
            name: `${firstName} ${lastName}`,
            tenantId,
            emailVerified: true, // Trust IdP verification
            ssoEnabled: true,
          });

          // Create SSO mapping
          await pool.query(
            `INSERT INTO sso_user_mappings (user_id, tenant_id, external_id, provider_type)
               VALUES ($1, $2, $3, 'saml')`,
            [user.id, tenantId, profile.nameID]
          );
        }

        done(null, user);
      } catch (error) {
        done(error);
      }
    });
  }
}
```

**SAML Service:**

```typescript
// src/services/saml.service.ts
import { SamlStrategy } from "passport-saml";
import { SAMLConfigService } from "../config/saml";

export class SAMLService {
  private strategies: Map<string, SamlStrategy> = new Map();

  /**
   * Get or create SAML strategy for tenant
   */
  async getStrategy(tenantId: string): Promise<SamlStrategy | null> {
    // Check cache
    if (this.strategies.has(tenantId)) {
      return this.strategies.get(tenantId)!;
    }

    // Create new strategy
    const strategy = await SAMLConfigService.createSAMLStrategy(tenantId);

    if (strategy) {
      this.strategies.set(tenantId, strategy);
    }

    return strategy;
  }

  /**
   * Clear cached strategy (when config changes)
   */
  clearStrategy(tenantId: string): void {
    this.strategies.delete(tenantId);
  }

  /**
   * Generate SAML metadata for tenant
   */
  async generateMetadata(tenantId: string): Promise<string> {
    const strategy = await this.getStrategy(tenantId);

    if (!strategy) {
      throw new Error("SAML not configured for tenant");
    }

    return strategy.generateServiceProviderMetadata(
      null, // No signing cert
      null // No encryption cert
    );
  }

  /**
   * Create SSO session
   */
  async createSSOSession(
    userId: string,
    tenantId: string,
    ssoConfigId: string,
    sessionIndex: string,
    nameId: string
  ): Promise<void> {
    const expiresAt = new Date(Date.now() + 8 * 60 * 60 * 1000); // 8 hours

    await pool.query(
      `INSERT INTO sso_sessions 
        (user_id, tenant_id, sso_config_id, session_index, name_id, expires_at)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [userId, tenantId, ssoConfigId, sessionIndex, nameId, expiresAt]
    );
  }
}

export const samlService = new SAMLService();
```

**SAML Controller:**

```typescript
// src/controllers/saml.controller.ts
import { Request, Response } from "express";
import { samlService } from "../services/saml.service";
import { JWTService } from "../services/jwt.service";

export class SAMLController {
  /**
   * Initiate SAML login
   */
  static async initiateLogin(req: Request, res: Response) {
    try {
      const { tenantId } = req.params;

      const strategy = await samlService.getStrategy(tenantId);

      if (!strategy) {
        return res
          .status(404)
          .json({ error: "SSO not configured for this organization" });
      }

      // Generate SAML auth request
      strategy.authenticate(req, {
        failureRedirect: "/login?error=sso_failed",
        session: false,
      });
    } catch (error) {
      console.error("SAML initiate error:", error);
      res.status(500).json({ error: "Failed to initiate SSO" });
    }
  }

  /**
   * Handle SAML callback (Assertion Consumer Service)
   */
  static async handleCallback(req: Request, res: Response) {
    try {
      const { tenantId } = req.params;

      const strategy = await samlService.getStrategy(tenantId);

      if (!strategy) {
        return res.redirect("/login?error=sso_not_configured");
      }

      // Authenticate with SAML response
      strategy.authenticate(
        req,
        { session: false },
        async (err, user, info) => {
          if (err || !user) {
            console.error("SAML auth error:", err);
            return res.redirect("/login?error=sso_failed");
          }

          // Create SSO session
          await samlService.createSSOSession(
            user.id,
            tenantId,
            info.ssoConfigId,
            info.sessionIndex,
            info.nameID
          );

          // Generate JWT tokens
          const tokens = JWTService.generateTokenPair({
            userId: user.id,
            email: user.email,
            role: user.role,
            tenantId,
          });

          // Redirect to frontend with tokens
          res.redirect(
            `${process.env.FRONTEND_URL}/auth/callback?` +
              `accessToken=${tokens.accessToken}&` +
              `refreshToken=${tokens.refreshToken}`
          );
        }
      );
    } catch (error) {
      console.error("SAML callback error:", error);
      res.redirect("/login?error=sso_failed");
    }
  }

  /**
   * Serve SAML metadata
   */
  static async getMetadata(req: Request, res: Response) {
    try {
      const { tenantId } = req.params;

      const metadata = await samlService.generateMetadata(tenantId);

      res.set("Content-Type", "application/xml");
      res.send(metadata);
    } catch (error) {
      console.error("SAML metadata error:", error);
      res.status(500).json({ error: "Failed to generate metadata" });
    }
  }

  /**
   * Handle SAML logout (Single Logout)
   */
  static async handleLogout(req: Request, res: Response) {
    try {
      const { tenantId } = req.params;
      const userId = req.user?.userId;

      if (userId) {
        // Delete SSO session
        await pool.query(
          "DELETE FROM sso_sessions WHERE user_id = $1 AND tenant_id = $2",
          [userId, tenantId]
        );
      }

      const strategy = await samlService.getStrategy(tenantId);

      if (strategy) {
        // Initiate SAML logout
        strategy.logout(req, (err, url) => {
          if (url) {
            res.redirect(url);
          } else {
            res.redirect("/login");
          }
        });
      } else {
        res.redirect("/login");
      }
    } catch (error) {
      console.error("SAML logout error:", error);
      res.redirect("/login");
    }
  }
}
```

**SAML Routes:**

```typescript
// src/routes/saml.routes.ts
import express from "express";
import { SAMLController } from "../controllers/saml.controller";

const router = express.Router();

// SAML endpoints per tenant
router.get("/:tenantId/login", SAMLController.initiateLogin);
router.post("/:tenantId/callback", SAMLController.handleCallback);
router.get("/:tenantId/metadata", SAMLController.getMetadata);
router.post("/:tenantId/logout", SAMLController.handleLogout);

export default router;

// Add to main app
app.use("/saml", samlRoutes);
```

---

### 6.4 Admin SSO Configuration

**SSO Configuration Controller:**

```typescript
// src/controllers/sso-config.controller.ts
import { Request, Response } from "express";
import { pool } from "../config/database";
import { samlService } from "../services/saml.service";

export class SSOConfigController {
  /**
   * Get SSO configuration
   */
  static async getConfig(req: Request, res: Response) {
    try {
      const tenantId = req.user!.tenantId;

      const query = `
        SELECT 
          id,
          provider_type,
          provider_name,
          saml_entity_id,
          saml_sso_url,
          saml_slo_url,
          enabled,
          default_role,
          jit_provisioning,
          created_at
        FROM sso_configurations
        WHERE tenant_id = $1
      `;

      const result = await pool.query(query, [tenantId]);

      if (result.rows.length === 0) {
        return res.json({ configured: false });
      }

      res.json({
        configured: true,
        config: result.rows[0],
      });
    } catch (error) {
      console.error("Get SSO config error:", error);
      res.status(500).json({ error: "Failed to get SSO configuration" });
    }
  }

  /**
   * Configure SAML SSO
   */
  static async configureSAML(req: Request, res: Response) {
    try {
      const tenantId = req.user!.tenantId;
      const userId = req.user!.userId;
      const {
        providerName,
        entityId,
        ssoUrl,
        sloUrl,
        certificate,
        defaultRole,
        jitProvisioning,
      } = req.body;

      // Validate certificate format
      if (!certificate.includes("BEGIN CERTIFICATE")) {
        return res.status(400).json({
          error: "Invalid certificate format. Must be X.509 PEM format.",
        });
      }

      const query = `
        INSERT INTO sso_configurations (
          tenant_id,
          provider_type,
          provider_name,
          saml_entity_id,
          saml_sso_url,
          saml_slo_url,
          saml_x509_cert,
          default_role,
          jit_provisioning,
          created_by
        )
        VALUES ($1, 'saml', $2, $3, $4, $5, $6, $7, $8, $9)
        ON CONFLICT (tenant_id)
        DO UPDATE SET
          provider_name = $2,
          saml_entity_id = $3,
          saml_sso_url = $4,
          saml_slo_url = $5,
          saml_x509_cert = $6,
          default_role = $7,
          jit_provisioning = $8,
          updated_at = CURRENT_TIMESTAMP
        RETURNING id
      `;

      await pool.query(query, [
        tenantId,
        providerName,
        entityId,
        ssoUrl,
        sloUrl,
        certificate,
        defaultRole || "user",
        jitProvisioning !== false,
        userId,
      ]);

      // Clear cached strategy
      samlService.clearStrategy(tenantId);

      // Generate metadata URL
      const metadataUrl = `https://myapp.com/saml/${tenantId}/metadata`;
      const acsUrl = `https://myapp.com/saml/${tenantId}/callback`;

      res.json({
        message: "SSO configured successfully",
        metadataUrl,
        acsUrl,
        entityId: `https://myapp.com/saml/metadata/${tenantId}`,
      });
    } catch (error) {
      console.error("Configure SAML error:", error);
      res.status(500).json({ error: "Failed to configure SSO" });
    }
  }

  /**
   * Test SSO configuration
   */
  static async testConfig(req: Request, res: Response) {
    try {
      const tenantId = req.user!.tenantId;

      // Try to create strategy
      const strategy = await samlService.getStrategy(tenantId);

      if (!strategy) {
        return res.status(400).json({
          error: "SSO not configured",
          success: false,
        });
      }

      res.json({
        success: true,
        message: "SSO configuration is valid",
        loginUrl: `https://myapp.com/saml/${tenantId}/login`,
      });
    } catch (error) {
      res.status(400).json({
        success: false,
        error: error.message || "Invalid SSO configuration",
      });
    }
  }

  /**
   * Disable SSO
   */
  static async disableSSO(req: Request, res: Response) {
    try {
      const tenantId = req.user!.tenantId;

      await pool.query(
        "UPDATE sso_configurations SET enabled = FALSE WHERE tenant_id = $1",
        [tenantId]
      );

      samlService.clearStrategy(tenantId);

      res.json({ message: "SSO disabled" });
    } catch (error) {
      console.error("Disable SSO error:", error);
      res.status(500).json({ error: "Failed to disable SSO" });
    }
  }
}
```

---

## 📖 Section 2: Azure AD / Microsoft 365 SSO

### 6.5 Azure AD OAuth 2.0 Implementation

**Azure AD uses OAuth 2.0 / OpenID Connect for SSO**

**Dependencies:**

```bash
npm install passport-azure-ad
npm install --save-dev @types/passport-azure-ad
```

**Azure AD Configuration:**

```typescript
// src/config/azure-ad.ts
import { OIDCStrategy, IProfile, VerifyCallback } from "passport-azure-ad";
import passport from "passport";
import { User } from "../models/user.model";

export interface AzureADConfig {
  clientID: string;
  clientSecret: string;
  tenantID: string; // or 'common' for multi-tenant
  redirectUrl: string;
}

export class AzureADService {
  /**
   * Create Azure AD strategy
   */
  static createStrategy(config: AzureADConfig, tenantId: string) {
    return new OIDCStrategy(
      {
        identityMetadata: `https://login.microsoftonline.com/${config.tenantID}/v2.0/.well-known/openid-configuration`,
        clientID: config.clientID,
        clientSecret: config.clientSecret,
        responseType: "code",
        responseMode: "form_post",
        redirectUrl: config.redirectUrl,
        allowHttpForRedirectUrl: process.env.NODE_ENV === "development",
        scope: ["profile", "email", "openid"],
        passReqToCallback: true,
      },
      async (req: any, profile: IProfile, done: VerifyCallback) => {
        try {
          const email = profile._json.email || profile._json.preferred_username;
          const name = profile.displayName || profile._json.name;

          // Find or create user
          let user = await User.findByEmail(email);

          if (!user) {
            // JIT Provisioning
            user = await User.create({
              email,
              name,
              tenantId,
              emailVerified: true,
              ssoEnabled: true,
              azureAdId: profile.oid,
            });
          }

          done(null, user);
        } catch (error) {
          done(error);
        }
      }
    );
  }
}
```

**Azure AD Controller:**

```typescript
// src/controllers/azure-ad.controller.ts
import { Request, Response } from "express";
import passport from "passport";
import { AzureADService } from "../config/azure-ad";
import { JWTService } from "../services/jwt.service";

export class AzureADController {
  /**
   * Initiate Azure AD login
   */
  static async initiateLogin(req: Request, res: Response, next: any) {
    const { tenantId } = req.params;

    // Get Azure AD config for tenant
    const config = await getAzureADConfig(tenantId);

    if (!config) {
      return res.status(404).json({ error: "Azure AD not configured" });
    }

    // Create dynamic strategy
    const strategy = AzureADService.createStrategy(config, tenantId);
    passport.use("azuread-custom", strategy);

    // Authenticate
    passport.authenticate("azuread-custom", {
      session: false,
      failureRedirect: "/login?error=azure_ad_failed",
    })(req, res, next);
  }

  /**
   * Handle Azure AD callback
   */
  static async handleCallback(req: Request, res: Response, next: any) {
    const { tenantId } = req.params;

    passport.authenticate(
      "azuread-custom",
      {
        session: false,
        failureRedirect: "/login?error=azure_ad_failed",
      },
      (err: any, user: any) => {
        if (err || !user) {
          return res.redirect("/login?error=azure_ad_failed");
        }

        // Generate tokens
        const tokens = JWTService.generateTokenPair({
          userId: user.id,
          email: user.email,
          role: user.role,
          tenantId,
        });

        res.redirect(
          `${process.env.FRONTEND_URL}/auth/callback?` +
            `accessToken=${tokens.accessToken}&` +
            `refreshToken=${tokens.refreshToken}`
        );
      }
    )(req, res, next);
  }
}

async function getAzureADConfig(tenantId: string) {
  const result = await pool.query(
    `SELECT 
      oauth_client_id as clientID,
      oauth_client_secret as clientSecret,
      provider_name as tenantID
    FROM sso_configurations
    WHERE tenant_id = $1 AND provider_type = 'azure_ad' AND enabled = TRUE`,
    [tenantId]
  );

  if (result.rows.length === 0) return null;

  return {
    ...result.rows[0],
    redirectUrl: `https://myapp.com/azure-ad/${tenantId}/callback`,
  };
}
```

---

## 📖 Section 3: Google Workspace SSO

### 6.6 Google Workspace OAuth 2.0

**Google Workspace Configuration:**

```typescript
// src/config/google-workspace.ts
import { Strategy as GoogleStrategy } from "passport-google-oauth20";
import passport from "passport";
import { User } from "../models/user.model";

export interface GoogleWorkspaceConfig {
  clientID: string;
  clientSecret: string;
  callbackURL: string;
  hostedDomain?: string; // Restrict to specific domain
}

export class GoogleWorkspaceService {
  static createStrategy(config: GoogleWorkspaceConfig, tenantId: string) {
    return new GoogleStrategy(
      {
        clientID: config.clientID,
        clientSecret: config.clientSecret,
        callbackURL: config.callbackURL,
        passReqToCallback: true,
      },
      async (
        req: any,
        accessToken: string,
        refreshToken: string,
        profile: any,
        done: any
      ) => {
        try {
          const email = profile.emails![0].value;

          // Verify domain if hostedDomain is set
          if (config.hostedDomain) {
            const domain = email.split("@")[1];
            if (domain !== config.hostedDomain) {
              return done(new Error("Email domain not allowed"));
            }
          }

          // Find or create user
          let user = await User.findByEmail(email);

          if (!user) {
            user = await User.create({
              email,
              name: profile.displayName,
              tenantId,
              emailVerified: true,
              ssoEnabled: true,
              googleId: profile.id,
            });
          }

          done(null, user);
        } catch (error) {
          done(error);
        }
      }
    );
  }
}
```

---

## 📖 Section 4: Okta Integration

### 6.7 Okta SAML/OAuth Configuration

Okta supports both SAML 2.0 and OAuth 2.0/OpenID Connect.

**Okta OAuth 2.0:**

```typescript
// src/config/okta.ts
import { Strategy as OAuthStrategy } from "passport-oauth2";
import axios from "axios";

export interface OktaConfig {
  clientID: string;
  clientSecret: string;
  authorizationURL: string; // https://{yourOktaDomain}/oauth2/v1/authorize
  tokenURL: string; // https://{yourOktaDomain}/oauth2/v1/token
  callbackURL: string;
}

export class OktaService {
  static createStrategy(config: OktaConfig, tenantId: string) {
    const strategy = new OAuthStrategy(
      {
        authorizationURL: config.authorizationURL,
        tokenURL: config.tokenURL,
        clientID: config.clientID,
        clientSecret: config.clientSecret,
        callbackURL: config.callbackURL,
        scope: ["openid", "profile", "email"],
      },
      async (
        accessToken: string,
        refreshToken: string,
        profile: any,
        done: any
      ) => {
        try {
          // Get user info from Okta
          const response = await axios.get(
            `https://${config.oktaDomain}/oauth2/v1/userinfo`,
            {
              headers: { Authorization: `Bearer ${accessToken}` },
            }
          );

          const userInfo = response.data;
          const email = userInfo.email;

          // Find or create user
          let user = await User.findByEmail(email);

          if (!user) {
            user = await User.create({
              email,
              name: userInfo.name,
              tenantId,
              emailVerified: true,
              ssoEnabled: true,
              oktaId: userInfo.sub,
            });
          }

          done(null, user);
        } catch (error) {
          done(error);
        }
      }
    );

    return strategy;
  }
}
```

---

## 📖 Section 5: Just-In-Time (JIT) Provisioning

### 6.8 Automatic User Creation on First Login

**JIT Provisioning Service:**

```typescript
// src/services/jit-provisioning.service.ts
import { pool } from "../config/database";
import { User } from "../models/user.model";

export interface JITUserData {
  email: string;
  firstName?: string;
  lastName?: string;
  displayName?: string;
  department?: string;
  title?: string;
  groups?: string[];
  externalId: string; // ID from IdP
}

export class JITProvisioningService {
  /**
   * Create or update user from SSO
   */
  static async provisionUser(
    tenantId: string,
    providerType: string,
    userData: JITUserData
  ): Promise<User> {
    // Check if user already exists
    let user = await User.findByEmail(userData.email);

    if (user) {
      // Update existing user
      return await this.updateExistingUser(user, userData);
    }

    // Get SSO configuration
    const config = await this.getSSOConfig(tenantId);

    if (!config.jitProvisioning) {
      throw new Error("JIT provisioning not enabled for this organization");
    }

    // Create new user
    return await this.createNewUser(tenantId, providerType, userData, config);
  }

  /**
   * Create new user from SSO data
   */
  private static async createNewUser(
    tenantId: string,
    providerType: string,
    userData: JITUserData,
    config: any
  ): Promise<User> {
    const client = await pool.connect();

    try {
      await client.query("BEGIN");

      // Create user
      const name =
        userData.displayName ||
        `${userData.firstName} ${userData.lastName}`.trim() ||
        userData.email.split("@")[0];

      const user = await User.create({
        email: userData.email,
        name,
        tenantId,
        role: config.defaultRole || "user",
        emailVerified: true,
        ssoEnabled: true,
        department: userData.department,
        title: userData.title,
      });

      // Create SSO mapping
      await client.query(
        `INSERT INTO sso_user_mappings (user_id, tenant_id, external_id, provider_type)
         VALUES ($1, $2, $3, $4)`,
        [user.id, tenantId, userData.externalId, providerType]
      );

      // Assign groups/roles if provided
      if (userData.groups && userData.groups.length > 0) {
        await this.assignGroupBasedRoles(
          client,
          user.id,
          userData.groups,
          config
        );
      }

      await client.query("COMMIT");

      // Send welcome email
      await this.sendWelcomeEmail(user.email, name, tenantId);

      return user;
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Update existing user with SSO data
   */
  private static async updateExistingUser(
    user: User,
    userData: JITUserData
  ): Promise<User> {
    // Update profile if changed
    const updates: any = {};

    if (userData.department && user.department !== userData.department) {
      updates.department = userData.department;
    }

    if (userData.title && user.title !== userData.title) {
      updates.title = userData.title;
    }

    if (Object.keys(updates).length > 0) {
      await User.update(user.id, updates);
    }

    return user;
  }

  /**
   * Assign roles based on IdP groups
   */
  private static async assignGroupBasedRoles(
    client: any,
    userId: string,
    groups: string[],
    config: any
  ): Promise<void> {
    // Get group-to-role mappings
    const mappings = config.groupMappings || {};

    for (const group of groups) {
      const roleName = mappings[group];

      if (roleName) {
        // Assign role
        await client.query(
          `INSERT INTO user_roles (user_id, role_id)
           SELECT $1, id FROM roles WHERE name = $2
           ON CONFLICT DO NOTHING`,
          [userId, roleName]
        );
      }
    }
  }

  /**
   * Get SSO configuration
   */
  private static async getSSOConfig(tenantId: string): Promise<any> {
    const result = await pool.query(
      `SELECT jit_provisioning, default_role, group_mappings
       FROM sso_configurations
       WHERE tenant_id = $1`,
      [tenantId]
    );

    if (result.rows.length === 0) {
      throw new Error("SSO not configured");
    }

    return result.rows[0];
  }

  /**
   * Send welcome email to new SSO user
   */
  private static async sendWelcomeEmail(
    email: string,
    name: string,
    tenantId: string
  ): Promise<void> {
    // Implementation depends on your email service
    console.log(`Welcome email sent to ${email}`);
  }
}
```

**Group Mapping Configuration:**

```typescript
// Example group mapping configuration
const groupMappings = {
  Admins: "admin",
  Managers: "manager",
  Developers: "developer",
  Support: "support",
  Users: "user",
};

// Store in database
await pool.query(
  `UPDATE sso_configurations 
   SET group_mappings = $1 
   WHERE tenant_id = $2`,
  [JSON.stringify(groupMappings), tenantId]
);
```

---

## 📖 Section 6: SCIM (System for Cross-domain Identity Management)

### 6.9 SCIM User Provisioning

**SCIM allows IdPs to automatically create/update/delete users in your SaaS application.**

**Dependencies:**

```bash
npm install express-validator
```

**SCIM User Schema:**

```typescript
// src/models/scim.types.ts
export interface SCIMUser {
  schemas: string[];
  id?: string;
  externalId?: string;
  userName: string;
  name: {
    formatted?: string;
    familyName?: string;
    givenName?: string;
  };
  displayName?: string;
  emails: Array;
  active: boolean;
  groups?: Array;
  meta?: {
    resourceType: string;
    created?: string;
    lastModified?: string;
  };
}

export interface SCIMGroup {
  schemas: string[];
  id?: string;
  displayName: string;
  members?: Array;
}
```

**SCIM Service:**

```typescript
// src/services/scim.service.ts
import { pool } from "../config/database";
import { User } from "../models/user.model";
import { SCIMUser } from "../models/scim.types";

export class SCIMService {
  /**
   * Create user via SCIM
   */
  static async createUser(tenantId: string, scimUser: SCIMUser): Promise {
    const email =
      scimUser.emails.find((e) => e.primary)?.value || scimUser.emails[0].value;
    const name =
      scimUser.displayName ||
      `${scimUser.name.givenName} ${scimUser.name.familyName}`.trim();

    // Create user
    const user = await User.create({
      email,
      name,
      tenantId,
      emailVerified: true,
      ssoEnabled: true,
      scimExternalId: scimUser.externalId,
      active: scimUser.active,
    });

    return this.toSCIMUser(user);
  }

  /**
   * Get user by ID
   */
  static async getUser(tenantId: string, userId: string): Promise {
    const user = await User.findById(userId);

    if (!user || user.tenantId !== tenantId) {
      return null;
    }

    return this.toSCIMUser(user);
  }

  /**
   * Update user
   */
  static async updateUser(
    tenantId: string,
    userId: string,
    scimUser: Partial
  ): Promise {
    const updates: any = {};

    if (scimUser.displayName) {
      updates.name = scimUser.displayName;
    }

    if (scimUser.active !== undefined) {
      updates.active = scimUser.active;
    }

    if (scimUser.emails && scimUser.emails.length > 0) {
      const email =
        scimUser.emails.find((e) => e.primary)?.value ||
        scimUser.emails[0].value;
      updates.email = email;
    }

    await User.update(userId, updates);

    const user = await User.findById(userId);
    return this.toSCIMUser(user!);
  }

  /**
   * Delete (deactivate) user
   */
  static async deleteUser(tenantId: string, userId: string): Promise {
    await User.update(userId, { active: false });
  }

  /**
   * List users (with filtering and pagination)
   */
  static async listUsers(
    tenantId: string,
    options: {
      startIndex?: number;
      count?: number;
      filter?: string;
    }
  ): Promise {
    const startIndex = options.startIndex || 1;
    const count = Math.min(options.count || 100, 1000); // Max 1000
    const offset = startIndex - 1;

    // Build query
    let query = "SELECT * FROM users WHERE tenant_id = $1";
    const params: any[] = [tenantId];

    // Apply filter (simplified - full SCIM filter parsing is complex)
    if (options.filter) {
      // Example: userName eq "john@example.com"
      const emailMatch = options.filter.match(/userName eq "([^"]+)"/);
      if (emailMatch) {
        query += " AND email = $2";
        params.push(emailMatch[1]);
      }
    }

    // Get total count
    const countResult = await pool.query(
      query.replace("SELECT *", "SELECT COUNT(*)"),
      params
    );
    const totalResults = parseInt(countResult.rows[0].count);

    // Get page of results
    query += ` ORDER BY created_at LIMIT ${params.length + 1} OFFSET ${
      params.length + 2
    }`;
    params.push(count, offset);

    const result = await pool.query(query, params);

    return {
      schemas: ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
      totalResults,
      startIndex,
      itemsPerPage: result.rows.length,
      Resources: result.rows.map((row) => this.toSCIMUser(row)),
    };
  }

  /**
   * Convert internal user to SCIM format
   */
  private static toSCIMUser(user: any): SCIMUser {
    const [firstName, ...lastNameParts] = user.name.split(" ");
    const lastName = lastNameParts.join(" ");

    return {
      schemas: ["urn:ietf:params:scim:schemas:core:2.0:User"],
      id: user.id,
      externalId: user.scimExternalId,
      userName: user.email,
      name: {
        formatted: user.name,
        givenName: firstName,
        familyName: lastName,
      },
      displayName: user.name,
      emails: [
        {
          value: user.email,
          type: "work",
          primary: true,
        },
      ],
      active: user.active !== false,
      meta: {
        resourceType: "User",
        created: user.createdAt?.toISOString(),
        lastModified: user.updatedAt?.toISOString(),
      },
    };
  }
}
```

**SCIM Controller:**

```typescript
// src/controllers/scim.controller.ts
import { Request, Response } from "express";
import { SCIMService } from "../services/scim.service";
import { body, validationResult } from "express-validator";

export class SCIMController {
  /**
   * Create user (POST /scim/v2/Users)
   */
  static async createUser(req: Request, res: Response) {
    try {
      const tenantId = req.scimTenant!; // From auth middleware
      const scimUser = req.body;

      const user = await SCIMService.createUser(tenantId, scimUser);

      res.status(201).set("Location", `/scim/v2/Users/${user.id}`).json(user);
    } catch (error) {
      console.error("SCIM create user error:", error);
      res.status(400).json({
        schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
        status: "400",
        detail: error.message,
      });
    }
  }

  /**
   * Get user (GET /scim/v2/Users/:id)
   */
  static async getUser(req: Request, res: Response) {
    try {
      const tenantId = req.scimTenant!;
      const { id } = req.params;

      const user = await SCIMService.getUser(tenantId, id);

      if (!user) {
        return res.status(404).json({
          schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
          status: "404",
          detail: "User not found",
        });
      }

      res.json(user);
    } catch (error) {
      console.error("SCIM get user error:", error);
      res.status(500).json({
        schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
        status: "500",
        detail: "Internal server error",
      });
    }
  }

  /**
   * Update user (PUT /scim/v2/Users/:id)
   */
  static async updateUser(req: Request, res: Response) {
    try {
      const tenantId = req.scimTenant!;
      const { id } = req.params;
      const updates = req.body;

      const user = await SCIMService.updateUser(tenantId, id, updates);

      res.json(user);
    } catch (error) {
      console.error("SCIM update user error:", error);
      res.status(400).json({
        schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
        status: "400",
        detail: error.message,
      });
    }
  }

  /**
   * Delete user (DELETE /scim/v2/Users/:id)
   */
  static async deleteUser(req: Request, res: Response) {
    try {
      const tenantId = req.scimTenant!;
      const { id } = req.params;

      await SCIMService.deleteUser(tenantId, id);

      res.status(204).send();
    } catch (error) {
      console.error("SCIM delete user error:", error);
      res.status(404).json({
        schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
        status: "404",
        detail: "User not found",
      });
    }
  }

  /**
   * List users (GET /scim/v2/Users)
   */
  static async listUsers(req: Request, res: Response) {
    try {
      const tenantId = req.scimTenant!;
      const { startIndex, count, filter } = req.query;

      const result = await SCIMService.listUsers(tenantId, {
        startIndex: startIndex ? parseInt(startIndex as string) : undefined,
        count: count ? parseInt(count as string) : undefined,
        filter: filter as string,
      });

      res.json(result);
    } catch (error) {
      console.error("SCIM list users error:", error);
      res.status(500).json({
        schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
        status: "500",
        detail: "Internal server error",
      });
    }
  }

  /**
   * Service Provider Config (GET /scim/v2/ServiceProviderConfig)
   */
  static getServiceProviderConfig(req: Request, res: Response) {
    res.json({
      schemas: ["urn:ietf:params:scim:schemas:core:2.0:ServiceProviderConfig"],
      patch: {
        supported: true,
      },
      bulk: {
        supported: false,
      },
      filter: {
        supported: true,
        maxResults: 1000,
      },
      changePassword: {
        supported: false,
      },
      sort: {
        supported: true,
      },
      authenticationSchemes: [
        {
          name: "OAuth Bearer Token",
          description: "Authentication scheme using the OAuth Bearer Token",
          specUri: "http://www.rfc-editor.org/info/rfc6750",
          type: "oauthbearertoken",
        },
      ],
    });
  }
}
```

**SCIM Authentication Middleware:**

```typescript
// src/middleware/scim-auth.middleware.ts
import { Request, Response, NextFunction } from "express";
import { pool } from "../config/database";

declare global {
  namespace Express {
    interface Request {
      scimTenant?: string;
    }
  }
}

export async function authenticateSCIM(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res.status(401).json({
      schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
      status: "401",
      detail: "Unauthorized",
    });
  }

  const token = authHeader.substring(7);

  try {
    // Validate SCIM token (store in sso_configurations)
    const result = await pool.query(
      `SELECT tenant_id 
       FROM sso_configurations 
       WHERE scim_token = $1 AND enabled = TRUE`,
      [token]
    );

    if (result.rows.length === 0) {
      return res.status(401).json({
        schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
        status: "401",
        detail: "Invalid token",
      });
    }

    req.scimTenant = result.rows[0].tenant_id;
    next();
  } catch (error) {
    console.error("SCIM auth error:", error);
    res.status(500).json({
      schemas: ["urn:ietf:params:scim:api:messages:2.0:Error"],
      status: "500",
      detail: "Internal server error",
    });
  }
}
```

**SCIM Routes:**

```typescript
// src/routes/scim.routes.ts
import express from "express";
import { SCIMController } from "../controllers/scim.controller";
import { authenticateSCIM } from "../middleware/scim-auth.middleware";

const router = express.Router();

// All SCIM routes require authentication
router.use(authenticateSCIM);

// Service Provider Config
router.get("/ServiceProviderConfig", SCIMController.getServiceProviderConfig);

// Users
router.get("/Users", SCIMController.listUsers);
router.get("/Users/:id", SCIMController.getUser);
router.post("/Users", SCIMController.createUser);
router.put("/Users/:id", SCIMController.updateUser);
router.delete("/Users/:id", SCIMController.deleteUser);

export default router;

// Add to main app
app.use("/scim/v2", scimRoutes);
```

**Generate SCIM Token:**

```typescript
// Generate SCIM token for tenant
import crypto from "crypto";

async function generateSCIMToken(tenantId: string): Promise {
  const token = crypto.randomBytes(32).toString("hex");

  await pool.query(
    `UPDATE sso_configurations 
     SET scim_token = $1, scim_enabled = TRUE 
     WHERE tenant_id = $2`,
    [token, tenantId]
  );

  return token;
}

// SCIM endpoint for tenant
const scimEndpoint = `https://myapp.com/scim/v2`;
const scimToken = await generateSCIMToken(tenantId);

console.log("SCIM Configuration:");
console.log("Endpoint:", scimEndpoint);
console.log("Token:", scimToken);
```

---

## 📖 Section 7: SSO Best Practices

### 6.10 Security Considerations

**Certificate Validation:**

```typescript
// Always validate SAML signatures
const samlConfig = {
  // ...
  validateInResponseTo: true,
  requestIdExpirationPeriodMs: 3600000, // 1 hour
  cacheProvider: new InMemoryCacheProvider({
    keyExpirationPeriodMs: 3600000,
  }),
  // Require signed assertions
  wantAssertionsSigned: true,
  // Validate certificate
  cert: idpCertificate,
};
```

**Replay Attack Prevention:**

```typescript
// Store and validate SAML request IDs
const requestIdCache = new Map();

function validateRequestId(requestId: string): boolean {
  const now = Date.now();

  // Check if already used
  if (requestIdCache.has(requestId)) {
    return false; // Replay attack
  }

  // Store with expiry
  requestIdCache.set(requestId, now);

  // Clean old entries
  for (const [id, timestamp] of requestIdCache.entries()) {
    if (now - timestamp > 3600000) {
      // 1 hour
      requestIdCache.delete(id);
    }
  }

  return true;
}
```

**Audit Logging:**

```typescript
// Log all SSO events
export async function logSSOEvent(event: {
  tenantId: string;
  userId?: string;
  action: string;
  provider: string;
  success: boolean;
  ipAddress: string;
  userAgent?: string;
  error?: string;
}): Promise {
  await pool.query(
    `INSERT INTO sso_audit_log 
      (tenant_id, user_id, action, provider, success, ip_address, user_agent, error)
     VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`,
    [
      event.tenantId,
      event.userId,
      event.action,
      event.provider,
      event.success,
      event.ipAddress,
      event.userAgent,
      event.error,
    ]
  );
}

// Usage
await logSSOEvent({
  tenantId,
  userId: user.id,
  action: "saml_login",
  provider: "okta",
  success: true,
  ipAddress: req.ip,
  userAgent: req.headers["user-agent"],
});
```

---

### 6.11 User Experience Patterns

**Domain-Based SSO Detection:**

```typescript
// Automatically detect if user should use SSO
export async function detectSSO(email: string): Promise {
  const domain = email.split("@")[1];

  // Check if domain has SSO configured
  const result = await pool.query(
    `SELECT 
      t.id as tenant_id,
      sso.provider_type,
      sso.provider_name
     FROM tenants t
     INNER JOIN sso_configurations sso ON t.id = sso.tenant_id
     WHERE t.domain = $1 AND sso.enabled = TRUE`,
    [domain]
  );

  if (result.rows.length === 0) {
    return { ssoRequired: false };
  }

  const config = result.rows[0];

  return {
    ssoRequired: true,
    ssoUrl: `/saml/${config.tenant_id}/login`,
    provider: config.provider_name,
  };
}

// Login flow
app.post("/api/auth/check-email", async (req, res) => {
  const { email } = req.body;

  const ssoInfo = await detectSSO(email);

  if (ssoInfo.ssoRequired) {
    return res.json({
      method: "sso",
      redirectUrl: ssoInfo.ssoUrl,
      provider: ssoInfo.provider,
    });
  }

  res.json({
    method: "password",
  });
});
```

**SSO Fallback to Password:**

```typescript
// Allow password login even with SSO configured (with admin approval)
export async function allowPasswordFallback(
  email: string,
  password: string
): Promise {
  const user = await User.findByEmail(email);

  if (!user) return false;

  // Check if SSO is enforced
  const tenant = await Tenant.findById(user.tenantId);

  if (tenant.enforceSSOOnly) {
    return false; // No password login allowed
  }

  // Allow password login as fallback
  return await user.verifyPassword(password);
}
```

---

## 📚 Additional Resources

### SSO Standards & Documentation

- **SAML 2.0 Specification:** http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
- **OAuth 2.0:** https://oauth.net/2/
- **OpenID Connect:** https://openid.net/connect/
- **SCIM 2.0:** http://www.simplecloud.info/

### Identity Providers Documentation

- **Okta SAML Setup:** https://developer.okta.com/docs/guides/saml-application-setup/overview/
- **Azure AD SSO:** https://docs.microsoft.com/en-us/azure/active-directory/saas-apps/
- **Google Workspace SSO:** https://support.google.com/a/answer/6087519
- **OneLogin:** https://developers.onelogin.com/
- **Auth0:** https://auth0.com/docs/authenticate/protocols/saml

### Testing Tools

- **SAML Tracer (Browser Extension):** Debug SAML flows
- **SAMLTest.id:** https://samltest.id/ - Test your SAML implementation
- **jwt.io:** Decode and verify JWT tokens

---

## ✅ Day 6 Complete Checklist

By the end of this day, you should have:

- [ ] Understood SAML 2.0 protocol and flow
- [ ] Implemented SAML SSO for Node.js
- [ ] Integrated with Azure AD (Microsoft 365)
- [ ] Integrated with Google Workspace
- [ ] Set up Okta integration
- [ ] Implemented JIT (Just-In-Time) provisioning
- [ ] Built SCIM endpoints for user provisioning
- [ ] Created SSO configuration admin panel
- [ ] Implemented domain-based SSO detection
- [ ] Added audit logging for SSO events
- [ ] Understood security best practices

---

## 💡 Key Takeaways

1. **SAML is the enterprise standard** - Required for B2B SaaS sales

2. **JIT provisioning is essential** - Automatically create users on first login

3. **SCIM enables automation** - Let IdPs manage user lifecycle

4. **Security is critical** - Validate certificates, prevent replays, audit everything

5. **Support multiple providers** - Azure AD, Google, Okta are most common

6. **Test thoroughly** - SSO bugs are hard to debug

7. **Provide clear documentation** - Help customers configure SSO

8. **Plan for fallback** - What happens when SSO is down?

9. **Domain-based detection** - Improve UX by auto-detecting SSO

10. **Audit everything** - Log all SSO events for security and compliance

---

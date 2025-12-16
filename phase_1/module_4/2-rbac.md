# Module 4 - Day 4: Role-Based Access Control (RBAC)

**Part of:** Module 4 - Authentication & Authorization  
**Duration:** Day 4 of 7

---

## 📚 Learning Objectives

By the end of this day, you will:

- Understand RBAC vs ABAC (Attribute-Based Access Control)
- Design effective role and permission systems
- Implement hierarchical roles
- Build fine-grained permission systems
- Handle dynamic permissions at runtime
- Implement resource-level permissions
- Use middleware for access control across all three tech stacks

---

## 🎯 Understanding Access Control Models

### RBAC vs ABAC vs ACL

| Model    | Description                              | Best For                  | Example                                                  |
| -------- | ---------------------------------------- | ------------------------- | -------------------------------------------------------- |
| **RBAC** | Users have roles, roles have permissions | Most SaaS apps            | Admin can delete users                                   |
| **ABAC** | Rules based on attributes                | Complex enterprise        | Manager can approve if department=sales AND amount<10000 |
| **ACL**  | Per-resource permissions                 | File systems, simple apps | User123 can read Document456                             |

**This module focuses on RBAC as it's most common in SaaS applications.**

---

## 📖 Section 1: RBAC Fundamentals

### 4.11 RBAC Concepts

**Core Components:**

```
User → Has → Role → Has → Permissions → On → Resources
```

**Example:**

```
John Doe
  ↓ has role
Admin
  ↓ has permissions
[users.create, users.read, users.update, users.delete]
  ↓ on resources
User Management System
```

---

### 4.12 Simple RBAC (Role-Only)

**Use Case:** Basic SaaS with 3-4 roles

**Roles:**

- `user` - Regular user
- `manager` - Team manager
- `admin` - System administrator
- `super_admin` - Full system access

**Database Schema:**

```sql
-- Simple role in users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  role VARCHAR(50) DEFAULT 'user', -- Simple role column
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_role ON users(role);
```

**Implementation (Node.js):**

```typescript
// src/middleware/rbac.middleware.ts

export function requireRole(role: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: "Authentication required" });
    }

    if (req.user.role !== role) {
      return res.status(403).json({
        error: "Insufficient permissions",
        required: role,
        current: req.user.role,
      });
    }

    next();
  };
}

export function requireRoles(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: "Authentication required" });
    }

    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        error: "Insufficient permissions",
        required: roles,
        current: req.user.role,
      });
    }

    next();
  };
}

// Usage in routes
app.get(
  "/api/admin/users",
  authenticateJWT,
  requireRole("admin"),
  userController.getAllUsers
);

app.get(
  "/api/reports",
  authenticateJWT,
  requireRoles("manager", "admin", "super_admin"),
  reportController.getReports
);
```

**Pros:**

- ✅ Simple to implement
- ✅ Easy to understand
- ✅ Fast authorization checks

**Cons:**

- ❌ Not flexible (adding permissions requires code changes)
- ❌ Difficult to customize per client
- ❌ Role explosion (too many specific roles)

---

### 4.13 Advanced RBAC (Roles + Permissions)

**Use Case:** SaaS with complex permission requirements

**Separation of Concerns:**

- **Roles:** Group users (Admin, Manager, Editor)
- **Permissions:** Specific actions (users.create, posts.delete)
- **Resources:** What the permission applies to (users, posts, comments)

**Database Schema:**

```sql
-- Roles table
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(50) UNIQUE NOT NULL,
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Permissions table
CREATE TABLE permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) UNIQUE NOT NULL, -- e.g., 'users.create'
  description TEXT,
  resource VARCHAR(50), -- e.g., 'users'
  action VARCHAR(50),   -- e.g., 'create'
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Role-Permission mapping (many-to-many)
CREATE TABLE role_permissions (
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
  PRIMARY KEY (role_id, permission_id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User-Role mapping (many-to-many for multiple roles)
CREATE TABLE user_roles (
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  PRIMARY KEY (user_id, role_id),
  granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  granted_by UUID REFERENCES users(id)
);

-- Direct user permissions (override role permissions)
CREATE TABLE user_permissions (
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
  PRIMARY KEY (user_id, permission_id),
  granted BOOLEAN DEFAULT TRUE, -- FALSE to revoke
  granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  granted_by UUID REFERENCES users(id)
);

-- Indexes
CREATE INDEX idx_user_roles_user ON user_roles(user_id);
CREATE INDEX idx_user_roles_role ON user_roles(role_id);
CREATE INDEX idx_role_permissions_role ON role_permissions(role_id);
CREATE INDEX idx_user_permissions_user ON user_permissions(user_id);
```

**Seed Data:**

```sql
-- Insert roles
INSERT INTO roles (id, name, description) VALUES
  ('admin-role-id', 'admin', 'System administrator with full access'),
  ('manager-role-id', 'manager', 'Team manager with limited admin access'),
  ('editor-role-id', 'editor', 'Content editor'),
  ('user-role-id', 'user', 'Regular user');

-- Insert permissions
INSERT INTO permissions (name, resource, action, description) VALUES
  -- User permissions
  ('users.create', 'users', 'create', 'Create new users'),
  ('users.read', 'users', 'read', 'View user information'),
  ('users.update', 'users', 'update', 'Update user information'),
  ('users.delete', 'users', 'delete', 'Delete users'),

  -- Post permissions
  ('posts.create', 'posts', 'create', 'Create new posts'),
  ('posts.read', 'posts', 'read', 'View posts'),
  ('posts.update', 'posts', 'update', 'Update posts'),
  ('posts.delete', 'posts', 'delete', 'Delete posts'),
  ('posts.publish', 'posts', 'publish', 'Publish posts'),

  -- Comment permissions
  ('comments.create', 'comments', 'create', 'Create comments'),
  ('comments.read', 'comments', 'read', 'View comments'),
  ('comments.moderate', 'comments', 'moderate', 'Moderate comments'),

  -- Settings permissions
  ('settings.read', 'settings', 'read', 'View system settings'),
  ('settings.update', 'settings', 'update', 'Update system settings');

-- Assign permissions to roles
-- Admin gets all permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 'admin-role-id', id FROM permissions;

-- Manager gets user and post management
INSERT INTO role_permissions (role_id, permission_id)
SELECT 'manager-role-id', id FROM permissions
WHERE name IN (
  'users.read', 'users.update',
  'posts.read', 'posts.update', 'posts.delete', 'posts.publish',
  'comments.moderate'
);

-- Editor gets post management
INSERT INTO role_permissions (role_id, permission_id)
SELECT 'editor-role-id', id FROM permissions
WHERE name IN (
  'posts.create', 'posts.read', 'posts.update',
  'comments.read', 'comments.moderate'
);

-- User gets basic read permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 'user-role-id', id FROM permissions
WHERE name IN (
  'posts.read', 'comments.create', 'comments.read'
);
```

---

## 📖 Section 2: Implementation Across Tech Stacks

### 4.14 Node.js/TypeScript Implementation

**Permission Service:**

```typescript
// src/services/permission.service.ts
import { pool } from "../config/database";

export class PermissionService {
  /**
   * Get all permissions for a user (from roles + direct permissions)
   */
  static async getUserPermissions(userId: string): Promise<string[]> {
    const query = `
      -- Permissions from roles
      SELECT DISTINCT p.name
      FROM permissions p
      INNER JOIN role_permissions rp ON p.id = rp.permission_id
      INNER JOIN user_roles ur ON rp.role_id = ur.role_id
      WHERE ur.user_id = $1
      
      UNION
      
      -- Direct user permissions (granted)
      SELECT DISTINCT p.name
      FROM permissions p
      INNER JOIN user_permissions up ON p.id = up.permission_id
      WHERE up.user_id = $1 AND up.granted = TRUE
      
      EXCEPT
      
      -- Revoked direct permissions
      SELECT DISTINCT p.name
      FROM permissions p
      INNER JOIN user_permissions up ON p.id = up.permission_id
      WHERE up.user_id = $1 AND up.granted = FALSE
    `;

    const result = await pool.query(query, [userId]);
    return result.rows.map((row) => row.name);
  }

  /**
   * Check if user has specific permission
   */
  static async userHasPermission(
    userId: string,
    permission: string
  ): Promise<boolean> {
    const permissions = await this.getUserPermissions(userId);
    return permissions.includes(permission);
  }

  /**
   * Check if user has any of the permissions
   */
  static async userHasAnyPermission(
    userId: string,
    permissions: string[]
  ): Promise<boolean> {
    const userPermissions = await this.getUserPermissions(userId);
    return permissions.some((p) => userPermissions.includes(p));
  }

  /**
   * Check if user has all permissions
   */
  static async userHasAllPermissions(
    userId: string,
    permissions: string[]
  ): Promise<boolean> {
    const userPermissions = await this.getUserPermissions(userId);
    return permissions.every((p) => userPermissions.includes(p));
  }

  /**
   * Get user roles
   */
  static async getUserRoles(userId: string): Promise<string[]> {
    const query = `
      SELECT r.name
      FROM roles r
      INNER JOIN user_roles ur ON r.id = ur.role_id
      WHERE ur.user_id = $1
    `;

    const result = await pool.query(query, [userId]);
    return result.rows.map((row) => row.name);
  }

  /**
   * Assign role to user
   */
  static async assignRole(
    userId: string,
    roleName: string,
    grantedBy: string
  ): Promise<void> {
    const query = `
      INSERT INTO user_roles (user_id, role_id, granted_by)
      SELECT $1, id, $3
      FROM roles
      WHERE name = $2
      ON CONFLICT (user_id, role_id) DO NOTHING
    `;

    await pool.query(query, [userId, roleName, grantedBy]);
  }

  /**
   * Remove role from user
   */
  static async removeRole(userId: string, roleName: string): Promise<void> {
    const query = `
      DELETE FROM user_roles
      WHERE user_id = $1
      AND role_id = (SELECT id FROM roles WHERE name = $2)
    `;

    await pool.query(query, [userId, roleName]);
  }

  /**
   * Grant permission directly to user
   */
  static async grantPermission(
    userId: string,
    permissionName: string,
    grantedBy: string
  ): Promise<void> {
    const query = `
      INSERT INTO user_permissions (user_id, permission_id, granted, granted_by)
      SELECT $1, id, TRUE, $3
      FROM permissions
      WHERE name = $2
      ON CONFLICT (user_id, permission_id) 
      DO UPDATE SET granted = TRUE, granted_at = CURRENT_TIMESTAMP
    `;

    await pool.query(query, [userId, permissionName, grantedBy]);
  }

  /**
   * Revoke permission from user
   */
  static async revokePermission(
    userId: string,
    permissionName: string
  ): Promise<void> {
    const query = `
      INSERT INTO user_permissions (user_id, permission_id, granted)
      SELECT $1, id, FALSE
      FROM permissions
      WHERE name = $2
      ON CONFLICT (user_id, permission_id) 
      DO UPDATE SET granted = FALSE, granted_at = CURRENT_TIMESTAMP
    `;

    await pool.query(query, [userId, permissionName]);
  }
}
```

**Permission Middleware:**

```typescript
// src/middleware/permission.middleware.ts
import { Request, Response, NextFunction } from "express";
import { PermissionService } from "../services/permission.service";

// Cache permissions for the request lifecycle
interface RequestWithPermissions extends Request {
  permissions?: string[];
}

/**
 * Load user permissions into request object
 */
export async function loadPermissions(
  req: RequestWithPermissions,
  res: Response,
  next: NextFunction
) {
  if (!req.user?.userId) {
    return next();
  }

  try {
    req.permissions = await PermissionService.getUserPermissions(
      req.user.userId
    );
    next();
  } catch (error) {
    console.error("Error loading permissions:", error);
    next(error);
  }
}

/**
 * Require specific permission
 */
export function requirePermission(permission: string) {
  return async (
    req: RequestWithPermissions,
    res: Response,
    next: NextFunction
  ) => {
    if (!req.user) {
      return res.status(401).json({ error: "Authentication required" });
    }

    try {
      // Use cached permissions if available
      let hasPermission: boolean;

      if (req.permissions) {
        hasPermission = req.permissions.includes(permission);
      } else {
        hasPermission = await PermissionService.userHasPermission(
          req.user.userId,
          permission
        );
      }

      if (!hasPermission) {
        return res.status(403).json({
          error: "Insufficient permissions",
          required: permission,
        });
      }

      next();
    } catch (error) {
      console.error("Permission check error:", error);
      res.status(500).json({ error: "Permission check failed" });
    }
  };
}

/**
 * Require any of the permissions (OR)
 */
export function requireAnyPermission(...permissions: string[]) {
  return async (
    req: RequestWithPermissions,
    res: Response,
    next: NextFunction
  ) => {
    if (!req.user) {
      return res.status(401).json({ error: "Authentication required" });
    }

    try {
      let hasPermission: boolean;

      if (req.permissions) {
        hasPermission = permissions.some((p) => req.permissions!.includes(p));
      } else {
        hasPermission = await PermissionService.userHasAnyPermission(
          req.user.userId,
          permissions
        );
      }

      if (!hasPermission) {
        return res.status(403).json({
          error: "Insufficient permissions",
          required: `Any of: ${permissions.join(", ")}`,
        });
      }

      next();
    } catch (error) {
      console.error("Permission check error:", error);
      res.status(500).json({ error: "Permission check failed" });
    }
  };
}

/**
 * Require all permissions (AND)
 */
export function requireAllPermissions(...permissions: string[]) {
  return async (
    req: RequestWithPermissions,
    res: Response,
    next: NextFunction
  ) => {
    if (!req.user) {
      return res.status(401).json({ error: "Authentication required" });
    }

    try {
      let hasAllPermissions: boolean;

      if (req.permissions) {
        hasAllPermissions = permissions.every((p) =>
          req.permissions!.includes(p)
        );
      } else {
        hasAllPermissions = await PermissionService.userHasAllPermissions(
          req.user.userId,
          permissions
        );
      }

      if (!hasAllPermissions) {
        return res.status(403).json({
          error: "Insufficient permissions",
          required: `All of: ${permissions.join(", ")}`,
        });
      }

      next();
    } catch (error) {
      console.error("Permission check error:", error);
      res.status(500).json({ error: "Permission check failed" });
    }
  };
}

/**
 * Optional permission check (doesn't fail, just adds flag)
 */
export function checkPermission(permission: string) {
  return async (
    req: RequestWithPermissions,
    res: Response,
    next: NextFunction
  ) => {
    if (!req.user) {
      (req as any).hasPermission = false;
      return next();
    }

    try {
      const hasPermission = req.permissions
        ? req.permissions.includes(permission)
        : await PermissionService.userHasPermission(
            req.user.userId,
            permission
          );

      (req as any).hasPermission = hasPermission;
      next();
    } catch (error) {
      (req as any).hasPermission = false;
      next();
    }
  };
}
```

**Usage in Routes:**

```typescript
// src/routes/user.routes.ts
import express from "express";
import { authenticateJWT } from "../middleware/jwt-auth.middleware";
import {
  loadPermissions,
  requirePermission,
  requireAnyPermission,
} from "../middleware/permission.middleware";
import { UserController } from "../controllers/user.controller";

const router = express.Router();

// Apply authentication and load permissions for all routes
router.use(authenticateJWT);
router.use(loadPermissions);

// Require specific permission
router.post(
  "/users",
  requirePermission("users.create"),
  UserController.createUser
);

router.get(
  "/users/:id",
  requirePermission("users.read"),
  UserController.getUser
);

router.put(
  "/users/:id",
  requirePermission("users.update"),
  UserController.updateUser
);

router.delete(
  "/users/:id",
  requirePermission("users.delete"),
  UserController.deleteUser
);

// Require any of multiple permissions
router.get(
  "/users",
  requireAnyPermission("users.read", "users.update"),
  UserController.listUsers
);

export default router;
```

**Permission Management Controller:**

```typescript
// src/controllers/permission.controller.ts
import { Request, Response } from "express";
import { PermissionService } from "../services/permission.service";

export class PermissionController {
  /**
   * Get user's permissions
   */
  static async getUserPermissions(req: Request, res: Response) {
    try {
      const { userId } = req.params;

      const permissions = await PermissionService.getUserPermissions(userId);
      const roles = await PermissionService.getUserRoles(userId);

      res.json({
        userId,
        roles,
        permissions,
      });
    } catch (error) {
      console.error("Error fetching permissions:", error);
      res.status(500).json({ error: "Failed to fetch permissions" });
    }
  }

  /**
   * Assign role to user
   */
  static async assignRole(req: Request, res: Response) {
    try {
      const { userId } = req.params;
      const { roleName } = req.body;
      const grantedBy = req.user!.userId;

      await PermissionService.assignRole(userId, roleName, grantedBy);

      res.json({
        message: "Role assigned successfully",
        userId,
        roleName,
      });
    } catch (error) {
      console.error("Error assigning role:", error);
      res.status(500).json({ error: "Failed to assign role" });
    }
  }

  /**
   * Remove role from user
   */
  static async removeRole(req: Request, res: Response) {
    try {
      const { userId, roleName } = req.params;

      await PermissionService.removeRole(userId, roleName);

      res.json({
        message: "Role removed successfully",
        userId,
        roleName,
      });
    } catch (error) {
      console.error("Error removing role:", error);
      res.status(500).json({ error: "Failed to remove role" });
    }
  }

  /**
   * Grant permission to user
   */
  static async grantPermission(req: Request, res: Response) {
    try {
      const { userId } = req.params;
      const { permissionName } = req.body;
      const grantedBy = req.user!.userId;

      await PermissionService.grantPermission(
        userId,
        permissionName,
        grantedBy
      );

      res.json({
        message: "Permission granted successfully",
        userId,
        permissionName,
      });
    } catch (error) {
      console.error("Error granting permission:", error);
      res.status(500).json({ error: "Failed to grant permission" });
    }
  }

  /**
   * Revoke permission from user
   */
  static async revokePermission(req: Request, res: Response) {
    try {
      const { userId, permissionName } = req.params;

      await PermissionService.revokePermission(userId, permissionName);

      res.json({
        message: "Permission revoked successfully",
        userId,
        permissionName,
      });
    } catch (error) {
      console.error("Error revoking permission:", error);
      res.status(500).json({ error: "Failed to revoke permission" });
    }
  }

  /**
   * List all available roles
   */
  static async listRoles(req: Request, res: Response) {
    try {
      const query = "SELECT * FROM roles ORDER BY name";
      const result = await pool.query(query);

      res.json({ roles: result.rows });
    } catch (error) {
      console.error("Error listing roles:", error);
      res.status(500).json({ error: "Failed to list roles" });
    }
  }

  /**
   * List all available permissions
   */
  static async listPermissions(req: Request, res: Response) {
    try {
      const query = "SELECT * FROM permissions ORDER BY resource, action";
      const result = await pool.query(query);

      res.json({ permissions: result.rows });
    } catch (error) {
      console.error("Error listing permissions:", error);
      res.status(500).json({ error: "Failed to list permissions" });
    }
  }
}
```

**Permission Routes:**

```typescript
// src/routes/permission.routes.ts
import express from "express";
import { authenticateJWT } from "../middleware/jwt-auth.middleware";
import { requirePermission } from "../middleware/permission.middleware";
import { PermissionController } from "../controllers/permission.controller";

const router = express.Router();

router.use(authenticateJWT);

// Get user permissions
router.get(
  "/users/:userId/permissions",
  requirePermission("users.read"),
  PermissionController.getUserPermissions
);

// Assign role
router.post(
  "/users/:userId/roles",
  requirePermission("users.update"),
  PermissionController.assignRole
);

// Remove role
router.delete(
  "/users/:userId/roles/:roleName",
  requirePermission("users.update"),
  PermissionController.removeRole
);

// Grant permission
router.post(
  "/users/:userId/permissions",
  requirePermission("users.update"),
  PermissionController.grantPermission
);

// Revoke permission
router.delete(
  "/users/:userId/permissions/:permissionName",
  requirePermission("users.update"),
  PermissionController.revokePermission
);

// List roles and permissions
router.get(
  "/roles",
  requirePermission("users.read"),
  PermissionController.listRoles
);

router.get(
  "/permissions",
  requirePermission("users.read"),
  PermissionController.listPermissions
);

export default router;
```

---

### 4.15 Java/Spring Boot Implementation

**Permission Models:**

```java
// Role.java
package com.mycompany.saas.model;

import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.Set;

@Entity
@Table(name = "roles")
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(unique = true, nullable = false)
    private String name;

    private String description;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    private Set<Permission> permissions;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    // Getters and setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public Set<Permission> getPermissions() { return permissions; }
    public void setPermissions(Set<Permission> permissions) { this.permissions = permissions; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
}

// Permission.java
package com.mycompany.saas.model;

import javax.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "permissions")
public class Permission {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(unique = true, nullable = false)
    private String name;  // e.g., "users.create"

    private String description;
    private String resource;  // e.g., "users"
    private String action;    // e.g., "create"

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    // Getters and setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public String getResource() { return resource; }
    public void setResource(String resource) { this.resource = resource; }

    public String getAction() { return action; }
    public void setAction(String action) { this.action = action; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
}

// User.java (updated with roles)
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    // ... other fields

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles;

    // Get all permissions from all roles
    public Set<String> getPermissions() {
        Set<String> permissions = new HashSet<>();
        for (Role role : roles) {
            for (Permission permission : role.getPermissions()) {
                permissions.add(permission.getName());
            }
        }
        return permissions;
    }

    public boolean hasPermission(String permissionName) {
        return getPermissions().contains(permissionName);
    }

    public boolean hasAnyPermission(String... permissionNames) {
        Set<String> userPermissions = getPermissions();
        return Arrays.stream(permissionNames)
                .anyMatch(userPermissions::contains);
    }

    public boolean hasAllPermissions(String... permissionNames) {
        Set<String> userPermissions = getPermissions();
        return Arrays.stream(permissionNames)
                .allMatch(userPermissions::contains);
    }

    // Getters and setters
}
```

**Permission Service:**

```java
// PermissionService.java
package com.mycompany.saas.service;

import com.mycompany.saas.model.Permission;
import com.mycompany.saas.model.Role;
import com.mycompany.saas.model.User;
import com.mycompany.saas.repository.PermissionRepository;
import com.mycompany.saas.repository.RoleRepository;
import com.mycompany.saas.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Set;
import java.util.stream.Collectors;

@Service
public class PermissionService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private RoleRepository roleRepository;

    @Autowired
    private PermissionRepository permissionRepository;

    /**
     * Get all permissions for a user
     */
    public Set<String> getUserPermissions(String userId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

        return user.getPermissions();
    }

    /**
     * Check if user has specific permission
     */
    public boolean userHasPermission(String userId, String permissionName) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

        return user.hasPermission(permissionName);
    }

    /**
     * Get user roles
     */
    public Set<String> getUserRoles(String userId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

        return user.getRoles().stream()
                .map(Role::getName)
                .collect(Collectors.toSet());
    }

    /**
     * Assign role to user
     */
    @Transactional
    public void assignRole(String userId, String roleName) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

        Role role = roleRepository.findByName(roleName)
                .orElseThrow(() -> new RuntimeException("Role not found"));

        user.getRoles().add(role);
        userRepository.save(user);
    }

    /**
     * Remove role from user
     */
    @Transactional
    public void removeRole(String userId, String roleName) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

        Role role = roleRepository.findByName(roleName)
                .orElseThrow(() -> new RuntimeException("Role not found"));

        user.getRoles().remove(role);
        userRepository.save(user);
    }

    /**
     * Get all available roles
     */
    public List<Role> getAllRoles() {
        return roleRepository.findAll();
    }

    /**
     * Get all available permissions
     */
    public List<Permission> getAllPermissions() {
        return permissionRepository.findAll();
    }
}
```

**Custom Security Annotations:**

```java
// HasPermission.java
package com.mycompany.saas.security.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface HasPermission {
    String value();
}

// HasAnyPermission.java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface HasAnyPermission {
    String[] value();
}

// HasAllPermissions.java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface HasAllPermissions {
    String[] value();
}
```

**Permission Aspect (AOP):**

```java
// PermissionAspect.java
package com.mycompany.saas.security.aspect;

import com.mycompany.saas.security.annotation.HasPermission;
import com.mycompany.saas.security.annotation.HasAnyPermission;
import com.mycompany.saas.security.annotation.HasAllPermissions;
import com.mycompany.saas.service.PermissionService;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.Set;

@Aspect
@Component
public class PermissionAspect {

    @Autowired
    private PermissionService permissionService;

    @Around("@annotation(hasPermission)")
    public Object checkPermission(ProceedingJoinPoint joinPoint, HasPermission hasPermission) throws Throwable {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth == null || !auth.isAuthenticated()) {
            throw new SecurityException("Authentication required");
        }

        String userId = (String) auth.getPrincipal();
        String requiredPermission = hasPermission.value();

        if (!permissionService.userHasPermission(userId, requiredPermission)) {
            throw new SecurityException("Insufficient permissions. Required: " + requiredPermission);
        }

        return joinPoint.proceed();
    }

    @Around("@annotation(hasAnyPermission)")
    public Object checkAnyPermission(ProceedingJoinPoint joinPoint, HasAnyPermission hasAnyPermission) throws Throwable {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth == null || !auth.isAuthenticated()) {
            throw new SecurityException("Authentication required");
        }

        String userId = (String) auth.getPrincipal();
        String[] requiredPermissions = hasAnyPermission.value();
        Set userPermissions = permissionService.getUserPermissions(userId);

        boolean hasPermission = Arrays.stream(requiredPermissions)
                .anyMatch(userPermissions::contains);

        if (!hasPermission) {
            throw new SecurityException("Insufficient permissions. Required any of: " +
                    Arrays.toString(requiredPermissions));
        }

        return joinPoint.proceed();
    }

    @Around("@annotation(hasAllPermissions)")
    public Object checkAllPermissions(ProceedingJoinPoint joinPoint, HasAllPermissions hasAllPermissions) throws Throwable {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth == null || !auth.isAuthenticated()) {
            throw new SecurityException("Authentication required");
        }

        String userId = (String) auth.getPrincipal();
        String[] requiredPermissions = hasAllPermissions.value();
        Set userPermissions = permissionService.getUserPermissions(userId);

        boolean hasAllPermissions = Arrays.stream(requiredPermissions)
                .allMatch(userPermissions::contains);

        if (!hasAllPermissions) {
            throw new SecurityException("Insufficient permissions. Required all of: " +
                    Arrays.toString(requiredPermissions));
        }

        return joinPoint.proceed();
    }
}
```

**Usage in Controllers:**

```java
// UserController.java
package com.mycompany.saas.controller;

import com.mycompany.saas.security.annotation.HasPermission;
import com.mycompany.saas.security.annotation.HasAnyPermission;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping
    @HasPermission("users.read")
    public ResponseEntity listUsers() {
        // List users logic
        return ResponseEntity.ok().build();
    }

    @PostMapping
    @HasPermission("users.create")
    public ResponseEntity createUser(@RequestBody UserDTO userDTO) {
        // Create user logic
        return ResponseEntity.ok().build();
    }

    @PutMapping("/{id}")
    @HasPermission("users.update")
    public ResponseEntity updateUser(@PathVariable String id, @RequestBody UserDTO userDTO) {
        // Update user logic
        return ResponseEntity.ok().build();
    }

    @DeleteMapping("/{id}")
    @HasPermission("users.delete")
    public ResponseEntity deleteUser(@PathVariable String id) {
        // Delete user logic
        return ResponseEntity.ok().build();
    }

    @GetMapping("/reports")
    @HasAnyPermission({"reports.read", "reports.admin"})
    public ResponseEntity getReports() {
        // Get reports logic
        return ResponseEntity.ok().build();
    }
}
```

**Permission Management Controller:**

```java
// PermissionController.java
package com.mycompany.saas.controller;

import com.mycompany.saas.security.annotation.HasPermission;
import com.mycompany.saas.service.PermissionService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/permissions")
public class PermissionController {

    @Autowired
    private PermissionService permissionService;

    @GetMapping("/users/{userId}")
    @HasPermission("users.read")
    public ResponseEntity getUserPermissions(@PathVariable String userId) {
        return ResponseEntity.ok(Map.of(
            "userId", userId,
            "roles", permissionService.getUserRoles(userId),
            "permissions", permissionService.getUserPermissions(userId)
        ));
    }

    @PostMapping("/users/{userId}/roles")
    @HasPermission("users.update")
    public ResponseEntity assignRole(
            @PathVariable String userId,
            @RequestBody Map body) {
        String roleName = body.get("roleName");
        permissionService.assignRole(userId, roleName);
        return ResponseEntity.ok(Map.of("message", "Role assigned successfully"));
    }

    @DeleteMapping("/users/{userId}/roles/{roleName}")
    @HasPermission("users.update")
    public ResponseEntity removeRole(
            @PathVariable String userId,
            @PathVariable String roleName) {
        permissionService.removeRole(userId, roleName);
        return ResponseEntity.ok(Map.of("message", "Role removed successfully"));
    }

    @GetMapping("/roles")
    @HasPermission("users.read")
    public ResponseEntity listRoles() {
        return ResponseEntity.ok(Map.of("roles", permissionService.getAllRoles()));
    }

    @GetMapping("/list")
    @HasPermission("users.read")
    public ResponseEntity listPermissions() {
        return ResponseEntity.ok(Map.of("permissions", permissionService.getAllPermissions()));
    }
}
```

---

### 4.16 Go Implementation

**Models:**

```go
// models/permission.go
package models

import "time"

type Role struct {
	ID          string    `json:"id" db:"id"`
	Name        string    `json:"name" db:"name"`
	Description string    `json:"description" db:"description"`
	CreatedAt   time.Time `json:"created_at" db:"created_at"`
}

type Permission struct {
	ID          string    `json:"id" db:"id"`
	Name        string    `json:"name" db:"name"`
	Description string    `json:"description" db:"description"`
	Resource    string    `json:"resource" db:"resource"`
	Action      string    `json:"action" db:"action"`
	CreatedAt   time.Time `json:"created_at" db:"created_at"`
}

type UserRole struct {
	UserID    string    `db:"user_id"`
	RoleID    string    `db:"role_id"`
	GrantedAt time.Time `db:"granted_at"`
	GrantedBy string    `db:"granted_by"`
}
```

**Permission Service:**

```go
// services/permission.go
package services

import (
	"database/sql"
	"fmt"

	"myapp/models"
)

type PermissionService struct {
	db *sql.DB
}

func NewPermissionService(db *sql.DB) *PermissionService {
	return &PermissionService{db: db}
}

// GetUserPermissions gets all permissions for a user
func (s *PermissionService) GetUserPermissions(userID string) ([]string, error) {
	query := `
		-- Permissions from roles
		SELECT DISTINCT p.name
		FROM permissions p
		INNER JOIN role_permissions rp ON p.id = rp.permission_id
		INNER JOIN user_roles ur ON rp.role_id = ur.role_id
		WHERE ur.user_id = $1

		UNION

		-- Direct user permissions (granted)
		SELECT DISTINCT p.name
		FROM permissions p
		INNER JOIN user_permissions up ON p.id = up.permission_id
		WHERE up.user_id = $1 AND up.granted = TRUE

		EXCEPT

		-- Revoked direct permissions
		SELECT DISTINCT p.name
		FROM permissions p
		INNER JOIN user_permissions up ON p.id = up.permission_id
		WHERE up.user_id = $1 AND up.granted = FALSE
	`

	rows, err := s.db.Query(query, userID)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var permissions []string
	for rows.Next() {
		var permission string
		if err := rows.Scan(&permission); err != nil {
			return nil, err
		}
		permissions = append(permissions, permission)
	}

	return permissions, nil
}

// UserHasPermission checks if user has specific permission
func (s *PermissionService) UserHasPermission(userID, permission string) (bool, error) {
	permissions, err := s.GetUserPermissions(userID)
	if err != nil {
		return false, err
	}

	for _, p := range permissions {
		if p == permission {
			return true, nil
		}
	}

	return false, nil
}

// UserHasAnyPermission checks if user has any of the permissions
func (s *PermissionService) UserHasAnyPermission(userID string, permissions []string) (bool, error) {
	userPermissions, err := s.GetUserPermissions(userID)
	if err != nil {
		return false, err
	}

	permMap := make(map[string]bool)
	for _, p := range userPermissions {
		permMap[p] = true
	}

	for _, p := range permissions {
		if permMap[p] {
			return true, nil
		}
	}

	return false, nil
}

// UserHasAllPermissions checks if user has all permissions
func (s *PermissionService) UserHasAllPermissions(userID string, permissions []string) (bool, error) {
	userPermissions, err := s.GetUserPermissions(userID)
	if err != nil {
		return false, err
	}

	permMap := make(map[string]bool)
	for _, p := range userPermissions {
		permMap[p] = true
	}

	for _, p := range permissions {
		if !permMap[p] {
			return false, nil
		}
	}

	return true, nil
}

// GetUserRoles gets all roles for a user
func (s *PermissionService) GetUserRoles(userID string) ([]string, error) {
	query := `
		SELECT r.name
		FROM roles r
		INNER JOIN user_roles ur ON r.id = ur.role_id
		WHERE ur.user_id = $1
	`

	rows, err := s.db.Query(query, userID)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var roles []string
	for rows.Next() {
		var role string
		if err := rows.Scan(&role); err != nil {
			return nil, err
		}
		roles = append(roles, role)
	}

	return roles, nil
}

// AssignRole assigns a role to user
func (s *PermissionService) AssignRole(userID, roleName, grantedBy string) error {
	query := `
		INSERT INTO user_roles (user_id, role_id, granted_by)
		SELECT $1, id, $3
		FROM roles
		WHERE name = $2
		ON CONFLICT (user_id, role_id) DO NOTHING
	`

	_, err := s.db.Exec(query, userID, roleName, grantedBy)
	return err
}

// RemoveRole removes a role from user
func (s *PermissionService) RemoveRole(userID, roleName string) error {
	query := `
		DELETE FROM user_roles
		WHERE user_id = $1
		AND role_id = (SELECT id FROM roles WHERE name = $2)
	`

	_, err := s.db.Exec(query, userID, roleName)
	return err
}

// GetAllRoles gets all available roles
func (s *PermissionService) GetAllRoles() ([]models.Role, error) {
	query := `SELECT id, name, description, created_at FROM roles ORDER BY name`

	rows, err := s.db.Query(query)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var roles []models.Role
	for rows.Next() {
		var role models.Role
		if err := rows.Scan(&role.ID, &role.Name, &role.Description, &role.CreatedAt); err != nil {
			return nil, err
		}
		roles = append(roles, role)
	}

	return roles, nil
}

// GetAllPermissions gets all available permissions
func (s *PermissionService) GetAllPermissions() ([]models.Permission, error) {
	query := `SELECT id, name, description, resource, action, created_at FROM permissions ORDER BY resource, action`

	rows, err := s.db.Query(query)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var permissions []models.Permission
	for rows.Next() {
		var perm models.Permission
		if err := rows.Scan(&perm.ID, &perm.Name, &perm.Description, &perm.Resource, &perm.Action, &perm.CreatedAt); err != nil {
			return nil, err
		}
		permissions = append(permissions, perm)
	}

	return permissions, nil
}
```

**Permission Middleware:**

```go
// middleware/permission.go
package middleware

import (
	"context"
	"net/http"

	"myapp/services"
)

type PermissionMiddleware struct {
	permissionService *services.PermissionService
}

func NewPermissionMiddleware(permissionService *services.PermissionService) *PermissionMiddleware {
	return &PermissionMiddleware{
		permissionService: permissionService,
	}
}

// RequirePermission middleware
func (m *PermissionMiddleware) RequirePermission(permission string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			userID, ok := r.Context().Value(UserIDKey).(string)
			if !ok {
				http.Error(w, "Authentication required", http.StatusUnauthorized)
				return
			}

			hasPermission, err := m.permissionService.UserHasPermission(userID, permission)
			if err != nil {
				http.Error(w, "Permission check failed", http.StatusInternalServerError)
				return
			}

			if !hasPermission {
				http.Error(w, "Insufficient permissions", http.StatusForbidden)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// RequireAnyPermission middleware
func (m *PermissionMiddleware) RequireAnyPermission(permissions ...string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			userID, ok := r.Context().Value(UserIDKey).(string)
			if !ok {
				http.Error(w, "Authentication required", http.StatusUnauthorized)
				return
			}

			hasPermission, err := m.permissionService.UserHasAnyPermission(userID, permissions)
			if err != nil {
				http.Error(w, "Permission check failed", http.StatusInternalServerError)
				return
			}

			if !hasPermission {
				http.Error(w, "Insufficient permissions", http.StatusForbidden)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// RequireAllPermissions middleware
func (m *PermissionMiddleware) RequireAllPermissions(permissions ...string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			userID, ok := r.Context().Value(UserIDKey).(string)
			if !ok {
				http.Error(w, "Authentication required", http.StatusUnauthorized)
				return
			}

			hasAllPermissions, err := m.permissionService.UserHasAllPermissions(userID, permissions)
			if err != nil {
				http.Error(w, "Permission check failed", http.StatusInternalServerError)
				return
			}

			if !hasAllPermissions {
				http.Error(w, "Insufficient permissions", http.StatusForbidden)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// LoadPermissions loads user permissions into context
func (m *PermissionMiddleware) LoadPermissions(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		userID, ok := r.Context().Value(UserIDKey).(string)
		if !ok {
			next.ServeHTTP(w, r)
			return
		}

		permissions, err := m.permissionService.GetUserPermissions(userID)
		if err != nil {
			next.ServeHTTP(w, r)
			return
		}

		ctx := context.WithValue(r.Context(), PermissionsKey, permissions)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

**Permission Handler:**

```go
// handlers/permission.go
package handlers

import (
	"encoding/json"
	"net/http"

	"github.com/gorilla/mux"
	"myapp/middleware"
	"myapp/services"
)

type PermissionHandler struct {
	permissionService *services.PermissionService
}

func NewPermissionHandler(permissionService *services.PermissionService) *PermissionHandler {
	return &PermissionHandler{
		permissionService: permissionService,
	}
}

func (h *PermissionHandler) GetUserPermissions(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	userID := vars["userId"]

	permissions, err := h.permissionService.GetUserPermissions(userID)
	if err != nil {
		http.Error(w, "Failed to get permissions", http.StatusInternalServerError)
		return
	}

	roles, err := h.permissionService.GetUserRoles(userID)
	if err != nil {
		http.Error(w, "Failed to get roles", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"userId":      userID,
		"roles":       roles,
		"permissions": permissions,
	})
}

func (h *PermissionHandler) AssignRole(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	userID := vars["userId"]

	var req struct {
		RoleName string `json:"roleName"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	grantedBy := r.Context().Value(middleware.UserIDKey).(string)

	if err := h.permissionService.AssignRole(userID, req.RoleName, grantedBy); err != nil {
		http.Error(w, "Failed to assign role", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{
		"message": "Role assigned successfully",
	})
}

func (h *PermissionHandler) RemoveRole(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	userID := vars["userId"]
	roleName := vars["roleName"]

	if err := h.permissionService.RemoveRole(userID, roleName); err != nil {
		http.Error(w, "Failed to remove role", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{
		"message": "Role removed successfully",
	})
}

func (h *PermissionHandler) ListRoles(w http.ResponseWriter, r *http.Request) {
	roles, err := h.permissionService.GetAllRoles()
	if err != nil {
		http.Error(w, "Failed to list roles", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"roles": roles,
	})
}

func (h *PermissionHandler) ListPermissions(w http.ResponseWriter, r *http.Request) {
	permissions, err := h.permissionService.GetAllPermissions()
	if err != nil {
		http.Error(w, "Failed to list permissions", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"permissions": permissions,
	})
}
```

**Router Setup:**

```go
// main.go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/mux"
	"myapp/handlers"
	"myapp/middleware"
	"myapp/services"
)

func main() {
	// Initialize services
	db := initDB()
	permissionService := services.NewPermissionService(db)

	// Initialize middleware
	jwtMiddleware := middleware.NewJWTMiddleware()
	permMiddleware := middleware.NewPermissionMiddleware(permissionService)

	// Initialize handlers
	permHandler := handlers.NewPermissionHandler(permissionService)
	userHandler := handlers.NewUserHandler()

	// Setup router
	r := mux.NewRouter()

	// API routes with permissions
	api := r.PathPrefix("/api").Subrouter()
	api.Use(jwtMiddleware.Authenticate)
	api.Use(permMiddleware.LoadPermissions)

	// User routes
	users := api.PathPrefix("/users").Subrouter()
	users.HandleFunc("", userHandler.ListUsers).
		Methods("GET").
		Handler(permMiddleware.RequirePermission("users.read")(http.HandlerFunc(userHandler.ListUsers)))

	users.HandleFunc("", userHandler.CreateUser).
		Methods("POST").
		Handler(permMiddleware.RequirePermission("users.create")(http.HandlerFunc(userHandler.CreateUser)))

	users.HandleFunc("/{id}", userHandler.UpdateUser).
		Methods("PUT").
		Handler(permMiddleware.RequirePermission("users.update")(http.HandlerFunc(userHandler.UpdateUser)))

	users.HandleFunc("/{id}", userHandler.DeleteUser).
		Methods("DELETE").
		Handler(permMiddleware.RequirePermission("users.delete")(http.HandlerFunc(userHandler.DeleteUser)))

	// Permission management routes
	perms := api.PathPrefix("/permissions").Subrouter()
	perms.Use(permMiddleware.RequirePermission("users.read"))

	perms.HandleFunc("/users/{userId}", permHandler.GetUserPermissions).Methods("GET")
	perms.HandleFunc("/users/{userId}/roles", permHandler.AssignRole).Methods("POST")
	perms.HandleFunc("/users/{userId}/roles/{roleName}", permHandler.RemoveRole).Methods("DELETE")
	perms.HandleFunc("/roles", permHandler.ListRoles).Methods("GET")
	perms.HandleFunc("/list", permHandler.ListPermissions).Methods("GET")

	log.Println("Server starting on :8080")
	log.Fatal(http.ListenAndServe(":8080", r))
}
```

---

## 📖 Section 3: Resource-Level Permissions

### 4.17 Ownership-Based Access Control

**Scenario:** Users can only edit their own posts

**Database Schema:**

```sql
CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(255) NOT NULL,
  content TEXT,
  author_id UUID REFERENCES users(id) ON DELETE CASCADE,
  status VARCHAR(50) DEFAULT 'draft',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_posts_author ON posts(author_id);
```

**Implementation (Node.js):**

```typescript
// src/middleware/ownership.middleware.ts
import { Request, Response, NextFunction } from "express";
import { Post } from "../models/post.model";

export function requireOwnership(resourceType: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userId = req.user!.userId;
    const resourceId = req.params.id;

    try {
      let isOwner = false;

      switch (resourceType) {
        case "post":
          const post = await Post.findById(resourceId);
          isOwner = post?.authorId === userId;
          break;
        // Add more resource types
      }

      if (!isOwner) {
        return res.status(403).json({
          error: "You can only modify your own resources",
        });
      }

      next();
    } catch (error) {
      res.status(500).json({ error: "Ownership check failed" });
    }
  };
}

// Combined permission + ownership check
export function requirePermissionOrOwnership(
  permission: string,
  resourceType: string
) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userId = req.user!.userId;
    const resourceId = req.params.id;

    try {
      // Check if user has permission (e.g., admin can edit all)
      const hasPermission = await PermissionService.userHasPermission(
        userId,
        permission
      );

      if (hasPermission) {
        return next(); // Admin/manager can edit any post
      }

      // Otherwise, check ownership
      let isOwner = false;

      switch (resourceType) {
        case "post":
          const post = await Post.findById(resourceId);
          isOwner = post?.authorId === userId;
          break;
        case "comment":
          const comment = await Comment.findById(resourceId);
          isOwner = comment?.authorId === userId;
          break;
      }

      if (!isOwner) {
        return res.status(403).json({
          error: "You can only modify your own resources",
        });
      }

      next();
    } catch (error) {
      console.error("Authorization check failed:", error);
      res.status(500).json({ error: "Authorization check failed" });
    }
  };
}

// Usage in routes
router.put(
  "/posts/:id",
  authenticateJWT,
  requirePermissionOrOwnership("posts.update.any", "post"),
  postController.updatePost
);

router.delete(
  "/posts/:id",
  authenticateJWT,
  requirePermissionOrOwnership("posts.delete.any", "post"),
  postController.deletePost
);
```

---

### 4.18 Hierarchical Role-Based Access

**Scenario:** Manager can do everything their subordinates can do

**Role Hierarchy:**

```
super_admin
  ↓ inherits from
admin
  ↓ inherits from
manager
  ↓ inherits from
editor
  ↓ inherits from
user
```

**Database Schema:**

```sql
-- Add parent_role_id for hierarchy
ALTER TABLE roles ADD COLUMN parent_role_id UUID REFERENCES roles(id);

-- Example hierarchy
UPDATE roles SET parent_role_id = (SELECT id FROM roles WHERE name = 'admin')
WHERE name = 'manager';

UPDATE roles SET parent_role_id = (SELECT id FROM roles WHERE name = 'manager')
WHERE name = 'editor';

UPDATE roles SET parent_role_id = (SELECT id FROM roles WHERE name = 'editor')
WHERE name = 'user';
```

**Implementation:**

```typescript
// src/services/hierarchical-permission.service.ts
export class HierarchicalPermissionService {
  /**
   * Get all permissions including inherited from parent roles
   */
  static async getUserPermissionsWithInheritance(userId: string): Promise {
    const query = `
      WITH RECURSIVE role_hierarchy AS (
        -- Get user's direct roles
        SELECT r.id, r.name, r.parent_role_id
        FROM roles r
        INNER JOIN user_roles ur ON r.id = ur.role_id
        WHERE ur.user_id = $1
        
        UNION
        
        -- Get parent roles recursively
        SELECT r.id, r.name, r.parent_role_id
        FROM roles r
        INNER JOIN role_hierarchy rh ON r.id = rh.parent_role_id
      )
      SELECT DISTINCT p.name
      FROM permissions p
      INNER JOIN role_permissions rp ON p.id = rp.permission_id
      INNER JOIN role_hierarchy rh ON rp.role_id = rh.id
    `;

    const result = await pool.query(query, [userId]);
    return result.rows.map((row) => row.name);
  }

  /**
   * Check if role A is higher than role B in hierarchy
   */
  static async isRoleHigherThan(roleA: string, roleB: string): Promise {
    const query = `
      WITH RECURSIVE role_hierarchy AS (
        -- Start with roleA
        SELECT id, name, parent_role_id, 0 as level
        FROM roles
        WHERE name = $1
        
        UNION
        
        -- Get parents
        SELECT r.id, r.name, r.parent_role_id, rh.level + 1
        FROM roles r
        INNER JOIN role_hierarchy rh ON r.id = rh.parent_role_id
      )
      SELECT EXISTS(
        SELECT 1 FROM role_hierarchy WHERE name = $2
      ) as is_higher
    `;

    const result = await pool.query(query, [roleA, roleB]);
    return result.rows[0].is_higher;
  }
}
```

---

### 4.19 Tenant-Based Access Control (Multi-Tenancy)

**Scenario:** Users can only access data from their organization/tenant

**Database Schema:**

```sql
-- Add tenant_id to relevant tables
ALTER TABLE users ADD COLUMN tenant_id UUID REFERENCES tenants(id);
ALTER TABLE posts ADD COLUMN tenant_id UUID REFERENCES tenants(id);
ALTER TABLE comments ADD COLUMN tenant_id UUID REFERENCES tenants(id);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_posts_tenant ON posts(tenant_id);
CREATE INDEX idx_comments_tenant ON comments(tenant_id);
```

**Middleware:**

```typescript
// src/middleware/tenant.middleware.ts
import { Request, Response, NextFunction } from "express";

declare global {
  namespace Express {
    interface Request {
      tenantId?: string;
    }
  }
}

export async function extractTenant(
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (!req.user?.userId) {
    return next();
  }

  try {
    // Get user's tenant
    const user = await User.findById(req.user.userId);

    if (!user) {
      return res.status(404).json({ error: "User not found" });
    }

    req.tenantId = user.tenantId;
    next();
  } catch (error) {
    console.error("Tenant extraction failed:", error);
    res.status(500).json({ error: "Tenant extraction failed" });
  }
}

export function enforceTenantIsolation(
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (!req.tenantId) {
    return res.status(403).json({ error: "Tenant required" });
  }
  next();
}

// Usage
app.use("/api", authenticateJWT, extractTenant, enforceTenantIsolation);
```

**Query with Tenant Filter:**

```typescript
// src/services/post.service.ts
export class PostService {
  static async getAll(tenantId: string): Promise {
    // Automatically filter by tenant
    const query = "SELECT * FROM posts WHERE tenant_id = $1";
    const result = await pool.query(query, [tenantId]);
    return result.rows;
  }

  static async create(data: CreatePostDTO, tenantId: string): Promise {
    // Automatically set tenant
    const query = `
      INSERT INTO posts (title, content, author_id, tenant_id)
      VALUES ($1, $2, $3, $4)
      RETURNING *
    `;

    const result = await pool.query(query, [
      data.title,
      data.content,
      data.authorId,
      tenantId,
    ]);

    return result.rows[0];
  }

  static async update(
    id: string,
    data: UpdatePostDTO,
    tenantId: string
  ): Promise {
    // Only update if belongs to tenant
    const query = `
      UPDATE posts
      SET title = $1, content = $2, updated_at = CURRENT_TIMESTAMP
      WHERE id = $3 AND tenant_id = $4
      RETURNING *
    `;

    const result = await pool.query(query, [
      data.title,
      data.content,
      id,
      tenantId,
    ]);

    if (result.rows.length === 0) {
      throw new Error("Post not found or access denied");
    }

    return result.rows[0];
  }
}

// Controller
export class PostController {
  static async getAll(req: Request, res: Response) {
    try {
      const posts = await PostService.getAll(req.tenantId!);
      res.json({ posts });
    } catch (error) {
      res.status(500).json({ error: "Failed to fetch posts" });
    }
  }
}
```

---

## 📖 Section 4: Advanced RBAC Patterns

### 4.20 Dynamic Permissions (Conditions)

**Scenario:** "Can approve expenses under $10,000"

**Database Schema:**

```sql
CREATE TABLE conditional_permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  permission_name VARCHAR(100) NOT NULL,
  conditions JSONB, -- Store conditions as JSON
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Example data
INSERT INTO conditional_permissions (user_id, permission_name, conditions)
VALUES (
  'manager-user-id',
  'expenses.approve',
  '{"amount": {"max": 10000}, "department": "sales"}'::jsonb
);
```

**Implementation:**

```typescript
// src/services/conditional-permission.service.ts
interface Condition {
  field: string;
  operator: "eq" | "ne" | "gt" | "gte" | "lt" | "lte" | "in" | "contains";
  value: any;
}

interface ConditionalPermission {
  permission: string;
  conditions: Record;
}

export class ConditionalPermissionService {
  /**
   * Check if user has permission with conditions
   */
  static async checkPermissionWithConditions(
    userId: string,
    permission: string,
    context: Record
  ): Promise {
    // First check if user has base permission
    const hasBasePermission = await PermissionService.userHasPermission(
      userId,
      permission
    );

    if (!hasBasePermission) {
      return false;
    }

    // Get conditional permissions
    const query = `
      SELECT conditions
      FROM conditional_permissions
      WHERE user_id = $1 AND permission_name = $2
    `;

    const result = await pool.query(query, [userId, permission]);

    // If no conditions, allow
    if (result.rows.length === 0) {
      return true;
    }

    // Check all conditions
    for (const row of result.rows) {
      const conditions = row.conditions;

      if (this.evaluateConditions(conditions, context)) {
        return true;
      }
    }

    return false;
  }

  /**
   * Evaluate conditions against context
   */
  private static evaluateConditions(
    conditions: Record,
    context: Record
  ): boolean {
    for (const [field, condition] of Object.entries(conditions)) {
      const contextValue = context[field];

      // Handle different condition types
      if (typeof condition === "object") {
        if (condition.max !== undefined && contextValue > condition.max) {
          return false;
        }
        if (condition.min !== undefined && contextValue < condition.min) {
          return false;
        }
        if (condition.eq !== undefined && contextValue !== condition.eq) {
          return false;
        }
        if (
          condition.in !== undefined &&
          !condition.in.includes(contextValue)
        ) {
          return false;
        }
      } else {
        // Simple equality check
        if (contextValue !== condition) {
          return false;
        }
      }
    }

    return true;
  }
}

// Usage
export class ExpenseController {
  static async approveExpense(req: Request, res: Response) {
    const { expenseId } = req.params;
    const userId = req.user!.userId;

    // Get expense details
    const expense = await Expense.findById(expenseId);

    // Check permission with conditions
    const canApprove =
      await ConditionalPermissionService.checkPermissionWithConditions(
        userId,
        "expenses.approve",
        {
          amount: expense.amount,
          department: expense.department,
          category: expense.category,
        }
      );

    if (!canApprove) {
      return res.status(403).json({
        error: "Cannot approve this expense",
        reason: "Amount exceeds your approval limit or wrong department",
      });
    }

    // Approve expense
    await expense.approve(userId);
    res.json({ message: "Expense approved" });
  }
}
```

---

### 4.21 Time-Based Access Control

**Scenario:** "Can access reports only during business hours"

**Database Schema:**

```sql
CREATE TABLE time_based_permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  permission_name VARCHAR(100) NOT NULL,
  time_restrictions JSONB, -- {"days": [1,2,3,4,5], "hours": {"start": 9, "end": 17}}
  valid_from TIMESTAMP,
  valid_until TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Implementation:**

```typescript
// src/services/time-based-permission.service.ts
interface TimeRestriction {
  days?: number[]; // 0 = Sunday, 6 = Saturday
  hours?: {
    start: number; // 0-23
    end: number; // 0-23
  };
  timezone?: string;
}

export class TimeBasedPermissionService {
  static async checkTimeBasedPermission(
    userId: string,
    permission: string
  ): Promise {
    const query = `
      SELECT time_restrictions, valid_from, valid_until
      FROM time_based_permissions
      WHERE user_id = $1 AND permission_name = $2
    `;

    const result = await pool.query(query, [userId, permission]);

    if (result.rows.length === 0) {
      // No time restrictions
      return true;
    }

    const now = new Date();

    for (const row of result.rows) {
      // Check validity period
      if (row.valid_from && now < new Date(row.valid_from)) {
        continue;
      }
      if (row.valid_until && now > new Date(row.valid_until)) {
        continue;
      }

      // Check time restrictions
      const restrictions: TimeRestriction = row.time_restrictions;

      if (restrictions.days && !restrictions.days.includes(now.getDay())) {
        continue;
      }

      if (restrictions.hours) {
        const hour = now.getHours();
        if (hour < restrictions.hours.start || hour >= restrictions.hours.end) {
          continue;
        }
      }

      return true;
    }

    return false;
  }
}

// Middleware
export function checkTimeRestrictions(permission: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userId = req.user!.userId;

    const allowed = await TimeBasedPermissionService.checkTimeBasedPermission(
      userId,
      permission
    );

    if (!allowed) {
      return res.status(403).json({
        error: "Access denied",
        reason:
          "This resource is only available during business hours (9 AM - 5 PM, Mon-Fri)",
      });
    }

    next();
  };
}

// Usage
router.get(
  "/reports",
  authenticateJWT,
  requirePermission("reports.read"),
  checkTimeRestrictions("reports.read"),
  reportController.getReports
);
```

---

### 4.22 Temporary Permissions (Delegation)

**Scenario:** "Grant user temporary admin access for 24 hours"

**Database Schema:**

```sql
CREATE TABLE temporary_permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  permission_name VARCHAR(100) NOT NULL,
  granted_by UUID REFERENCES users(id),
  expires_at TIMESTAMP NOT NULL,
  reason TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT check_expiry CHECK (expires_at > created_at)
);

CREATE INDEX idx_temp_perms_expiry ON temporary_permissions(expires_at);

-- Cleanup job (run periodically)
DELETE FROM temporary_permissions WHERE expires_at < CURRENT_TIMESTAMP;
```

**Implementation:**

```typescript
// src/services/temporary-permission.service.ts
export class TemporaryPermissionService {
  /**
   * Grant temporary permission
   */
  static async grantTemporaryPermission(
    userId: string,
    permission: string,
    durationHours: number,
    grantedBy: string,
    reason?: string
  ): Promise {
    const query = `
      INSERT INTO temporary_permissions 
        (user_id, permission_name, granted_by, expires_at, reason)
      VALUES ($1, $2, $3, CURRENT_TIMESTAMP + INTERVAL '${durationHours} hours', $4)
    `;

    await pool.query(query, [userId, permission, grantedBy, reason]);

    // Log the grant
    await AuditLog.create({
      action: "temporary_permission_granted",
      userId: grantedBy,
      targetUserId: userId,
      details: {
        permission,
        durationHours,
        reason,
      },
    });
  }

  /**
   * Check if user has temporary permission
   */
  static async hasTemporaryPermission(
    userId: string,
    permission: string
  ): Promise {
    const query = `
      SELECT EXISTS(
        SELECT 1
        FROM temporary_permissions
        WHERE user_id = $1 
          AND permission_name = $2
          AND expires_at > CURRENT_TIMESTAMP
      ) as has_permission
    `;

    const result = await pool.query(query, [userId, permission]);
    return result.rows[0].has_permission;
  }

  /**
   * Revoke temporary permission
   */
  static async revokeTemporaryPermission(
    userId: string,
    permission: string,
    revokedBy: string
  ): Promise {
    const query = `
      DELETE FROM temporary_permissions
      WHERE user_id = $1 AND permission_name = $2
    `;

    await pool.query(query, [userId, permission]);

    await AuditLog.create({
      action: "temporary_permission_revoked",
      userId: revokedBy,
      targetUserId: userId,
      details: { permission },
    });
  }

  /**
   * Get active temporary permissions for user
   */
  static async getActiveTemporaryPermissions(userId: string) {
    const query = `
      SELECT permission_name, expires_at, granted_by, reason, created_at
      FROM temporary_permissions
      WHERE user_id = $1 AND expires_at > CURRENT_TIMESTAMP
      ORDER BY expires_at ASC
    `;

    const result = await pool.query(query, [userId]);
    return result.rows;
  }
}

// Enhanced permission check with temporary permissions
export class EnhancedPermissionService extends PermissionService {
  static async userHasPermission(userId: string, permission: string): Promise {
    // Check permanent permissions
    const hasPermanent = await super.userHasPermission(userId, permission);
    if (hasPermanent) return true;

    // Check temporary permissions
    const hasTemporary =
      await TemporaryPermissionService.hasTemporaryPermission(
        userId,
        permission
      );

    return hasTemporary;
  }
}
```

**API Endpoints:**

```typescript
// src/controllers/temporary-permission.controller.ts
export class TemporaryPermissionController {
  static async grant(req: Request, res: Response) {
    const { userId, permission, durationHours, reason } = req.body;
    const grantedBy = req.user!.userId;

    // Verify granter has permission to grant
    const canGrant = await PermissionService.userHasPermission(
      grantedBy,
      "permissions.delegate"
    );

    if (!canGrant) {
      return res.status(403).json({ error: "Cannot delegate permissions" });
    }

    await TemporaryPermissionService.grantTemporaryPermission(
      userId,
      permission,
      durationHours,
      grantedBy,
      reason
    );

    res.json({
      message: "Temporary permission granted",
      expiresIn: `${durationHours} hours`,
    });
  }

  static async revoke(req: Request, res: Response) {
    const { userId, permission } = req.params;
    const revokedBy = req.user!.userId;

    await TemporaryPermissionService.revokeTemporaryPermission(
      userId,
      permission,
      revokedBy
    );

    res.json({ message: "Temporary permission revoked" });
  }

  static async list(req: Request, res: Response) {
    const { userId } = req.params;

    const permissions =
      await TemporaryPermissionService.getActiveTemporaryPermissions(userId);

    res.json({ temporaryPermissions: permissions });
  }
}
```

---

## 🎯 Best Practices Summary

### Permission Naming Conventions

```typescript
// ✅ GOOD: Clear, hierarchical naming
"users.create";
"users.read";
"users.update";
"users.delete";
"users.manage"; // Includes all user operations

"posts.create.own"; // Create own posts
"posts.update.own"; // Update own posts
"posts.update.any"; // Update any post (admin)
"posts.delete.any"; // Delete any post

"reports.view.sales"; // Department-specific
"reports.view.finance";
"reports.export";

// ❌ BAD: Unclear, inconsistent
"can_add_user";
"user_permission";
"admin_access";
"full_control";
```

### Role Design

```typescript
// ✅ GOOD: Small, focused roles
roles = {
  viewer: ["posts.read", "comments.read"],
  contributor: ["posts.read", "posts.create.own", "posts.update.own"],
  editor: ["posts.read", "posts.create", "posts.update", "posts.publish"],
  moderator: ["comments.read", "comments.moderate", "comments.delete"],
  admin: ["*"], // All permissions
};

// ❌ BAD: Monolithic roles
roles = {
  basic_user: ["...100 permissions..."],
  power_user: ["...200 permissions..."],
};
```

### Performance Optimization

```typescript
// ✅ Cache permissions in Redis
import Redis from "ioredis";
const redis = new Redis();

export class CachedPermissionService {
  private static CACHE_TTL = 300; // 5 minutes

  static async getUserPermissions(userId: string): Promise {
    // Try cache first
    const cacheKey = `permissions:${userId}`;
    const cached = await redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Fetch from database
    const permissions = await PermissionService.getUserPermissions(userId);

    // Cache for future requests
    await redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(permissions));

    return permissions;
  }

  static async invalidateCache(userId: string): Promise {
    await redis.del(`permissions:${userId}`);
  }
}

// Invalidate cache when permissions change
await PermissionService.assignRole(userId, roleName, grantedBy);
await CachedPermissionService.invalidateCache(userId);
```

---

## 📚 Additional Resources

**Libraries:**

- **Node.js:** `accesscontrol`, `casl`, `casbin`
- **Java:** Apache Shiro, Spring Security ACL
- **Go:** `casbin/casbin`

**Documentation:**

- OWASP Access Control Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html
- NIST RBAC Standard: https://csrc.nist.gov/projects/role-based-access-control

**Tools:**

- **Permit.io** - Authorization as a service
- **Ory Keto** - Open-source permission system
- **Auth0 FGA** - Fine-Grained Authorization

---

## ✅ Day 4 Complete!

You now understand:

- ✅ Simple vs advanced RBAC
- ✅ Role and permission design
- ✅ Implementation across Java, Node.js, and Go
- ✅ Resource-level permissions
- ✅ Hierarchical roles
- ✅ Tenant-based isolation
- ✅ Dynamic and conditional permissions
- ✅ Time-based and temporary permissions

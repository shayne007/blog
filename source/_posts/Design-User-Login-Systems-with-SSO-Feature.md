---
title: Design User Login Systems with SSO Feature
date: 2025-06-12 23:47:50
tags: [system-design, user login system]
categories: [system-design]
---

## System Architecture Overview

### Core Components

A robust login system with SSO consists of several interconnected components:

```mermaid
graph TB
    A[Client Application] --> B[Load Balancer]
    B --> C[API Gateway]
    C --> D[Authentication Service]
    C --> E[Authorization Service]
    C --> F[User Management Service]
    
    D --> G[Session Store/Redis]
    D --> H[Identity Providers]
    H --> I[Google OAuth]
    H --> J[Microsoft AD]
    H --> K[SAML Provider]
    
    F --> L[User Database]
    E --> M[Permissions Database]
    
    D --> N[JWT Token Service]
    N --> O[Token Blacklist]
```

**Interview Insight:** *Candidates should understand the separation of concerns between authentication (who you are) and authorization (what you can do). A common mistake is coupling these tightly together.*

### Key Architectural Decisions

1. **Stateless vs Stateful Authentication**
   - **Stateless (JWT)**: Better for distributed systems, no server-side session storage
   - **Stateful (Sessions)**: Better control over session lifecycle, easier to revoke

2. **Microservices vs Monolithic**
   - **Microservices**: Better scalability, technology diversity
   - **Monolithic**: Simpler deployment, easier testing

**Interview Question:** *"How would you handle user sessions in a distributed system with multiple load-balanced servers?"*

---

## Authentication vs Authorization

### Authentication Flow

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client App
    participant A as Auth Service
    participant I as Identity Provider
    participant D as Database
    
    U->>C: Login Request
    C->>A: Authentication Request
    A->>I: Redirect to IdP
    I->>U: Login Form
    U->>I: Credentials
    I->>A: Auth Code/Token
    A->>D: Validate/Store User
    A->>C: JWT/Session Token
    C->>U: Login Success
```

### Authorization Models

1. **Role-Based Access Control (RBAC)**
```
User → Role → Permissions
john@company.com → Admin → [create_user, delete_user, read_reports]
```

2. **Attribute-Based Access Control (ABAC)**
```
Policy: Allow if (user.department == "HR" AND resource.type == "employee_record" AND action == "read")
```

**Interview Insight:** *Discuss trade-offs between RBAC (simpler) and ABAC (more flexible but complex). Most enterprise systems use hybrid approaches.*

---

## URL Design Patterns

### RESTful Authentication Endpoints

```
Authentication Endpoints:
├── POST   /api/v1/auth/login
├── POST   /api/v1/auth/logout
├── POST   /api/v1/auth/refresh
├── POST   /api/v1/auth/forgot-password
├── POST   /api/v1/auth/reset-password
└── GET    /api/v1/auth/me

SSO Endpoints:
├── GET    /api/v1/auth/sso/{provider}/authorize
├── GET    /api/v1/auth/sso/{provider}/callback
├── POST   /api/v1/auth/sso/{provider}/link
└── DELETE /api/v1/auth/sso/{provider}/unlink

User Management:
├── GET    /api/v1/users
├── POST   /api/v1/users
├── GET    /api/v1/users/{id}
├── PUT    /api/v1/users/{id}
└── DELETE /api/v1/users/{id}
```

### Frontend Route Structure

```
Public Routes:
├── /login
├── /register
├── /forgot-password
├── /reset-password/{token}
└── /sso/callback/{provider}

Protected Routes:
├── /dashboard
├── /profile
├── /settings
└── /admin/*

SSO Integration Routes:
├── /auth/google
├── /auth/microsoft
├── /auth/okta
└── /auth/saml/{domain}
```

### URL Security Best Practices

1. **Use HTTPS Everywhere**
   ```
   ✅ https://api.example.com/auth/login
   ❌ http://api.example.com/auth/login
   ```

2. **Avoid Sensitive Data in URLs**
   ```
   ✅ POST /auth/reset-password (token in body)
   ❌ GET /auth/reset-password?token=sensitive_token
   ```

3. **Implement Rate Limiting**
   ```
   /auth/login - 5 requests per minute per IP
   /auth/forgot-password - 3 requests per hour per email
   ```

**Interview Question:** *"How would you design URLs for a multi-tenant SaaS application with SSO?"*

---

## SSO Implementation Strategies

### OAuth 2.0 Flow

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client App
    participant A as Auth Server
    participant R as Resource Server
    participant I as Identity Provider
    
    U->>C: Access Protected Resource
    C->>A: Authorization Request
    A->>I: Redirect to IdP
    I->>U: Login Prompt
    U->>I: Authenticate
    I->>A: Authorization Code
    A->>I: Exchange Code for Token
    I->>A: Access Token
    A->>C: Access Token
    C->>R: API Request with Token
    R->>C: Protected Resource
    C->>U: Display Resource
```

### SAML 2.0 Integration

```xml
<!-- Example SAML Response Structure -->
<saml:Assertion>
  <saml:Subject>
    <saml:NameID Format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent">
      user@company.com
    </saml:NameID>
  </saml:Subject>
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>user@company.com</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="role">
      <saml:AttributeValue>admin</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

### OpenID Connect (OIDC)

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",
  "aud": "your-client-id",
  "exp": 1633024800,
  "iat": 1633021200,
  "email": "user@example.com",
  "email_verified": true,
  "name": "John Doe",
  "picture": "https://example.com/photo.jpg"
}
```

### Provider-Specific Implementations

| Provider | Protocol | Key Features | Configuration URL |
|----------|----------|--------------|-------------------|
| Google | OAuth 2.0/OIDC | Gmail integration, G Suite | `/auth/google` |
| Microsoft | OAuth 2.0/OIDC | Office 365, Azure AD | `/auth/microsoft` |
| Okta | SAML/OIDC | Enterprise SSO | `/auth/okta` |
| Auth0 | Multiple | Universal login | `/auth/auth0` |

**Interview Insight:** *Candidates should understand the differences between OAuth 2.0 (authorization), OpenID Connect (authentication), and SAML (enterprise SSO). Each serves different use cases.*

---

## Security Considerations

### JWT Token Security

```javascript
// Secure JWT Configuration
const jwtConfig = {
  algorithm: 'RS256',           // Asymmetric signing
  expiresIn: '15m',            // Short-lived access tokens
  issuer: 'your-app.com',
  audience: 'your-api.com',
  clockTolerance: 30,          // Account for clock skew
  ignoreExpiration: false,
  ignoreNotBefore: false
};

// Refresh Token Strategy
const refreshTokenConfig = {
  expiresIn: '7d',             // Longer-lived refresh tokens
  httpOnly: true,              // Prevent XSS attacks
  secure: true,                // HTTPS only
  sameSite: 'strict'           // CSRF protection
};
```

### Security Headers

```javascript
// Essential Security Headers
const securityHeaders = {
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Content-Security-Policy': "default-src 'self'; script-src 'self' 'unsafe-inline'",
  'Referrer-Policy': 'strict-origin-when-cross-origin'
};
```

### Password Security

```python
# Password Hashing Best Practices
import bcrypt
import secrets

class PasswordManager:
    @staticmethod
    def hash_password(password: str) -> str:
        # Generate salt and hash password
        salt = bcrypt.gensalt(rounds=12)  # Adjust rounds based on security needs
        return bcrypt.hashpw(password.encode('utf-8'), salt).decode('utf-8')
    
    @staticmethod
    def verify_password(password: str, hashed: str) -> bool:
        return bcrypt.checkpw(password.encode('utf-8'), hashed.encode('utf-8'))
    
    @staticmethod
    def generate_secure_token(length: int = 32) -> str:
        return secrets.token_urlsafe(length)
```

### Common Security Vulnerabilities

1. **Session Fixation**
   ```python
   # ✅ Regenerate session ID after login
   def login_user(user_id):
       session.regenerate_id()
       session['user_id'] = user_id
   ```

2. **CSRF Protection**
   ```javascript
   // ✅ Use CSRF tokens
   const csrfToken = generateCSRFToken();
   res.cookie('csrf-token', csrfToken, { httpOnly: false });
   ```

3. **Rate Limiting**
   ```python
   # ✅ Implement progressive delays
   login_attempts = {
       1: 0,      # No delay
       2: 1,      # 1 second
       3: 5,      # 5 seconds
       4: 15,     # 15 seconds
       5: 60      # 1 minute
   }
   ```

**Interview Question:** *"How would you prevent brute force attacks on login endpoints while maintaining good user experience?"*

---

## Database Schema Design

### Core Tables

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP,
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP
);

-- SSO Providers
CREATE TABLE sso_providers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL,
    type VARCHAR(20) NOT NULL, -- 'oauth', 'saml', 'oidc'
    client_id VARCHAR(255),
    client_secret VARCHAR(255),
    authorization_url VARCHAR(500),
    token_url VARCHAR(500),
    user_info_url VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User SSO Connections
CREATE TABLE user_sso_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    provider_id UUID REFERENCES sso_providers(id),
    external_user_id VARCHAR(255) NOT NULL,
    access_token TEXT,
    refresh_token TEXT,
    token_expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(provider_id, external_user_id)
);

-- Refresh Tokens
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP,
    user_agent TEXT,
    ip_address INET
);

-- Roles and Permissions
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    resource VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE role_permissions (
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_by UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);
```

### Indexing Strategy

```sql
-- Performance indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active_email ON users(email) WHERE is_active = TRUE;
CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_expires ON refresh_tokens(expires_at);
CREATE INDEX idx_user_sso_external_id ON user_sso_connections(provider_id, external_user_id);
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
```

**Interview Insight:** *Discuss database normalization vs denormalization trade-offs. For high-traffic systems, consider denormalizing frequently accessed data like user permissions.*

---

## API Design and Error Handling

### Authentication API Endpoints

```yaml
# OpenAPI 3.0 Specification
paths:
  /api/v1/auth/login:
    post:
      summary: User login
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LoginRequest'
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthResponse'
        '400':
          description: Invalid credentials
        '429':
          description: Too many login attempts
        '423':
          description: Account locked

  /api/v1/auth/sso/{provider}/authorize:
    get:
      summary: Initiate SSO login
      parameters:
        - name: provider
          in: path
          required: true
          schema:
            type: string
            enum: [google, microsoft, okta]
        - name: redirect_uri
          in: query
          schema:
            type: string
      responses:
        '302':
          description: Redirect to SSO provider
        '400':
          description: Invalid provider or redirect URI
```

### Error Response Format

```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "The email or password you entered is incorrect.",
    "details": {
      "field": "password",
      "attempts_remaining": 3
    },
    "request_id": "req_1234567890",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Status Code Strategy

| Status Code | Use Case | Example |
|-------------|----------|---------|
| 200 | Successful authentication | Login success |
| 400 | Client error | Invalid email format |
| 401 | Authentication failed | Wrong password |
| 403 | Insufficient permissions | Access denied |
| 423 | Account locked | Too many failed attempts |
| 429 | Rate limited | Too many requests |
| 500 | Server error | Database connection failed |

**Interview Question:** *"How would you design error responses to be helpful for developers while not exposing security vulnerabilities?"*

---

## User Experience and Frontend Integration

### Login Flow States

```mermaid
stateDiagram-v2
    [*] --> Initial
    Initial --> Loading: Submit credentials
    Loading --> Success: Valid credentials
    Loading --> Error: Invalid credentials
    Loading --> MFA: MFA required
    MFA --> MFALoading: Submit MFA code
    MFALoading --> Success: Valid code
    MFALoading --> MFAError: Invalid code
    Error --> Initial: Try again
    MFAError --> MFA: Try again
    Success --> [*]
```

### Frontend Integration Example

```javascript
// React Hook for Authentication
import { useState, useEffect, useContext } from 'react';

const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const login = async (credentials) => {
    try {
      setLoading(true);
      const response = await fetch('/api/v1/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(credentials),
      });
      
      if (!response.ok) {
        throw new Error(await response.text());
      }
      
      const data = await response.json();
      setUser(data.user);
      localStorage.setItem('auth_token', data.token);
      return data;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const loginWithSSO = (provider) => {
    window.location.href = `/api/v1/auth/sso/${provider}/authorize?redirect_uri=${encodeURIComponent(window.location.origin)}`;
  };

  const logout = async () => {
    try {
      await fetch('/api/v1/auth/logout', { method: 'POST' });
      setUser(null);
      localStorage.removeItem('auth_token');
    } catch (err) {
      console.error('Logout error:', err);
    }
  };

  return { user, loading, error, login, loginWithSSO, logout };
};
```

### Progressive Enhancement

```javascript
// Graceful degradation for SSO
const SSOButton = ({ provider, onFallback }) => {
  const handleSSOLogin = () => {
    // Check if popup is blocked
    const popup = window.open('', '_blank', 'width=500,height=600');
    if (!popup || popup.closed) {
      // Fallback to redirect
      onFallback();
      return;
    }
    
    // Use popup for better UX
    popup.location.href = `/auth/sso/${provider}`;
  };

  return (
    <button onClick={handleSSOLogin}>
      Continue with {provider}
    </button>
  );
};
```

**Interview Insight:** *Discuss accessibility considerations: keyboard navigation, screen readers, color contrast, and error message clarity. Many candidates overlook these aspects.*

---

## Testing and Monitoring

### Test Strategy

```python
# Example test cases for authentication
import pytest
from unittest.mock import Mock, patch

class TestAuthenticationService:
    
    def test_login_success(self):
        """Test successful login with valid credentials"""
        service = AuthenticationService()
        result = service.login("user@example.com", "correct_password")
        assert result.success is True
        assert result.token is not None
    
    def test_login_invalid_credentials(self):
        """Test login failure with invalid credentials"""
        service = AuthenticationService()
        result = service.login("user@example.com", "wrong_password")
        assert result.success is False
        assert result.error_code == "INVALID_CREDENTIALS"
    
    def test_account_lockout(self):
        """Test account lockout after multiple failed attempts"""
        service = AuthenticationService()
        
        # Simulate multiple failed attempts
        for _ in range(5):
            service.login("user@example.com", "wrong_password")
        
        result = service.login("user@example.com", "correct_password")
        assert result.error_code == "ACCOUNT_LOCKED"
    
    @patch('requests.post')
    def test_sso_login_success(self, mock_post):
        """Test successful SSO login"""
        mock_post.return_value.json.return_value = {
            'access_token': 'mock_token',
            'user_info': {'email': 'user@example.com'}
        }
        
        service = SSOService()
        result = service.authenticate_with_provider('google', 'auth_code')
        assert result.success is True
```

### End-to-End Testing

```javascript
// Cypress E2E tests
describe('Authentication Flow', () => {
  it('should login with valid credentials', () => {
    cy.visit('/login');
    cy.get('[data-testid="email-input"]').type('user@example.com');
    cy.get('[data-testid="password-input"]').type('password123');
    cy.get('[data-testid="login-button"]').click();
    cy.url().should('include', '/dashboard');
  });

  it('should handle SSO login', () => {
    cy.visit('/login');
    cy.get('[data-testid="google-sso-button"]').click();
    
    // Mock SSO callback
    cy.window().then((win) => {
      win.postMessage({
        type: 'SSO_SUCCESS',
        user: { email: 'user@example.com' }
      }, '*');
    });
    
    cy.url().should('include', '/dashboard');
  });
});
```

### Monitoring & Alerting

```yaml
# Prometheus metrics
authentication_attempts_total:
  type: counter
  labels: [method, status]
  description: "Total authentication attempts"

authentication_duration_seconds:
  type: histogram
  labels: [method]
  description: "Authentication request duration"

active_sessions_total:
  type: gauge
  description: "Number of active user sessions"

failed_login_attempts_total:
  type: counter
  labels: [reason]
  description: "Failed login attempts by reason"
```

### Key Metrics to Monitor

1. **Authentication Success Rate**
   - Target: >99.5%
   - Alert: <95% over 5 minutes

2. **Response Time**
   - Target: <200ms p95
   - Alert: >500ms p95 over 2 minutes

3. **Account Lockouts**
   - Alert: >10 lockouts per hour

4. **SSO Provider Health**
   - Monitor: Response times and error rates
   - Alert: >5% error rate

**Interview Question:** *"What metrics would you monitor for an authentication system, and what would trigger alerts?"*

---

## Performance and Scalability

### Caching Strategies

```python
# Redis caching for authentication
import redis
import json
from datetime import timedelta

class AuthCache:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, db=0)
    
    def cache_user_session(self, user_id: str, session_data: dict, ttl: int = 3600):
        """Cache user session data"""
        key = f"session:{user_id}"
        self.redis_client.setex(key, ttl, json.dumps(session_data))
    
    def get_user_session(self, user_id: str) -> dict:
        """Retrieve cached user session"""
        key = f"session:{user_id}"
        data = self.redis_client.get(key)
        return json.loads(data) if data else None
    
    def cache_user_permissions(self, user_id: str, permissions: list, ttl: int = 1800):
        """Cache user permissions for faster authorization"""
        key = f"permissions:{user_id}"
        self.redis_client.setex(key, ttl, json.dumps(permissions))
    
    def invalidate_user_cache(self, user_id: str):
        """Invalidate all cached data for a user"""
        patterns = [f"session:{user_id}", f"permissions:{user_id}"]
        for pattern in patterns:
            self.redis_client.delete(pattern)
```

### Database Connection Pooling

```python
# SQLAlchemy connection pooling
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    "postgresql://user:password@localhost/auth_db",
    poolclass=QueuePool,
    pool_size=20,          # Number of connections to maintain
    max_overflow=30,       # Additional connections when needed
    pool_pre_ping=True,    # Validate connections before use
    pool_recycle=3600,     # Recycle connections after 1 hour
    echo=False
)
```

### Load Balancing Considerations

```yaml
# Nginx load balancer configuration
upstream auth_backend {
    least_conn;
    server auth-server-1:8000 weight=3;
    server auth-server-2:8000 weight=3;
    server auth-server-3:8000 weight=2;
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name auth.example.com;
    
    location /api/v1/auth {
        proxy_pass http://auth_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Session affinity for stateful sessions
        ip_hash;
    }
}
```

### Horizontal Scaling Architecture

```mermaid
graph TB
    A[Load Balancer] --> B[Auth Service 1]
    A --> C[Auth Service 2]
    A --> D[Auth Service 3]
    
    B --> E[Redis Cluster]
    C --> E
    D --> E
    
    B --> F[PostgreSQL Primary]
    C --> G[PostgreSQL Replica 1]
    D --> H[PostgreSQL Replica 2]
    
    F --> G
    F --> H
    
    I[Message Queue] --> B
    I --> C
    I --> D
```

### Performance Optimization Checklist

- [ ] **Database Optimization**
  - Proper indexing on frequently queried columns
  - Connection pooling
  - Read replicas for user lookups
  - Query optimization for complex permission checks

- [ ] **Caching Strategy**
  - Redis for session storage
  - Cache user permissions and roles
  - CDN for static assets
  - HTTP caching headers

- [ ] **Network Optimization**
  - Compress responses (gzip)
  - Minimize API round trips
  - Use HTTP/2
  - Implement proper timeout configurations

- [ ] **Code Optimization**
  - Async/await for I/O operations
  - Batch database operations
  - Lazy loading for user data
  - Efficient JWT token verification

**Interview Question:** *"How would you handle 10,000 concurrent login requests while maintaining sub-200ms response times?"*

---

## Advanced Topics

### Multi-Factor Authentication (MFA)

```python
# TOTP implementation example
import pyotp
import qrcode
from io import BytesIO

class MFAService:
    def generate_secret(self, user_email: str) -> str:
        """Generate TOTP secret for user"""
        secret = pyotp.random_base32()
        return secret
    
    def generate_qr_code(self, user_email: str, secret: str) -> bytes:
        """Generate QR code for authenticator app setup"""
        totp_uri = pyotp.totp.TOTP(secret).provisioning_uri(
            name=user_email,
            issuer_name="Your App Name"
        )
        
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(totp_uri)
        qr.make(fit=True)
        
        img = qr.make_image(fill='black', back_color='white')
        buffer = BytesIO()
        img.save(buffer, format='PNG')
        return buffer.getvalue()
    
    def verify_totp(self, secret: str, token: str) -> bool:
        """Verify TOTP token"""
        totp = pyotp.TOTP(secret)
        return totp.verify(token, valid_window=1)
```

### Zero-Trust Architecture

```mermaid
graph TB
    A[User Request] --> B[Identity Verification]
    B --> C[Device Trust Check]
    C --> D[Network Analysis]
    D --> E[Risk Assessment]
    E --> F[Conditional Access]
    F --> G[Continuous Monitoring]
    
    B --> H[MFA Required?]
    C --> I[Device Compliance]
    D --> J[Location Analysis]
    E --> K[Behavioral Analytics]
```


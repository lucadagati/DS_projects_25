# Project 2: Keycloak Integration in RETROSPECT Middleware

## General Information

**Title:** Keycloak Integration in RETROSPECT Middleware: Unified Identity and Access Management for IoT Orchestration

**Required Skills:** Rust, Kubernetes, Keycloak, OIDC/OAuth2, TLS/mTLS, RBAC

---

## Context and Motivation

According to Deliverable D3.1, RETROSPECT currently uses:
- **X.509 certificates** for device authentication (mTLS)
- **Kubernetes service account tokens** for internal services
- **OIDC mentioned** but not implemented for human operators
- **RBAC** for authorization

**Current Limitations:**
- No centralized Identity Provider for human operators
- No SSO between dashboard and API
- Difficult integration with external Identity Providers
- Limited authorization beyond basic RBAC

**Objective:** Integrate Keycloak as centralized Identity Provider for human operators, maintaining existing certificate-based authentication for devices and service account tokens for internal services.

---

## Project Objectives

### Main Objective
Integrate Keycloak into RETROSPECT middleware to provide unified authentication and authorization for human operators, while maintaining existing mechanisms for devices and internal services.

### Specific Objectives
1. Configure Keycloak as Identity Provider for RETROSPECT
2. Implement OIDC authentication for operators in dashboard
3. Integrate Keycloak with Kubernetes RBAC for authorization
4. Implement multi-mode authentication middleware (Keycloak + certificates + service accounts)
5. Extend Gateway Controller for Keycloak authentication
6. Implement automatic token refresh
7. Provide SSO between dashboard and API

---

## Technical Description

### 1. Keycloak Configuration for RETROSPECT

**Activities:**
- Setup dedicated Keycloak realm for RETROSPECT
- Client configuration:
  - Dashboard UI (public client)
  - API Server (confidential client)
  - Kubernetes API Server (OIDC integration)
- User management:
  - Operator accounts
  - Role definitions (admin, operator, viewer)
  - Group management for multi-tenant
- Service accounts for internal components (optional)

**Deliverable:** Keycloak realm configuration + documentation

---

### 2. Multi-Mode Authentication Middleware

**Components to implement:**

#### 2.1 Authentication Trait
```rust
// New crate: wasmbed-auth-keycloak
pub trait Authenticator {
    async fn authenticate(&self, request: &AuthRequest) -> Result<AuthResult, AuthError>;
    async fn validate_token(&self, token: &str) -> Result<TokenClaims, AuthError>;
    async fn refresh_token(&self, refresh_token: &str) -> Result<TokenPair, AuthError>;
}
```

#### 2.2 Multi-Mode Authentication Handler
```rust
pub enum AuthenticationMode {
    KeycloakOIDC { token: String },
    DeviceCertificate { cert: X509Certificate },
    ServiceAccount { token: String },
}

pub struct UnifiedAuthHandler {
    keycloak: KeycloakAuthenticator,
    cert_validator: CertificateValidator,
    service_account_validator: ServiceAccountValidator,
}

impl UnifiedAuthHandler {
    pub async fn authenticate(&self, mode: AuthenticationMode) -> Result<AuthContext>;
    pub fn determine_mode(&self, request: &Request) -> Option<AuthenticationMode>;
}
```

**Deliverable:** Authentication middleware with multi-mode support

---

### 3. Dashboard Integration

**Activities:**
- Integrate Keycloak login in React dashboard
- Implement OIDC Authorization Code flow
- Token management in frontend:
  - Secure token storage
  - Automatic token refresh
  - Token expiration handling
- Logout and session cleanup
- Protected routes with authentication checks

**React Components:**
```typescript
// Keycloak context provider
export const KeycloakProvider: React.FC = ({ children }) => {
  const keycloak = useKeycloak();
  // Token management, refresh logic
};

// Protected route wrapper
export const ProtectedRoute: React.FC<RouteProps> = ({ ... }) => {
  const { authenticated } = useAuth();
  // Route protection logic
};
```

**Deliverable:** Modified dashboard with integrated Keycloak login

---

### 4. API Server Authentication

**Activities:**
- Modify `wasmbed-api-server` to:
  - Accept JWT tokens from Keycloak
  - Validate tokens using Keycloak public key
  - Extract user claims and roles
  - Middleware for authentication checks
- Support multi-mode authentication:
  - Keycloak JWT for operators
  - X.509 certificates for devices (existing)
  - Service account tokens for internal services (existing)

**Middleware implementation:**
```rust
pub async fn auth_middleware(
    request: Request<Body>,
    next: Next<Body>,
) -> Result<Response, AuthError> {
    // Determine authentication mode
    // Validate token/certificate
    // Extract user context
    // Attach to request
}
```

**Deliverable:** API Server with multi-mode authentication middleware

---

### 5. Kubernetes RBAC Integration

**Activities:**
- Configure Kubernetes OIDC authenticator for Keycloak
- Mapping Keycloak roles → Kubernetes RBAC Roles
- Dynamic role assignment based on Keycloak groups
- Policy engine combining:
  - Keycloak permissions
  - Kubernetes RBAC policies
  - Device capabilities
- Audit logging of all authorization decisions

**Components:**
```yaml
# Kubernetes API Server OIDC configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-apiserver-config
data:
  oidc-issuer-url: "https://keycloak.example.com/realms/retrospect"
  oidc-client-id: "retrospect-k8s"
  oidc-username-claim: "preferred_username"
  oidc-groups-claim: "groups"
```

**Deliverable:** Kubernetes RBAC integration with Keycloak

---

### 6. Gateway Controller Extension

**Activities:**
- Extend Gateway Controller to support Keycloak authentication
- Modify S4T client calls to use Keycloak tokens when needed
- Token management for Stack4Things operations
- Integration with IoTronic using Keycloak tokens

**Modifications to Gateway Controller:**
```rust
pub struct GatewayController {
    client: Client,
    keycloak_client: KeycloakClient, // New
    // ... existing fields
}

impl GatewayController {
    async fn reconcile_with_auth(&self, gateway: Gateway) -> Result<()> {
        // Get Keycloak token for S4T operations
        let token = self.keycloak_client.get_token().await?;
        // Use token for IoTronic API calls
    }
}
```

**Deliverable:** Extended Gateway Controller with Keycloak support

---

### 7. Device Controller Integration (Optional)

**Activities:**
- Integrate Keycloak for device identity management
- Linking device certificates with Keycloak device identities
- Enhanced authorization based on Keycloak device roles
- Coordinated revocation between CA and Keycloak

**Deliverable:** Device Controller integration with Keycloak (optional)

---

## Architecture

### System Architecture Diagram

```mermaid
graph TB
    subgraph Frontend["Frontend Layer"]
        Dashboard["React Dashboard"]
        KeycloakLogin["Keycloak Login<br/>- OIDC Auth Code Flow<br/>- Token Management"]
        Dashboard --> KeycloakLogin
    end
    
    subgraph Middleware["RETROSPECT Middleware"]
        subgraph API["API Server"]
            AuthMW["Unified Auth Middleware<br/>- Keycloak JWT<br/>- Device Certificates<br/>- Service Accounts"]
        end
        
        subgraph Controllers["Controllers"]
            GC["Gateway Controller<br/>(Keycloak)"]
            DC["Device Controller<br/>(Certificates)"]
            AC["App Controller<br/>(Service Account)"]
        end
        
        API --> Controllers
        GC --> GC
        DC --> DC
        AC --> AC
    end
    
    subgraph Keycloak["Keycloak Identity Provider"]
        Realm["Realm: retrospect"]
        Clients["Clients<br/>- Dashboard<br/>- API Server<br/>- K8s"]
        Roles["Roles & Permissions"]
        Realm --> Clients
        Realm --> Roles
    end
    
    subgraph K8s["Kubernetes"]
        APIServer["API Server<br/>- OIDC Authenticator<br/>- RBAC Integration"]
        CRDs["CRDs<br/>(Device/Gateway/App)"]
        APIServer --> CRDs
    end
    
    subgraph S4T["Stack4Things"]
        IoTronic["IoTronic"]
        WAMP["WAMP Router"]
        IoTronic --> WAMP
    end
    
    subgraph Edge["Edge Devices"]
        LR["Lightning-Rod"]
        Device["IoT Devices"]
        LR --> Device
    end
    
    Frontend -->|OIDC Login| Keycloak
    Keycloak -->|JWT Token| Frontend
    Frontend -->|API Call + JWT| API
    API -->|Validate Token| Keycloak
    Controllers -->|K8s Operations| K8s
    K8s -->|OIDC Auth| Keycloak
    GC -->|S4T Operations| S4T
    S4T -->|Commands| Edge
```

### Multi-Mode Authentication Flow

```mermaid
sequenceDiagram
    participant User as Human Operator
    participant Dashboard as Dashboard
    participant Keycloak as Keycloak
    participant API as API Server
    participant MW as Auth Middleware
    participant Controller as Controllers
    
    Note over User,Keycloak: Operator Authentication (Keycloak OIDC)
    User->>Dashboard: 1. Access Dashboard
    Dashboard->>Keycloak: 2. Redirect to Login
    Keycloak->>User: 3. Login Form
    User->>Keycloak: 4. Credentials
    Keycloak->>Dashboard: 5. Authorization Code
    Dashboard->>Keycloak: 6. Exchange Code for Token
    Keycloak->>Dashboard: 7. JWT Access Token + Refresh Token
    
    Note over Dashboard,Controller: API Request with Authentication
    Dashboard->>API: 8. API Request + JWT Token
    API->>MW: 9. Validate Request
    MW->>Keycloak: 10. Validate JWT Token
    Keycloak->>MW: 11. Token Valid + Claims
    MW->>API: 12. Auth Context
    API->>Controller: 13. Authorized Request
    Controller->>API: 14. Response
    API->>Dashboard: 15. Response
    
    Note over Dashboard,Keycloak: Token Refresh (before expiration)
    Dashboard->>Keycloak: 16. Refresh Token Request
    Keycloak->>Dashboard: 17. New Access Token
```

### Multi-Mode Authentication Architecture

```mermaid
graph TB
    subgraph Request["Incoming Request"]
        HTTP["HTTP Request"]
    end
    
    subgraph AuthMW["Unified Auth Middleware"]
        Detector["Auth Mode Detector"]
        KeycloakAuth["Keycloak Authenticator<br/>- JWT Validation<br/>- Role Extraction"]
        CertAuth["Certificate Validator<br/>- X.509 Validation<br/>- Device Identity"]
        SAAuth["Service Account Validator<br/>- K8s Token Validation"]
    end
    
    subgraph Keycloak["Keycloak"]
        Realm["Realm"]
        Validation["Token Validation"]
    end
    
    subgraph PKI["PKI"]
        CA["Certificate Authority"]
        CertVal["Certificate Validation"]
    end
    
    subgraph K8s["Kubernetes"]
        SA["Service Accounts"]
        TokenVal["Token Validation"]
    end
    
    HTTP --> Detector
    Detector -->|JWT Token| KeycloakAuth
    Detector -->|X.509 Cert| CertAuth
    Detector -->|SA Token| SAAuth
    
    KeycloakAuth --> Keycloak
    Keycloak --> Validation
    Validation --> KeycloakAuth
    
    CertAuth --> PKI
    PKI --> CA
    CA --> CertVal
    CertVal --> CertAuth
    
    SAAuth --> K8s
    K8s --> SA
    SA --> TokenVal
    TokenVal --> SAAuth
    
    KeycloakAuth -->|Auth Context| API
    CertAuth -->|Auth Context| API
    SAAuth -->|Auth Context| API
```

### Kubernetes RBAC Integration Flow

```mermaid
sequenceDiagram
    participant User as Operator
    participant Dashboard as Dashboard
    participant Keycloak as Keycloak
    participant K8sAPI as K8s API Server
    participant Controller as RETROSPECT Controller
    participant CRD as CRD Resource
    
    User->>Dashboard: 1. Create/Update Resource
    Dashboard->>Keycloak: 2. Get JWT Token
    Keycloak->>Dashboard: 3. JWT with Groups/Roles
    Dashboard->>K8sAPI: 4. kubectl apply + JWT Token
    K8sAPI->>Keycloak: 5. Validate JWT Token
    Keycloak->>K8sAPI: 6. Token Valid + Groups: ["admin", "operators"]
    K8sAPI->>K8sAPI: 7. Map Groups to RBAC Roles
    K8sAPI->>K8sAPI: 8. Check RBAC Permissions
    K8sAPI->>CRD: 9. Create/Update CRD
    CRD->>Controller: 10. Reconcile Event
    Controller->>Controller: 11. Process Resource
    Controller->>CRD: 12. Update Status
```

### Gateway Controller Integration with Stack4Things

```mermaid
graph LR
    subgraph K8s["Kubernetes"]
        GC["Gateway Controller"]
        GatewayCRD["Gateway CRD"]
        GC --> GatewayCRD
    end
    
    subgraph Keycloak["Keycloak"]
        Realm["Realm"]
        Token["Service Token"]
    end
    
    subgraph S4T["Stack4Things"]
        IoTronic["IoTronic API"]
        WAMP["WAMP Router"]
        IoTronic --> WAMP
    end
    
    subgraph Edge["Edge"]
        LR["Lightning-Rod"]
        Plugin["Plugins"]
        LR --> Plugin
    end
    
    GC -->|1. Get Keycloak Token| Keycloak
    Keycloak -->|2. JWT Token| GC
    GC -->|3. API Call with JWT| IoTronic
    IoTronic -->|4. Validate Token| Keycloak
    Keycloak -->|5. Authorized| IoTronic
    IoTronic -->|6. Deploy Plugin| WAMP
    WAMP -->|7. WebSocket Command| LR
    LR -->|8. Execute| Plugin
```

---

## Technology Stack

- **Languages:** Rust (middleware), TypeScript/React (dashboard)
- **Frameworks:** Keycloak Admin API, OIDC/OAuth2 libraries
- **Protocols:** OIDC, OAuth2, JWT, TLS 1.3, mTLS
- **Infrastructure:** Kubernetes, Keycloak
- **Tooling:** kubectl, Helm, GitOps, Rust toolchain

---

## Expected Deliverables

### Code and Implementation
1. Multi-mode authentication middleware (Rust crate)
2. Dashboard with integrated Keycloak login
3. API Server with authentication middleware
4. Gateway Controller extension with Keycloak support
5. Kubernetes RBAC integration

### Documentation
1. Keycloak configuration guide for RETROSPECT
2. Authentication architecture documentation
3. API documentation with authentication endpoints
4. Dashboard user guide
5. Deployment guide with Keycloak setup

### Testing
1. Unit tests for authentication components
2. Integration tests: Dashboard → API → Controllers
3. End-to-end tests with complete scenario
4. Security tests (token validation, expiration, revocation)
5. Performance tests (token refresh, API latency)

### Use Case
1. Multi-operator deployment with SSO
2. Multi-tenant scenario with role-based access
3. Integration with external Identity Providers (Azure AD, Okta)

---

## Bibliography and References

### Stack4Things Repositories and Documentation
- Stack4Things GitHub: https://github.com/MDSLab/Stack4Things
- Stack4Things IoTronic API: https://github.com/MDSLab/iotronic
- Lightning-Rod Documentation: https://github.com/MDSLab/iotronic-lightningrod

### Related Stack4Things Integration Repositories
- **Stack4Things SDK for Go** (`https://github.com/MIKE9708/s4t-sdk-go.git`): 
  - Go SDK for Stack4Things API interactions
  - May be useful as reference for Gateway Controller integration with Stack4Things
- **Crossplane Provider for Stack4Things** (`https://github.com/MIKE9708/Provider4_S4T.git`):
  - Crossplane Provider implementation for Stack4Things
  - Useful reference for understanding Stack4Things API integration patterns

### Keycloak Documentation
- Keycloak Official Documentation: https://www.keycloak.org/documentation
- Keycloak Server Administration Guide: https://www.keycloak.org/docs/latest/server_admin/
- Keycloak Authorization Services: https://www.keycloak.org/docs/latest/authorization_services/
- Keycloak Admin REST API: https://www.keycloak.org/docs-api/latest/rest-api/
- Keycloak JavaScript Adapter: https://www.keycloak.org/docs/latest/securing_apps/#_javascript_adapter
- Keycloak OIDC/OAuth2 Configuration: https://www.keycloak.org/docs/latest/securing_apps/
- Keycloak Service Account Configuration: https://www.keycloak.org/docs/latest/server_admin/#service-accounts

### OIDC/OAuth2 Standards and Protocols
- RFC 6749 - OAuth 2.0 Authorization Framework: https://datatracker.ietf.org/doc/html/rfc6749
- RFC 6750 - OAuth 2.0 Bearer Token Usage: https://datatracker.ietf.org/doc/html/rfc6750
- RFC 7519 - JSON Web Token (JWT): https://datatracker.ietf.org/doc/html/rfc7519
- RFC 7517 - JSON Web Key (JWK): https://datatracker.ietf.org/doc/html/rfc7517
- OpenID Connect Core 1.0: https://openid.net/specs/openid-connect-core-1_0.html
- OpenID Connect Discovery 1.0: https://openid.net/specs/openid-connect-discovery-1_0.html
- OAuth 2.0 Token Introspection (RFC 7662): https://datatracker.ietf.org/doc/html/rfc7662
- OAuth 2.0 Device Flow (RFC 8628): https://datatracker.ietf.org/doc/html/rfc8628

### Kubernetes Documentation
- Kubernetes Custom Resource Definitions: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
- Kubernetes Controller Pattern: https://kubernetes.io/docs/concepts/architecture/controller/
- Kubernetes API Server OIDC Authenticator: https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens
- Kubernetes RBAC Authorization: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Kubernetes Service Accounts: https://kubernetes.io/docs/concepts/security/service-accounts/
- Kubernetes Admission Controllers: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/

### Rust Programming Language
- Rust Official Documentation: https://doc.rust-lang.org/
- Rust Async Programming: https://rust-lang.github.io/async-book/
- Rust HTTP Clients:
  - reqwest: https://docs.rs/reqwest/
  - hyper: https://hyper.rs/
- Rust JWT Libraries:
  - jsonwebtoken: https://docs.rs/jsonwebtoken/
  - jwt-simple: https://docs.rs/jwt-simple/
- Rust OAuth2/OIDC Libraries:
  - oauth2: https://docs.rs/oauth2/
- Rust Kubernetes Client (kube-rs): https://docs.rs/kube/
- Rust TLS Libraries:
  - rustls: https://docs.rs/rustls/
  - tokio-rustls: https://docs.rs/tokio-rustls/

### React/TypeScript Documentation
- React Official Documentation: https://react.dev/
- TypeScript Documentation: https://www.typescriptlang.org/docs/
- React Keycloak Integration:
  - @react-keycloak/web: https://www.npmjs.com/package/@react-keycloak/web
  - keycloak-js: https://www.npmjs.com/package/keycloak-js
- React Router: https://reactrouter.com/

### TLS and Certificate Management
- RFC 8446 - TLS 1.3 Protocol: https://datatracker.ietf.org/doc/html/rfc8446
- RFC 5280 - X.509 Certificate Profile: https://datatracker.ietf.org/doc/html/rfc5280
- PKI Best Practices: https://www.keycloak.org/docs/latest/server_admin/#pki

### Additional Resources
- WebSocket Protocol (RFC 6455): https://datatracker.ietf.org/doc/html/rfc6455
- WAMP Protocol Documentation: https://wamp-proto.org/
- CBOR Encoding (RFC 7049): https://datatracker.ietf.org/doc/html/rfc7049
- WebAssembly Documentation: https://webassembly.org/
- Wasmtime Runtime: https://wasmtime.dev/

---

## Notes

This project focuses on integrating Keycloak throughout the RETROSPECT middleware stack, providing unified identity and access management while maintaining existing security mechanisms for devices and internal services.


# Progetto 2: Integrazione Keycloak nel Middleware RETROSPECT

## Informazioni Generali

**Titolo:** Integrazione Keycloak nel Middleware RETROSPECT: Unified Identity and Access Management per Orchestrazione IoT

**Competenze Richieste:** Rust, Kubernetes, Keycloak, OIDC/OAuth2, TLS/mTLS, RBAC

---

## Contesto e Motivazione

Dal Deliverable D3.1, RETROSPECT attualmente utilizza:
- **Certificati X.509** per autenticazione device (mTLS)
- **Token service account Kubernetes** per servizi interni
- **OIDC menzionato** ma non implementato per operatori umani
- **RBAC** per autorizzazione

**Limitazioni Attuali:**
- Nessun Identity Provider centralizzato per operatori umani
- Nessun SSO tra dashboard e API
- Difficile integrazione con Identity Provider esterni
- Autorizzazione limitata oltre RBAC base

**Obiettivo:** Integrare Keycloak come Identity Provider centralizzato per operatori umani, mantenendo l'autenticazione basata su certificati esistente per i device e i token service account per i servizi interni.

---

## Obiettivi del Progetto

### Obiettivo Principale
Integrare Keycloak nel middleware RETROSPECT per fornire autenticazione e autorizzazione unificata per operatori umani, mantenendo i meccanismi esistenti per device e servizi interni.

### Obiettivi Specifici
1. Configurare Keycloak come Identity Provider per RETROSPECT
2. Implementare autenticazione OIDC per operatori nel dashboard
3. Integrare Keycloak con Kubernetes RBAC per autorizzazione
4. Implementare middleware di autenticazione multi-mode (Keycloak + certificati + service accounts)
5. Estendere Gateway Controller per autenticazione Keycloak
6. Implementare refresh automatico token
7. Fornire SSO tra dashboard e API

---

## Descrizione Tecnica

### 1. Configurazione Keycloak per RETROSPECT

**Attività:**
- Setup realm Keycloak dedicato per RETROSPECT
- Configurazione client:
  - Dashboard UI (public client)
  - API Server (confidential client)
  - Kubernetes API Server (integrazione OIDC)
- Gestione utenti:
  - Account operatori
  - Definizione ruoli (admin, operator, viewer)
  - Gestione gruppi per multi-tenant
- Service accounts per componenti interni (opzionale)

**Deliverable:** Configurazione realm Keycloak + documentazione

---

### 2. Middleware di Autenticazione Multi-Mode

**Componenti da implementare:**

#### 2.1 Authentication Trait
```rust
// Nuovo crate: wasmbed-auth-keycloak
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

**Deliverable:** Middleware di autenticazione con supporto multi-mode

---

### 3. Integrazione Dashboard

**Attività:**
- Integrare login Keycloak nel dashboard React
- Implementare OIDC Authorization Code flow
- Gestione token nel frontend:
  - Storage sicuro token
  - Refresh automatico token
  - Gestione expiration token
- Logout e cleanup sessione
- Route protette con controlli autenticazione

**Componenti React:**
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

**Deliverable:** Dashboard modificato con login Keycloak integrato

---

### 4. Autenticazione API Server

**Attività:**
- Modificare `wasmbed-api-server` per:
  - Accettare token JWT da Keycloak
  - Validare token usando chiave pubblica Keycloak
  - Estrarre user claims e ruoli
  - Middleware per controlli autenticazione
- Supportare autenticazione multi-mode:
  - JWT Keycloak per operatori
  - Certificati X.509 per device (esistente)
  - Token service account per servizi interni (esistente)

**Implementazione middleware:**
```rust
pub async fn auth_middleware(
    request: Request<Body>,
    next: Next<Body>,
) -> Result<Response, AuthError> {
    // Determinare modalità autenticazione
    // Validare token/certificato
    // Estrarre contesto utente
    // Allegare a request
}
```

**Deliverable:** API Server con middleware autenticazione multi-mode

---

### 5. Integrazione Kubernetes RBAC

**Attività:**
- Configurare OIDC authenticator Kubernetes per Keycloak
- Mapping ruoli Keycloak → Kubernetes RBAC Roles
- Assegnazione ruoli dinamica basata su gruppi Keycloak
- Policy engine che combina:
  - Permessi Keycloak
  - Politiche Kubernetes RBAC
  - Capabilities device
- Audit logging di tutte le decisioni di autorizzazione

**Componenti:**
```yaml
# Configurazione Kubernetes API Server OIDC
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

**Deliverable:** Integrazione Kubernetes RBAC con Keycloak

---

### 6. Estensione Gateway Controller

**Attività:**
- Estendere Gateway Controller per supportare autenticazione Keycloak
- Modificare chiamate client S4T per usare token Keycloak quando necessario
- Gestione token per operazioni Stack4Things
- Integrazione con IoTronic usando token Keycloak

**Modifiche a Gateway Controller:**
```rust
pub struct GatewayController {
    client: Client,
    keycloak_client: KeycloakClient, // Nuovo
    // ... campi esistenti
}

impl GatewayController {
    async fn reconcile_with_auth(&self, gateway: Gateway) -> Result<()> {
        // Ottenere token Keycloak per operazioni S4T
        let token = self.keycloak_client.get_token().await?;
        // Usare token per chiamate API IoTronic
    }
}
```

**Deliverable:** Gateway Controller esteso con supporto Keycloak

---

### 7. Integrazione Device Controller (Opzionale)

**Attività:**
- Integrare Keycloak per gestione identità device
- Linking certificati device con identità Keycloak device
- Autorizzazione avanzata basata su ruoli Keycloak device
- Revoca coordinata tra CA e Keycloak

**Deliverable:** Integrazione Device Controller con Keycloak (opzionale)

---

## Architettura

### Diagramma Architettura Sistema

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

### Diagramma Flusso Autenticazione Multi-Mode

```mermaid
sequenceDiagram
    participant User as Operatore Umano
    participant Dashboard as Dashboard
    participant Keycloak as Keycloak
    participant API as API Server
    participant MW as Auth Middleware
    participant Controller as Controllers
    
    Note over User,Keycloak: Autenticazione Operatore (Keycloak OIDC)
    User->>Dashboard: 1. Accesso Dashboard
    Dashboard->>Keycloak: 2. Redirect a Login
    Keycloak->>User: 3. Form Login
    User->>Keycloak: 4. Credenziali
    Keycloak->>Dashboard: 5. Authorization Code
    Dashboard->>Keycloak: 6. Exchange Code for Token
    Keycloak->>Dashboard: 7. JWT Access Token + Refresh Token
    
    Note over Dashboard,Controller: Richiesta API con Autenticazione
    Dashboard->>API: 8. API Request + JWT Token
    API->>MW: 9. Valida Request
    MW->>Keycloak: 10. Valida JWT Token
    Keycloak->>MW: 11. Token Valido + Claims
    MW->>API: 12. Auth Context
    API->>Controller: 13. Richiesta Autorizzata
    Controller->>API: 14. Response
    API->>Dashboard: 15. Response
    
    Note over Dashboard,Keycloak: Token Refresh (prima expiration)
    Dashboard->>Keycloak: 16. Refresh Token Request
    Keycloak->>Dashboard: 17. Nuovo Access Token
```

### Architettura Autenticazione Multi-Mode

```mermaid
graph TB
    subgraph Request["Request Incoming"]
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

### Flusso Integrazione Kubernetes RBAC

```mermaid
sequenceDiagram
    participant User as Operatore
    participant Dashboard as Dashboard
    participant Keycloak as Keycloak
    participant K8sAPI as K8s API Server
    participant Controller as RETROSPECT Controller
    participant CRD as CRD Resource
    
    User->>Dashboard: 1. Create/Update Resource
    Dashboard->>Keycloak: 2. Get JWT Token
    Keycloak->>Dashboard: 3. JWT con Groups/Roles
    Dashboard->>K8sAPI: 4. kubectl apply + JWT Token
    K8sAPI->>Keycloak: 5. Valida JWT Token
    Keycloak->>K8sAPI: 6. Token Valido + Groups: ["admin", "operators"]
    K8sAPI->>K8sAPI: 7. Mappa Groups su RBAC Roles
    K8sAPI->>K8sAPI: 8. Controlla Permessi RBAC
    K8sAPI->>CRD: 9. Create/Update CRD
    CRD->>Controller: 10. Reconcile Event
    Controller->>Controller: 11. Process Resource
    Controller->>CRD: 12. Update Status
```

### Integrazione Gateway Controller con Stack4Things

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

## Stack Tecnologico

- **Linguaggi:** Rust (middleware), TypeScript/React (dashboard)
- **Framework:** Keycloak Admin API, librerie OIDC/OAuth2
- **Protocolli:** OIDC, OAuth2, JWT, TLS 1.3, mTLS
- **Infrastruttura:** Kubernetes, Keycloak
- **Tooling:** kubectl, Helm, GitOps, Rust toolchain

---

## Deliverable Attesi

### Codice e Implementazione
1. Middleware autenticazione multi-mode (Rust crate)
2. Dashboard con login Keycloak integrato
3. API Server con middleware autenticazione
4. Estensione Gateway Controller con supporto Keycloak
5. Integrazione Kubernetes RBAC

### Documentazione
1. Guida configurazione Keycloak per RETROSPECT
2. Documentazione architettura autenticazione
3. Documentazione API con endpoint autenticazione
4. Guida utente dashboard
5. Guida deployment con setup Keycloak

### Testing
1. Unit test per componenti autenticazione
2. Test di integrazione: Dashboard → API → Controllers
3. Test end-to-end con scenario completo
4. Test di sicurezza (validazione token, expiration, revoca)
5. Test di performance (token refresh, latenza API)

### Use Case
1. Deployment multi-operatore con SSO
2. Scenario multi-tenant con accesso role-based
3. Integrazione con Identity Provider esterni (Azure AD, Okta)

---

## Bibliografia e Riferimenti

### Repository e Documentazione Stack4Things
- Stack4Things GitHub: https://github.com/MDSLab/Stack4Things
- Stack4Things IoTronic API: https://github.com/MDSLab/iotronic
- Documentazione Lightning-Rod: https://github.com/MDSLab/iotronic-lightningrod

### Repository di Integrazione Stack4Things Correlati
- **Stack4Things SDK for Go** (`https://github.com/MIKE9708/s4t-sdk-go.git`): 
  - SDK Go per interazioni API Stack4Things
  - Può essere utile come riferimento per integrazione Gateway Controller con Stack4Things
- **Crossplane Provider per Stack4Things** (`https://github.com/MIKE9708/Provider4_S4T.git`):
  - Implementazione Crossplane Provider per Stack4Things
  - Utile riferimento per comprendere pattern di integrazione API Stack4Things

### Documentazione Keycloak
- Documentazione Ufficiale Keycloak: https://www.keycloak.org/documentation
- Guida Amministrazione Keycloak Server: https://www.keycloak.org/docs/latest/server_admin/
- Keycloak Authorization Services: https://www.keycloak.org/docs/latest/authorization_services/
- Keycloak Admin REST API: https://www.keycloak.org/docs-api/latest/rest-api/
- Keycloak JavaScript Adapter: https://www.keycloak.org/docs/latest/securing_apps/#_javascript_adapter
- Configurazione Keycloak OIDC/OAuth2: https://www.keycloak.org/docs/latest/securing_apps/
- Configurazione Keycloak Service Account: https://www.keycloak.org/docs/latest/server_admin/#service-accounts

### Standard e Protocolli OIDC/OAuth2
- RFC 6749 - OAuth 2.0 Authorization Framework: https://datatracker.ietf.org/doc/html/rfc6749
- RFC 6750 - OAuth 2.0 Bearer Token Usage: https://datatracker.ietf.org/doc/html/rfc6750
- RFC 7519 - JSON Web Token (JWT): https://datatracker.ietf.org/doc/html/rfc7519
- RFC 7517 - JSON Web Key (JWK): https://datatracker.ietf.org/doc/html/rfc7517
- OpenID Connect Core 1.0: https://openid.net/specs/openid-connect-core-1_0.html
- OpenID Connect Discovery 1.0: https://openid.net/specs/openid-connect-discovery-1_0.html
- OAuth 2.0 Token Introspection (RFC 7662): https://datatracker.ietf.org/doc/html/rfc7662
- OAuth 2.0 Device Flow (RFC 8628): https://datatracker.ietf.org/doc/html/rfc8628

### Documentazione Kubernetes
- Kubernetes Custom Resource Definitions: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
- Kubernetes Controller Pattern: https://kubernetes.io/docs/concepts/architecture/controller/
- Kubernetes API Server OIDC Authenticator: https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens
- Kubernetes RBAC Authorization: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Kubernetes Service Accounts: https://kubernetes.io/docs/concepts/security/service-accounts/
- Kubernetes Admission Controllers: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/

### Linguaggio di Programmazione Rust
- Documentazione Ufficiale Rust: https://doc.rust-lang.org/
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

### Documentazione React/TypeScript
- Documentazione Ufficiale React: https://react.dev/
- Documentazione TypeScript: https://www.typescriptlang.org/docs/
- Integrazione React Keycloak:
  - @react-keycloak/web: https://www.npmjs.com/package/@react-keycloak/web
  - keycloak-js: https://www.npmjs.com/package/keycloak-js
- React Router: https://reactrouter.com/

### TLS e Gestione Certificati
- RFC 8446 - Protocollo TLS 1.3: https://datatracker.ietf.org/doc/html/rfc8446
- RFC 5280 - Profilo Certificato X.509: https://datatracker.ietf.org/doc/html/rfc5280
- PKI Best Practices: https://www.keycloak.org/docs/latest/server_admin/#pki

### Risorse Aggiuntive
- Protocollo WebSocket (RFC 6455): https://datatracker.ietf.org/doc/html/rfc6455
- Documentazione Protocollo WAMP: https://wamp-proto.org/
- CBOR Encoding (RFC 7049): https://datatracker.ietf.org/doc/html/rfc7049
- Documentazione WebAssembly: https://webassembly.org/
- Wasmtime Runtime: https://wasmtime.dev/

---

## Note

Questo progetto si concentra sull'integrazione di Keycloak in tutto lo stack middleware RETROSPECT, fornendo unified identity and access management mantenendo i meccanismi di sicurezza esistenti per device e servizi interni.


# Progetto 1: Crossplane Provider per Stack4Things con Autenticazione Keycloak

## Informazioni Generali

**Titolo:** Crossplane Provider per Stack4Things: Migrazione da Keystone a Keycloak per Autenticazione Unificata

**Livello:** Avanzato (Tesi Magistrale)

**Competenze Richieste:** Go, Kubernetes, Crossplane SDK, Keycloak, OIDC/OAuth2, Stack4Things API

---

## Contesto e Motivazione

Dal Deliverable D1.3, Stack4Things si integra con Crossplane per l'orchestrazione multi-cloud. Attualmente, il Crossplane Provider utilizza Keystone per l'autenticazione con IoTronic. Tuttavia, questo approccio presenta diverse limitazioni:

- **Dipendenza da Keystone:** Fortemente accoppiato all'ecosistema OpenStack
- **Gestione token limitata:** Token senza meccanismi di expiration adeguati
- **Limitazioni autorizzazione:** Difficile implementare politiche di autorizzazione fine-grained
- **Sfide federazione:** Difficile integrazione con Identity Provider esterni

**Obiettivo:** Sostituire Keystone con Keycloak nel Crossplane Provider, mantenendo piena compatibilità con Stack4Things IoTronic migliorando sicurezza e flessibilità.

---

## Obiettivi del Progetto

### Obiettivo Principale
Modificare il Crossplane Provider per Stack4Things per utilizzare Keycloak come Identity Provider al posto di Keystone, implementando autenticazione OIDC/OAuth2 per chiamate API IoTronic.

### Obiettivi Specifici
1. Analizzare il Crossplane Provider esistente e identificare i punti di integrazione con Keystone
2. Configurare Keycloak come Identity Provider per Stack4Things
3. Implementare client OIDC/OAuth2 nel Crossplane Provider
4. Sviluppare adapter per IoTronic che accetti token Keycloak
5. Mantenere backward compatibility durante la migrazione
6. Implementare meccanismo di refresh automatico dei token
7. Testare l'integrazione end-to-end

---

## Descrizione Tecnica

### 1. Analisi del Crossplane Provider Esistente

**Attività:**
- Analizzare il codice del Crossplane Provider per Stack4Things
- Identificare dove viene utilizzato Keystone:
  - Autenticazione iniziale
  - Generazione token
  - Validazione token
  - Controlli di autorizzazione
- Mappare le chiamate API IoTronic che richiedono autenticazione
- Documentare il flusso di autenticazione attuale

**Deliverable:** Documento di analisi con mappatura Keystone → Keycloak

---

### 2. Configurazione Keycloak per Stack4Things

**Attività:**
- Setup realm Keycloak dedicato per Stack4Things
- Configurazione client per:
  - Crossplane Provider (confidential client)
  - IoTronic API server (public client)
  - Lightning-Rod agents (service accounts)
- Definizione ruoli e permessi:
  - Ruoli di gestione board
  - Ruoli di deployment plugin
  - Ruoli di gestione servizi
  - Ruoli read-only
- Configurazione flow OIDC/OAuth2:
  - Client Credentials flow per service-to-service
  - Authorization Code flow per interazioni utente
  - Device Flow per dispositivi headless (opzionale)

**Deliverable:** Export configurazione realm Keycloak + documentazione

---

### 3. Modifiche al Crossplane Provider

**Componenti da modificare:**

#### 3.1 Authentication Package
```go
// Nuovo package: pkg/auth/keycloak
package keycloak

type KeycloakAuthenticator struct {
    clientID     string
    clientSecret string
    realmURL     string
    httpClient   *http.Client
}

func (k *KeycloakAuthenticator) Authenticate(ctx context.Context) (*Token, error)
func (k *KeycloakAuthenticator) RefreshToken(ctx context.Context, refreshToken string) (*Token, error)
func (k *KeycloakAuthenticator) ValidateToken(ctx context.Context, token string) error
```

#### 3.2 Provider Configuration
- Aggiungere configurazione Keycloak nel ProviderConfig:
  - URL server Keycloak
  - Nome realm
  - Client ID e Secret
  - Token endpoint
  - Refresh token endpoint

#### 3.3 Modifica IoTronic Client
- Modificare il client IoTronic per:
  - Usare token JWT da Keycloak invece di token Keystone
  - Includere token nell'header Authorization
  - Gestire expiration token e refresh automatico
  - Gestire errori di autenticazione

**Deliverable:** Codice modificato del Crossplane Provider con autenticazione Keycloak

---

### 4. Adapter IoTronic per Keycloak

**Attività:**
- Creare middleware per IoTronic API server che:
  - Accetti token JWT da Keycloak
  - Validi token usando la chiave pubblica Keycloak
  - Estragga claims (user, roles, permissions)
  - Mappi ruoli Keycloak su permessi Stack4Things
- Modificare delegation agent per usare permessi Keycloak
- Implementare backward compatibility per token Keystone (durante migrazione)

**Deliverable:** Adapter IoTronic con supporto Keycloak + compatibilità Keystone

---

### 5. Gestione Token e Refresh

**Attività:**
- Implementare caching token nel Crossplane Provider
- Refresh automatico token prima dell'expiration
- Rotazione token per operazioni long-running
- Storage sicuro token (Kubernetes Secrets)
- Supporto revoca token

**Deliverable:** Sistema di gestione token con refresh automatico

---

### 6. Strategia di Migrazione

**Attività:**
- Implementare autenticazione dual-mode:
  - Accettare sia token Keystone che Keycloak
  - Migrazione graduale degli utenti
- Token translation layer (opzionale):
  - Convertire token Keystone in token Keycloak
  - Permettere operazioni miste durante la transizione
- Tooling per migrazione:
  - Script per migrare utenti da Keystone a Keycloak
  - Validazione migrazione
  - Meccanismo di rollback

**Deliverable:** Strategia di migrazione documentata + tooling per migrazione

---

## Architettura

### Diagramma Architettura Sistema

```mermaid
graph TB
    subgraph K8s["Kubernetes Cluster"]
        subgraph CP["Crossplane Provider"]
            KA["Keycloak Authenticator<br/>- OIDC/OAuth2 Client<br/>- Token Management<br/>- Token Refresh"]
            IC["IoTronic Client<br/>- JWT Token Auth<br/>- Token Refresh Handler"]
            KA --> IC
        end
        CRD["S4T CRDs<br/>(Board/Plugin/Service)"]
        CP --> CRD
    end
    
    subgraph Keycloak["Keycloak Identity Provider"]
        Realm["Realm: stack4things"]
        OIDC["OIDC/OAuth2 Endpoints"]
        TokenVal["Token Validation"]
        Realm --> OIDC
        Realm --> TokenVal
    end
    
    subgraph S4T["Stack4Things IoTronic"]
        Adapter["Keycloak Adapter<br/>- JWT Validation<br/>- Role Mapping"]
        Legacy["Keystone Compatibility<br/>(Backward Compat)"]
        API["IoTronic API Server"]
        Adapter --> API
        Legacy --> API
    end
    
    subgraph Devices["IoT Devices"]
        LR["Lightning-Rod Agents"]
    end
    
    CP -->|OAuth2 Client Credentials| Keycloak
    Keycloak -->|JWT Token| CP
    IC -->|API Calls with JWT| S4T
    S4T -->|Commands via WAMP| Devices
    LR -->|WebSocket Tunnel| S4T
```

### Diagramma Flusso Autenticazione

```mermaid
sequenceDiagram
    participant CP as Crossplane Provider
    participant KC as Keycloak
    participant IC as IoTronic Client
    participant IOT as IoTronic API
    participant LR as Lightning-Rod
    
    CP->>KC: 1. Client Credentials Request<br/>(client_id, client_secret)
    KC->>CP: 2. Access Token + Refresh Token<br/>(JWT)
    CP->>CP: 3. Cache Token
    CP->>IC: 4. API Request with JWT<br/>(Authorization: Bearer <token>)
    IC->>IOT: 5. Forward Request
    IOT->>KC: 6. Validate JWT Token
    KC->>IOT: 7. Token Valid + Claims
    IOT->>IC: 8. API Response
    IC->>CP: 9. Response
    
    Note over CP,KC: Token Refresh (before expiration)
    CP->>KC: 10. Refresh Token Request
    KC->>CP: 11. New Access Token
    CP->>CP: 12. Update Cached Token
```

### Diagramma Interazione Componenti

```mermaid
graph LR
    subgraph User["User/Operator"]
        KUBECTL["kubectl apply"]
        YAML["YAML Manifest"]
    end
    
    subgraph Crossplane["Crossplane Provider"]
        Reconciler["Reconciler"]
        Auth["Keycloak Auth"]
        Client["S4T Client"]
    end
    
    subgraph Keycloak["Keycloak"]
        Realm["Realm"]
        Clients["Clients"]
        Roles["Roles"]
    end
    
    subgraph S4T["Stack4Things"]
        IoTronic["IoTronic"]
        WAMP["WAMP Router"]
    end
    
    subgraph Edge["Edge Devices"]
        LR["Lightning-Rod"]
        Plugin["Plugins"]
    end
    
    User -->|1. Apply CRD| Crossplane
    Reconciler -->|2. Authenticate| Auth
    Auth -->|3. Get Token| Keycloak
    Keycloak -->|4. JWT Token| Auth
    Reconciler -->|5. API Call| Client
    Client -->|6. JWT Auth| IoTronic
    IoTronic -->|7. Validate| Keycloak
    Keycloak -->|8. Claims| IoTronic
    IoTronic -->|9. Command| WAMP
    WAMP -->|10. WebSocket| LR
    LR -->|11. Execute| Plugin
```

### Diagramma Strategia Migrazione

```mermaid
graph TB
    subgraph Phase1["Fase 1: Dual Mode"]
        DM["Dual Mode Authentication"]
        KS["Keystone Support"]
        KC["Keycloak Support"]
        DM --> KS
        DM --> KC
    end
    
    subgraph Phase2["Fase 2: Migrazione Graduale"]
        User1["User Group 1<br/>(Migrati a Keycloak)"]
        User2["User Group 2<br/>(Ancora su Keystone)"]
        Translation["Token Translation Layer<br/>(Opzionale)"]
    end
    
    subgraph Phase3["Fase 3: Migrazione Completa"]
        KeycloakOnly["Keycloak Only"]
        Deprecated["Keystone Deprecato"]
    end
    
    Phase1 --> Phase2
    Phase2 --> Phase3
    Phase2 --> Translation
    Translation --> User1
    Translation --> User2
```

---

## Stack Tecnologico

- **Linguaggi:** Go (Crossplane Provider), Python (adapter IoTronic)
- **Framework:** Crossplane SDK, Keycloak Admin API
- **Protocolli:** OIDC, OAuth2, JWT
- **Infrastruttura:** Kubernetes, Keycloak
- **Tooling:** kubectl, Helm, GitOps tools

---

## Deliverable Attesi

### Codice e Implementazione
1. Crossplane Provider modificato con autenticazione Keycloak
2. Adapter IoTronic per validazione JWT Keycloak
3. Sistema di gestione token con refresh automatico
4. Tooling e script per migrazione

### Documentazione
1. Analisi del provider esistente
2. Guida configurazione Keycloak
3. Guida migrazione da Keystone a Keycloak
4. Documentazione API delle modifiche
5. Guida deployment

### Testing
1. Unit test per authentication package
2. Test di integrazione: Crossplane Provider ↔ Keycloak ↔ IoTronic
3. Test end-to-end con scenario completo
4. Test di performance (token refresh, chiamate API)
5. Test di sicurezza (validazione token, expiration)

### Use Case
1. Deployment multi-cloud usando Crossplane con autenticazione Keycloak
2. Scenario multi-tenant con accesso role-based

---

## Bibliografia e Riferimenti

### Documentazione RETROSPECT e Stack4Things
- `RETROSPECT_Deliverable_D1_3_v2/main_final.tex` - Sezione "Kubernetes + S4T + Crossplane: Unified Orchestration"
- `RETROSPECT_Deliverable_D3_1_v2/main_final.tex` - Sezione "Kubernetes and Stack4Things Integration"
- `AssemblingSmartCPSs/chapter8.tex` - Meccanismi di autenticazione Stack4Things con Keystone
- `AssemblingSmartCPSs/chapter4.tex` - Architettura Stack4Things e servizio IoTronic
- `AssemblingSmartCPSs/chapter11.tex` - Uso Keycloak nel progetto SLICES
- `RETROSPECT_Deliverable_D2_2_v2/main_final.tex` - Sezione "Implementation of Stack4Things Client Library"
- Stack4Things GitHub: https://github.com/MDSLab/Stack4Things
- Documentazione API IoTronic Stack4Things: https://github.com/MDSLab/iotronic

### Documentazione Crossplane
- Documentazione Ufficiale Crossplane: https://crossplane.io/docs
- Guida Sviluppo Provider Crossplane: https://crossplane.io/docs/v1.11/concepts/providers
- Documentazione Crossplane SDK: https://github.com/crossplane/crossplane
- Configurazione Provider Crossplane: https://crossplane.io/docs/v1.11/concepts/provider-config
- Resource Reconciliation Crossplane: https://crossplane.io/docs/v1.11/concepts/managed-resources
- Composition Functions Crossplane: https://crossplane.io/docs/v1.11/concepts/composition-functions

### Documentazione Keycloak
- Documentazione Ufficiale Keycloak: https://www.keycloak.org/documentation
- Guida Amministrazione Keycloak Server: https://www.keycloak.org/docs/latest/server_admin/
- Keycloak Authorization Services: https://www.keycloak.org/docs/latest/authorization_services/
- Keycloak Admin REST API: https://www.keycloak.org/docs-api/latest/rest-api/
- Keycloak Java Adapter: https://www.keycloak.org/docs/latest/securing_apps/
- Configurazione Keycloak OIDC/OAuth2: https://www.keycloak.org/docs/latest/securing_apps/

### Standard e Protocolli OIDC/OAuth2
- RFC 6749 - OAuth 2.0 Authorization Framework: https://datatracker.ietf.org/doc/html/rfc6749
- RFC 6750 - OAuth 2.0 Bearer Token Usage: https://datatracker.ietf.org/doc/html/rfc6750
- RFC 7519 - JSON Web Token (JWT): https://datatracker.ietf.org/doc/html/rfc7519
- OpenID Connect Core 1.0: https://openid.net/specs/openid-connect-core-1_0.html
- OAuth 2.0 Device Flow (RFC 8628): https://datatracker.ietf.org/doc/html/rfc8628
- OAuth 2.0 Token Introspection (RFC 7662): https://datatracker.ietf.org/doc/html/rfc7662

### Documentazione Kubernetes
- Kubernetes Custom Resource Definitions: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
- Kubernetes Controller Pattern: https://kubernetes.io/docs/concepts/architecture/controller/
- Autenticazione Kubernetes API Server: https://kubernetes.io/docs/reference/access-authn-authz/authentication/
- Autorizzazione Kubernetes RBAC: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Kubernetes Service Accounts: https://kubernetes.io/docs/concepts/security/service-accounts/

### Linguaggio di Programmazione Go
- Documentazione Ufficiale Go: https://go.dev/doc/
- Go OAuth2 Client Library: https://pkg.go.dev/golang.org/x/oauth2
- Go JWT Libraries: https://github.com/golang-jwt/jwt
- Go HTTP Client: https://pkg.go.dev/net/http

### Librerie Python (per Adapter IoTronic)
- Python Requests Library: https://docs.python-requests.org/
- Python JWT Library: https://pyjwt.readthedocs.io/
- Python Keycloak Admin Client: https://python-keycloak.readthedocs.io/

### Risorse Aggiuntive
- Documentazione OpenStack Keystone: https://docs.openstack.org/keystone/
- Protocollo WebSocket (RFC 6455): https://datatracker.ietf.org/doc/html/rfc6455
- Documentazione Protocollo WAMP: https://wamp-proto.org/
- Integrazione WAMP Stack4Things: https://github.com/MDSLab/Stack4Things/wiki

---

## Note

Questo progetto si concentra specificamente sull'integrazione del Crossplane Provider con Stack4Things, esplorando la migrazione da Keystone a Keycloak mantenendo la compatibilità con l'infrastruttura Stack4Things esistente.


# User-and-Credential-Management_T6.1
The User and Credential Management component provides authentication, authorization for all CoEvolution Hub components. It ensures that verified users and processes can access sensitive model information, datasets, orchestration features. It manages password encryption, role assignments, and token issuance for inter-component communication.

# User & Credential Management (UCM) — CoEvolution Hub Backend

The **User and Credential Management (UCM)** component of the CoEvolution Hub Backend provides **authentication, authorization, and identity management services** for all Hub components.

Its main objective is to ensure that **only authenticated and authorized users or services** can access protected Hub resources (models, datasets, orchestration features) using **OAuth2 / OpenID Connect (OIDC)** and **Role-Based Access Control (RBAC)**.

---

## 1. Purpose and Responsibilities

The UCM component is responsible for:

- User registration and identity management  
- Secure credential management (hashed passwords, policies)  
- Role and permission management (RBAC)  
- Token issuance (JWT)  
- Token validation and access control enforcement  
- Audit logging for traceability and accountability  
- Exposure of REST and OIDC-compliant endpoints over HTTPS  

---

## 2. Technology Stack (as defined in the Architecture Deliverable)

- **IAM Solution:** Keycloak  
- **Runtime Environment:** Java 21  
- **Database:** PostgreSQL  
- **Interfaces:** REST APIs + OpenID Connect endpoints  
- **Security:** OAuth2, OIDC, RBAC, TLS encryption  
- **Deployment:** Docker-based standalone container  
- **Persistence:** Persistent storage for users, roles, credentials  

### Recommended Hardware Requirements

- Minimum: 2–4 CPU cores, 8GB RAM, 20GB disk  
- Recommended: 4+ CPU cores, 16GB RAM, 40GB disk  

---

## 3. Architectural Role in the CoEvolution Hub

The UCM acts as the **central authentication and authorization authority** of the CoEvolution Hub.

### Interacting Components

- **Dashboard (Frontend)**  
  Performs login operations and uses access tokens when invoking Hub Backend APIs.

- **Hub Backend Services**  
  Validate JWTs and enforce RBAC policies.

- **Orchestrator (if applicable)**  
  May request tokens for service-to-service authentication.

The UCM serves as the **security gatekeeper** of the platform.

---

## 4. Inputs and Outputs

### 4.1 Inputs

- User credentials (username/password)
- Access requests
- Token validation requests

**Format:** JSON payloads  
**Trigger:** Login attempt or token validation request

---

### 4.2 Outputs

- JWT Access Tokens  
- Refresh Tokens (if enabled)  
- User role metadata (claims inside tokens)  
- Audit log entries  

These outputs are consumed by all Hub components requiring authentication.

---

## 5. Authentication & Authorization Model

The UCM is implemented using **Keycloak** as the Identity and Access Management (IAM) solution.

### 5.1 Realm Structure (Recommended Setup)

- **Realm:** `coevolution`

### 5.2 Clients (Example Configuration)

- `coevolution-dashboard`  
  Used by the frontend application (Authorization Code Flow)

- `coevolution-hub-backend`  
  Confidential client for backend services

- `coevolution-orchestrator` (optional)  
  For service-to-service communication

---

### 5.3 Role-Based Access Control (RBAC)

Example roles:

- `admin` — Full administrative privileges  
- `developer` — Access to development and model-related features  
- `operator` — Orchestration-related permissions  
- `auditor` — Read-only access to logs and reports  
- `viewer` — Dashboard read-only access  

Roles are included as **claims inside JWT tokens** and enforced by downstream services.

---

### 5.4 Token Strategy

The system issues **JWT tokens** compliant with OIDC.

Supported flows:

- **Authorization Code Flow** (for frontend login)
- **Client Credentials Flow** (for service-to-service authentication)

Typical JWT claims:

- `sub`
- `preferred_username`
- `realm_access.roles`
- `aud`
- `exp`
- `iss`

All downstream services must:

1. Verify JWT signature  
2. Validate issuer (`iss`) and audience (`aud`)  
3. Check expiration  
4. Enforce RBAC policies  

---

## 6. Exposed Endpoints

The UCM exposes standard **OIDC endpoints**, including:

- Authorization Endpoint  
- Token Endpoint  
- UserInfo Endpoint  
- JWKS Endpoint (public keys)  
- Token Introspection Endpoint (if enabled)  

Example OIDC discovery endpoint:


All endpoints are exposed via **HTTPS (TLS-secured)** communication.

---

## 7. Security Considerations

- Passwords are stored using secure hashing algorithms  
- TLS encryption for all communication  
- Token-based API protection  
- Audit logging for login attempts and access events  
- GDPR-aligned minimal storage of personal data  
- Role-based restriction of administrative operations  

Target authentication latency: **<200ms per request** under nominal load.

---

## 8. Deployment

### 8.1 Containerization

The UCM is deployed as a **standalone Docker container**.

It requires:

- Persistent volume for Keycloak data  
- PostgreSQL database connection  
- Network access from Hub components  

---

### 8.2 Example Environment Variables (Template)

```bash
KC_DB=postgres
KC_DB_URL=jdbc:postgresql://postgres:5432/keycloak
KC_DB_USERNAME=keycloak
KC_DB_PASSWORD=<secure_password>

KC_HOSTNAME=ucm.coevolution.local
KC_HTTP_ENABLED=false
KC_HTTPS_PORT=8443
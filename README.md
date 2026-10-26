# CoEvolution Hub: User & Credential Management (T6.1)

The **User & Credential Management (UCM)** component is the central authentication and authorization service of the **CoEvolution Hub**. It makes sure that only verified users and services can reach protected Hub resources: AI models, datasets, reports and orchestration features.

UCM is built on **[Keycloak](https://www.keycloak.org/)**, an open-source Identity and Access Management (IAM) server, configured for CoEvolution. It provides:

- login through **OAuth 2.0 / OpenID Connect (OIDC)**;
- **Role-Based Access Control (RBAC)**;
- issuing of **JWT** tokens for users and for component-to-component communication.

The component is developed in **WP6 / Task 6.1** and is described in deliverable **D6.2** as part of the Hub MVP.

> **Part of:** CoEvolution Hub Backend (WP6 / T6.1) · **Type:** Identity & Access Management · **Based on:** Keycloak · **Maintainer:** AEGIS IT Research

---

## Contents

- [What UCM does](#what-ucm-does)
- [Architecture](#architecture)
- [What is CoEvolution-specific](#what-is-coevolution-specific)
- [Quick start (Docker Compose)](#quick-start-docker-compose)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Realm, clients and roles](#realm-clients-and-roles)
- [Token model](#token-model)
- [Integrating a Hub or partner component](#integrating-a-hub-or-partner-component)
- [Endpoints](#endpoints)
- [Security](#security)
- [Verifying the deployment](#verifying-the-deployment)
- [Troubleshooting](#troubleshooting)
- [Project structure](#project-structure)
- [Acknowledgements and licence](#acknowledgements-and-licence)

---

## What UCM does

| Responsibility | How it is provided |
| --- | --- |
| User registration and identity management | Keycloak users in the `coevolution` realm |
| Secure credential management | Salted password hashing and configurable password policies |
| Role and permission management (RBAC) | Realm roles, assigned to users and service accounts |
| Token issuance | OIDC-compliant JWT access tokens and refresh tokens |
| Token validation and access control | Downstream services verify tokens against the UCM public keys (JWKS) |
| Audit logging | Keycloak login and admin events |
| Standard interfaces | OIDC endpoints and the Keycloak Admin REST API, over HTTPS |

### Inputs and outputs

| | Content | Format | Trigger |
| --- | --- | --- | --- |
| **Inputs** | User credentials, client credentials, token validation and introspection requests | HTTP form / JSON | Login, service authentication, API call |
| **Outputs** | JWT access tokens, refresh tokens, role claims, audit events | JWT / JSON | Successful authentication or validation |

---

## Architecture

```
                    ┌──────────────────────────────┐
  Dashboard ───────▶│                              │
  (Authorization    │  UCM: Keycloak               │──▶ PostgreSQL
   Code + PKCE)     │  realm: coevolution          │    (users, roles,
                    │                              │     credentials, sessions)
  Hub backend  ────▶│  /realms/coevolution/...     │
  services          │                              │
  (Client           └──────────────┬───────────────┘
   Credentials)                    │ JWKS (public keys)
                                   ▼
             Every Hub service verifies JWT signature, issuer,
             audience and expiry, then enforces RBAC from the roles
```

**Interacting components:**

- **Hub Dashboard (frontend):** signs users in with the Authorization Code flow and sends the access token with every call to the Hub Backend.
- **Orchestrator and other components:** validate JWTs and enforce RBAC on each request, obtain tokens with the Client Credentials flow to call other services.

---

## What is CoEvolution-specific

UCM uses the upstream Keycloak server. The CoEvolution work in this repository is the configuration around it:

| Item | Location in repo |
| --- | --- |
| `coevolution` realm definition: clients, roles, policies, event settings | `/realm-export.json`|

The realm is **imported automatically on first start**, so a new deployment gets the full CoEvolution configuration without manual steps.

---

## Quick start (Docker Compose)

```bash
git clone <REPOSITORY_URL>
cd User-and-Credential-Management_T6.1
cp .env.example .env        # set the passwords (see Configuration)
docker compose up -d
```

Once the containers are running (about a minute on first start):

- **Admin console:** `http://localhost:8080/admin/`. Log in with the bootstrap admin from `.env`.
- **OIDC discovery:** `http://localhost:8080/realms/coevolution/.well-known/openid-configuration`

> [!WARNING]
> The quick start runs over HTTP for local testing only. Production deployments must use HTTPS (see [Deployment](#deployment)).

### Reference `docker-compose.yaml`

<!-- TODO: replace with the repository's actual compose file if it differs -->
```yaml
# ── Keycloak ──────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      nodeSelector:
        node-role: worker
      imagePullSecrets:
      - name: regcred
      containers:
      - name: keycloak
        image: keycloak/keycloak:26.0
        args: ["start", "--import-realm"]
        ports:
        - containerPort: 8081
        env:
        - name: KEYCLOAK_ADMIN
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: KEYCLOAK_ADMIN
        - name: KEYCLOAK_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: KEYCLOAK_ADMIN_PASSWORD
        # Use the real domain so tokens have correct issuer URL
        - name: KC_HOSTNAME
          value: "aegis.coevolution-project.eu"
        - name: KC_HTTP_PORT
          value: "8081"
        - name: KC_PROXY
          value: "edge"               # tells Keycloak it sits behind a proxy
        - name: KC_HOSTNAME_STRICT_BACKCHANNEL
          value: "false"
        volumeMounts:
        - name: keycloak-data
          mountPath: /opt/keycloak/data
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
      volumes:
      - name: keycloak-data
        persistentVolumeClaim:
          claimName: keycloak-data-pvc
 
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
spec:
  type: NodePort
  selector:
    app: keycloak
  ports:
  - name: http
    port: 8081
    targetPort: 8081
    nodePort: 30012     
```

---

## Configuration

Configuration uses standard Keycloak environment variables. The most relevant ones are listed below; see the [Keycloak configuration guide](https://www.keycloak.org/server/all-config) for the rest.

| Variable | Required | Description | Example |
| --- | --- | --- | --- |
| `KC_DB` | Yes | Database vendor | `postgres` |
| `KC_DB_URL` | Yes | JDBC URL of the database | `jdbc:postgresql://postgres:5432/keycloak` |
| `KC_DB_USERNAME` / `KC_DB_PASSWORD` | Yes | Database credentials | from `.env` |
| `KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD` | First start | Initial admin account (Keycloak 26+; older versions use `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD`) | from `.env` |
| `KC_HOSTNAME` | Production | Public URL of UCM, which becomes the token issuer (`iss`) | `https://auth.example.eu` |
| `KC_HTTP_RELATIVE_PATH` | If needed | Path prefix when UCM is not served at the root | `/auth` |
| `KC_PROXY_HEADERS` | Behind a proxy | Trust forwarded headers from the reverse proxy | `xforwarded` |
| `KC_HTTP_ENABLED` | Behind a proxy | Allow HTTP between proxy and Keycloak when TLS ends at the proxy | `true` |
| `KC_HTTPS_CERTIFICATE_FILE` / `KC_HTTPS_CERTIFICATE_KEY_FILE` | Direct TLS | Certificate and key when Keycloak terminates TLS itself | `/certs/tls.crt` |

**Never commit real passwords.** Keep them in `.env` locally (git-ignored) or in Kubernetes Secrets.

The public URL in `KC_HOSTNAME` must match the `KEYCLOAK_URL` configured in the Hub Dashboard and the issuer expected by backend services. If they differ, token validation fails.

---

## Deployment

### Requirements

| | Minimum | Recommended |
| --- | --- | --- |
| CPU | 2–4 cores | 4+ cores |
| RAM | 8 GB | 16 GB |
| Disk | 20 GB | 40 GB |

These figures, from the Hub architecture specification, cover Keycloak and PostgreSQL together. Software requirements are Docker Engine 18.06+ and Docker Compose 1.28+, or a Kubernetes cluster. Java 21 is already included in the Keycloak image.

### Production checklist

1. Start Keycloak with `start --import-realm`, not `start-dev`.
2. Choose one TLS setup:
   - **TLS at a reverse proxy or Ingress (recommended):** set `KC_PROXY_HEADERS=xforwarded` and `KC_HTTP_ENABLED=true`.
   - **TLS in Keycloak itself:** provide the certificate files and expose port 8443.
3. Set `KC_HOSTNAME` to the public HTTPS URL.
4. Use a persistent volume for PostgreSQL and take regular database backups.
5. Change the bootstrap admin password, or replace that account with a named admin user.
6. Restrict access to `/admin/` at the proxy if it should not be public.

---

## Realm, clients and roles

### Realm

| Realm | Purpose |
| --- | --- |
| `coevolution` | All CoEvolution Hub users, clients and roles |

### Clients

| Client ID | Type | Flow | Used by |
| --- | --- | --- | --- |
| `coevolution-dashboard` | Public | Authorization Code + PKCE | Hub Dashboard (browser) |
| `coevolution-hub-backend` | Confidential | Client Credentials / bearer-only validation | Hub Backend services |
| `coevolution-orchestrator` | Confidential | Client Credentials | Orchestrator (service-to-service) |
<!-- TODO: align client IDs with the realm export and add partner clients if any -->

### Roles (RBAC)

| Role | Permissions |
| --- | --- |
| `admin` | Full administrative privileges |
| `developer` | Development and model-related features |
| `enduser` | Read-only access to logs, reports, and dashboard |


---

## Token model

UCM issues OIDC-compliant **JWT access tokens** and, for interactive users, **refresh tokens**.

| Flow | Used for |
| --- | --- |
| Authorization Code + PKCE | User login from the Dashboard |
| Client Credentials | Service-to-service calls between Hub components |


---

## Integrating a Hub or partner component

To protect a component with UCM:

1. **Request a client** in the `coevolution` realm from the Hub administrators. Use a confidential client for backends and a public client for browser apps.
2. **Fetch the public keys** from the JWKS endpoint and cache them.
3. **On every request**, validate the bearer token.
4. **To call another component**, obtain a token with the Client Credentials flow and send it as `Authorization: Bearer <token>`.

Keycloak lists maintained OIDC libraries for most stacks (Python/FastAPI, Node.js, Java, Angular), so you do not need to write token validation yourself.

---

## Endpoints

All endpoints are relative to `https://<ucm-host>/realms/coevolution`. If `KC_HTTP_RELATIVE_PATH` is set, prefix them with that path.

| Endpoint | Path |
| --- | --- |
| OIDC discovery | `/.well-known/openid-configuration` |
| Authorization | `/protocol/openid-connect/auth` |
| Token | `/protocol/openid-connect/token` |
| UserInfo | `/protocol/openid-connect/userinfo` |
| JWKS (public keys) | `/protocol/openid-connect/certs` |
| Token introspection | `/protocol/openid-connect/token/introspect` |
| Logout | `/protocol/openid-connect/logout` |
| Admin REST API | `https://<ucm-host>/admin/realms/coevolution/...` |

The discovery document lists every endpoint, so clients only need the discovery URL.

---


## Verifying the deployment

**1. Discovery document.** It should return JSON with `issuer` equal to your public realm URL.

```bash
curl -s https://<ucm-host>/realms/coevolution/.well-known/openid-configuration
```

**2. Service token (Client Credentials).**

```bash
curl -s -X POST https://<ucm-host>/realms/coevolution/protocol/openid-connect/token \
  -d grant_type=client_credentials \
  -d client_id=<client-id> \
  -d client_secret=<client-secret>
```

The response should contain an `access_token`. Paste it into a JWT decoder and check `iss`, `aud` and `realm_access.roles`.

**3. User login.** Open the Hub Dashboard. You should be redirected to the UCM login page, then returned to the dashboard after signing in.

---

## Acknowledgements and licence

This project has received funding from the European Union's **Horizon Europe** research and innovation programme under **Grant Agreement No. 101091996** (CoEvolution).

Keycloak is licensed under the Apache License 2.0. CoEvolution-specific configuration is licensed under the terms in the [LICENSE](LICENSE) file.

## Contact

- Stella Markopoulou, AEGIS IT Research (<s.markopoulou@aegisresearch.eu>)

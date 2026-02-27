
---

# 📄 docker.md

```markdown
# Dockerization & Deployment Guide  
Component: User & Credential Management (UCM)  
Task ID: [TASK_ID]

---

## 1. Purpose

This document defines:

- Docker requirements
- Build instructions
- Runtime configuration
- Deployment validation
- Integration requirements

All deployable components must include a working Dockerfile according to repository guidelines.

---

## 2. Dockerfile Requirements

The docker/ folder must include:

/docker/
└── Dockerfile

The Dockerfile must:

- Use official Keycloak image (or approved base image)
- Configure PostgreSQL connection
- Expose HTTPS port (8443)
- Avoid hardcoded credentials
- Support environment-based configuration

---

## 3. Environment Variables

Example configuration:

KC_DB=postgres  
KC_DB_URL=jdbc:postgresql://postgres:5432/keycloak  
KC_DB_USERNAME=keycloak  
KC_DB_PASSWORD=<secure_password>  

KC_HOSTNAME=ucm.coevolution.local  
KC_HTTP_ENABLED=false  
KC_HTTPS_PORT=8443  

---

## 4. Build Instructions

From root folder:

docker build -t coevolution-ucm ./docker

---

## 5. Run Instructions

Using Docker:

docker run -p 8443:8443 \
  --env-file .env \
  coevolution-ucm

Using Docker Compose (recommended for integration):

- PostgreSQL container required
- Persistent volumes required
- Network must allow access from Hub services

---

## 6. Kubernetes Readiness (Optional but Recommended)

If deploying to k8s:

- Provide deployment.yaml
- Provide service.yaml
- Use secrets for DB credentials
- Define readiness & liveness probes
- Enable horizontal scaling if required

---

## 7. Security Requirements

- TLS must be enabled
- Secrets must not be committed
- Use secure password policies
- Enable audit logging
- Rotate client secrets periodically

---

## 8. Validation Checklist

Before integration:

- [ ] Docker image builds successfully
- [ ] Container starts without errors
- [ ] Database connectivity confirmed
- [ ] OIDC discovery endpoint reachable
- [ ] JWT issuance confirmed
- [ ] HTTPS enforced

---

## 9. Milestone Reporting

Upon milestone completion:

- Confirm container compatibility with D3.2
- List external dependencies (PostgreSQL, certificates, etc.)
- Confirm integration tests executed
- Update status.md
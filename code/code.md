# Code Structure & Development Guidelines  
Component: User & Credential Management (UCM)  
Task ID: [TASK_ID]

---

## 1. Purpose

This document describes:

- The internal code structure of the UCM component
- Coding standards and branching model
- API exposure rules
- Integration readiness requirements
- Testing and merge policy

This file complements:
- README.md (functional overview)
- docs/architecture.md (D3.2-based technical documentation)
- docker/Dockerfile (deployment instructions)

---

## 2. Code Directory Structure

/code/
│
├── src/
│   ├── main/
│   │   ├── java/ (if custom extensions are implemented)
│   │   ├── resources/
│   │   └── config/
│   └── test/
│
├── k8s/
│   └── deployment.yaml
│   └── service.yaml
│
├── api-spec/
│   └── openapi.yaml (if applicable)
│
└── scripts/
    └── init-realm.sh
    └── create-client.sh

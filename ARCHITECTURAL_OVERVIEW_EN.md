## Component Description

### 1. Frontend – “Helm Chart Studio”

**Technology: React + TypeScript + TailwindCSS**
**Main Elements:**

- Dashboard – existing projects and their linting status

- Project Wizard – step-by-step form:
  - General info (name, namespace, image)
  - Components (Deployment, PVC, Ingress, etc.)
  - ConfigMap & Secret management
  - Resource limits
- YAML Preview – real-time rendering from backend
- Validation panel – displays helm lint results
- Export – downloadable .tgz or “Push to Git”

**User Interfaces:**

- Dynamic forms — adding a component adds the corresponding section.
- “+ Add Component” button → opens a dialog with options (PVC, Ingress, etc.)

### 2. Backend – “Helm Chart Generator API”**

**Technology: Python + FastAPI**
**Functions:**

| Endpoint          | Description                                               |
| ----------------- | ------------------------------------------------------- |
| `POST /generate`  | Võtab sisendi, genereerib charti failid ja pakib `.tgz` |
| `POST /lint`      | Käivitab `helm lint` genereeritud chartil               |
| `POST /preview`   | Tagastab YAML-i (renderdatud chart) ilma salvestamata   |
| `POST /validate`  | Käivitab `kubeval` ja tagastab valideerimise tulemuse   |
| `GET /blueprints` | Tagastab eelseadistatud projektiprofiilid               |
| `GET /templates`  | Tagastab saadaval olevad komponendid ja nende väljad    |

**Additional Details:**

- The chart is created in a temporary workspace (/tmp/chart-{uuid})
- Linting and validation occur after generation
- If all checks pass, the chart is packaged as .tgz and either returned in the response or pushed to a repository

### 3. Template Engine (Jinja2)

Structure:
```
templates/
├─ base/
│   ├─ Chart.yaml.j2
│   ├─ values.yaml.j2
│
└─ components/
    ├─ deployment.yaml.j2
    ├─ service.yaml.j2
    ├─ ingress.yaml.j2
    ├─ pvc.yaml.j2
    ├─ configmap.yaml.j2
    ├─ secret.yaml.j2
    ├─ cronjob.yaml.j2
```

**Rendering Logic:**

- The backend reads the input (JSON)
- For each enabled component (e.g., "ingress": true), the corresponding template is rendered
- All files are written into the templates/ directory
- values.yaml is populated based on input
- Finally, helm lint is executed for verification

### 4. Lint & Validation Layer

- helm lint: ensures chart structural correctness
- kubeval or kubeconform: validates resources against the Kubernetes schema
- Custom lint rules (Python):
  - Checks for presence of resources.limits
  - Checks for presence of labels
  - Checks for duplicate ConfigMap keys
  - Checks for specific naming conventions

Results are returned to the Frontend in JSON:
```
{
  "status": "ok",
  "warnings": [],
  "errors": [
    "Deployment 'ruuter' missing resource limits"
  ]
}
```

### 5. Output & Storage

- The chart is packaged into a .tgz file (using helm package)
- Options:
  - Download from the UI
  - Automatic Git push to a specific repository (e.g., infrastructure/helm-charts)

### 6. Audit

- Audit log (who generated it, when, which chart)
- Version control (chart revision history)

**NOTE:**

**Data Flow (Sequence Diagram)**
```
User
 │
 │ 1. Fills out the form in the UI
 ▼
Frontend (React)
 │
 │ 2. POST /generate (JSON)
 ▼
Backend (FastAPI)
 │
 │ 3. Render templates (Jinja2)
 │ 4. helm lint + kubeval
 ▼
Lint & Validation
 │
 │ 5. Return results
 ▼
Frontend
 │
 │ 6. Display YAML preview + errors
 │ 7. “Download chart” or “Push to Git”
 ▼
Output storage (repo/bucket)
```

**Example Data Model** (/generate)
```
{
  "project": "ruuter",
  "namespace": "staging",
  "image": "ghcr.io/company/ruuter:1.0.0",
  "replicas": 3,
  "components": {
    "deployment": {
      "resources": {
        "limits": {"cpu": "500m", "memory": "512Mi"},
        "requests": {"cpu": "250m", "memory": "256Mi"}
      },
      "env": {"LOG_LEVEL": "info"}
    },
    "service": {"port": 8080},
    "ingress": {
      "host": "ruuter.example.com",
      "tls": true,
      "path": "/"
    },
    "pvc": {
      "name": "data",
      "size": "5Gi",
      "accessMode": "RWX"
    },
    "configmap": {
      "app-config": {
        "APP_ENV": "production",
        "CACHE_ENABLED": "true"
      }
    }
  }
}
```

**Architecture Extension**

- TBA

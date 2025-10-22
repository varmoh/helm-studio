## Komponentide kirjeldus

### 1. Frontend – “Helm Chart Studio”

**Tehnoloogia: React + TypeScript + TailwindCSS**  
**Põhielemendid:**

- Dashboard – olemasolevad projektid, nende lintimise staatus

- Project Wizard – samm-sammult vorm:
  - Üldinfo (nimi, namespace, image)
  - Komponendid (Deployment, PVC, Ingress jne)
  - ConfigMap & Secret haldus
  - Resource limits
- YAML Preview – reaalajas renderdus backendist
- Validation paneel – kuvab helm lint tulemused
- Export – allalaaditav .tgz või “Push to Git”

**Kasutajaliidesed:**

- Formid, mis on dünaamilised — komponendi lisamine lisab vastava sektsiooni.
- “+ Add Component” nupp → toob dialoogi valikutega (PVC, Ingress, jne).


### 2. Backend – “Helm Chart Generator API”

**Tehnoloogia: Python + FastAPI**  
**Funktsioonid:**

| Endpoint          | Kirjeldus                                               |
| ----------------- | ------------------------------------------------------- |
| `POST /generate`  | Võtab sisendi, genereerib charti failid ja pakib `.tgz` |
| `POST /lint`      | Käivitab `helm lint` genereeritud chartil               |
| `POST /preview`   | Tagastab YAML-i (renderdatud chart) ilma salvestamata   |
| `POST /validate`  | Käivitab `kubeval` ja tagastab valideerimise tulemuse   |
| `GET /blueprints` | Tagastab eelseadistatud projektiprofiilid               |
| `GET /templates`  | Tagastab saadaval olevad komponendid ja nende väljad    |


**Lisaks:**

- Chart luuakse ajutises töökaustas (/tmp/chart-{uuid})
- Lintimine ja valideerimine käivad pärast genereerimist
- Kui kõik OK, pakitakse .tgz ja saadetakse vastuses või pushitakse repo-sse


### 3. Template Engine (Jinja2)

Struktuur:
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

**Renderdusloogika:**

- Backend loeb sisendi (JSON)
- Iga komponenti (nt “ingress”: true) puhul renderdatakse vastav mall
- Kõik failid kirjutatakse templates/ kausta
- `values.yaml` täidetakse sisendi põhjal
- Lõpuks käivitatakse helm lint kontroll

### 4. Lint & Validation Layer

- helm lint: tagab, et chart on struktuurselt korrektne
- kubeval või kubeconform: valideerib ressurssid Kubernetes schema vastu
- Custom lint rules (Python):
  - Kontrollib resources.limits olemasolu
  - Kontrollib labels olemasolu
  - Kontrollib ConfigMap key’ide duplikaate
  - Kontrollib kindlaid naming convention’e

Tulemused saadetakse JSON-is Frontendile:
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

- Chart pakitakse .tgz failiks (kasutades helm package)
- Võimalik:
  - Laadida alla UI-st
  - Automaatne git push kindlasse repo-sse (nt infrastructure/helm-charts)
 
### 6. Audit

- Audit log (kes genereeris, millal, mis chart)
- Versioonihaldus (chart revision ajalugu)

**LISA:**

**Andmevoog (Sequence diagramm)**

```
User
 │
 │ 1. Täidab vormi UI-s
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
 │ 5. Tagasta tulemused
 ▼
Frontend
 │
 │ 6. Kuvab YAML preview + vead
 │ 7. “Download chart” või “Push to Git”
 ▼
Output storage (repo/bucket)
```

**Andmemudeli näide** (/generate)

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

**Arhidektuuri laiendamine**

- TBA

# ☁️ GitOps Syncer — Kubernetes Manifest Automation Tool

![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg?logo=python&logoColor=white) ![kubectl](https://img.shields.io/badge/kubectl-required-326CE5.svg?logo=kubernetes) ![GitOps](https://img.shields.io/badge/GitOps-Automation-6C8EBF?logo=git) ![Docker](https://img.shields.io/badge/Docker-Optional-2496ED?logo=docker) ![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub_Actions-2088FF?logo=github-actions)

**GitOps Syncer** is a lightweight, dependency-free Python CLI that validates and applies Kubernetes manifests from a `manifests/` folder.  
Safe-by-default (dry-run unless `--apply` is used), notebook-safe (works in Jupyter/Colab), and CI/CD friendly.

--------

## 🛠 Tech & Languages

| Layer       | Tech                        | Notes |
|-------------|-----------------------------|-------|
| Language    | **Python 3.8+**             | Standard library only |
| CLI         | **argparse / subprocess**   | Executes `kubectl` commands |
| Hashing     | **hashlib**                 | Manifest fingerprinting (SHA-256) |
| Concurrency | **concurrent.futures**      | ThreadPoolExecutor for parallel apply |
| K8s Client  | **kubectl**                 | Required in PATH for apply operations |
| CI/CD       | **GitHub Actions** (example)| Dry-run on PRs, apply on protected branches |

---------

## 🌐 Architecture

<p align="center">
  <img src="https://raw.githubusercontent.com/your-org/gitops-syncer-assets/main/architecture.png" alt="GitOps Syncer Architecture" width="650" />
</p>

Flow:
1. Developer places/commits YAML manifests into `manifests/`.  
2. GitOps Syncer scans the directory for `.yml` / `.yaml` files.  
3. Computes SHA-256 fingerprint for change detection.  
4. Runs `kubectl apply -f <file>` for each manifest — default is `--dry-run=client`.  
5. Aggregates results, prints deterministic summary.  
6. Integrates with CI: run dry-run on PRs, run apply on protected merges.

---

## 📦 Repository Structure

project-gitops-syncer/
├─ gitops_syncer.py # Single-file CLI (code + optional embedded README)
├─ manifests/ # Put YAML manifests here
│ └─ test-configmap.yaml
├─ Dockerfile # Optional containerization
├─ .github/
│ └─ workflows/
│ └─ gitops-sync.yml # Example GitHub Actions workflow
└─ README.md # This file

yaml
Copy code

---

## ▶️ Run (Terminal & Google Colab)

### Create sample manifest
```bash
mkdir -p manifests
cat > manifests/test-configmap.yaml <<'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-cm
data:
  foo: bar
YAML
Terminal (dry-run)
bash
Copy code
python gitops_syncer.py --manifests=manifests
Terminal (apply)
bash
Copy code
python gitops_syncer.py --manifests=manifests --apply
Jupyter / Colab (programmatic)
python
Copy code
import gitops_syncer
gitops_syncer.main(['--manifests','manifests','--apply'])
🔗 CLI Options
Flag	Description
--manifests	Path to the folder containing YAML manifests (default: manifests)
--context	Kubernetes context to use (optional)
--apply	Actually apply manifests; otherwise runs dry-run
--parallel	Number of parallel apply threads (default: 4)
--verbose	Extra logging for debugging

💻 Examples
Validate manifests (dry-run)
bash
Copy code
python gitops_syncer.py --manifests=manifests
Apply manifests to cluster
bash
Copy code
python gitops_syncer.py --manifests=manifests --apply
Use a specific Kubernetes context
bash
Copy code
python gitops_syncer.py --manifests=manifests --context=staging --apply
🧪 Output Example
bash
Copy code
Found 1 manifest(s). dry_run=False
Manifests hash: 4a2c6e14eac3...
$ kubectl apply -f manifests/test-configmap.yaml
configmap/test-cm created
--- manifests/test-configmap.yaml (rc=0) ---
configmap/test-cm created
Sync complete: 1 succeeded, 0 failed.
[interactive] finished with return code: 0
🧩 CI/CD Integration (GitHub Actions)
yaml
Copy code
name: GitOps Sync
on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.10'
      - name: Install kubectl
        run: sudo apt-get update && sudo apt-get install -y kubectl
      - name: Dry-run manifests
        run: python gitops_syncer.py --manifests=manifests

  apply:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: [validate]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.10'
      - name: Install kubectl
        run: sudo apt-get update && sudo apt-get install -y kubectl
      - name: Apply manifests
        run: python gitops_syncer.py --manifests=manifests --apply
🐳 Docker
Dockerfile (example)
dockerfile
Copy code
FROM python:3.10-slim
WORKDIR /app
COPY gitops_syncer.py /app/gitops_syncer.py
COPY manifests/ /app/manifests/
ENTRYPOINT ["python", "gitops_syncer.py"]
Build & run
bash
Copy code
docker build -t gitops-syncer:latest .
docker run -v ~/.kube:/root/.kube -v $(pwd)/manifests:/app/manifests gitops-syncer:latest --apply
☸️ Kubernetes Scheduling Example
yaml
Copy code
apiVersion: batch/v1
kind: CronJob
metadata:
  name: gitops-syncer
spec:
  schedule: "0 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: gitops-syncer
              image: ghcr.io/your-org/gitops-syncer:latest
              args: ["--manifests=/data/manifests","--apply"]
              volumeMounts:
                - name: manifests
                  mountPath: /data/manifests
          restartPolicy: OnFailure
          volumes:
            - name: manifests
              configMap:
                name: app-manifests
📊 Metrics & Observability (suggested)
Expose job metrics in CI (duration, failures) as build artifacts.

Example metrics to collect:

gitops_sync_runs_total

gitops_sync_failures_total

gitops_sync_duration_seconds

🐛 Troubleshooting
Common issues
No manifests found: ensure manifests/ contains .yml/.yaml files.

kubectl not found: install and ensure it’s in PATH.

Cluster access errors: verify ~/.kube/config or KUBECONFIG env var and RBAC.

Permission denied (file write): run in writable directory or adjust permissions.

Quick checks
bash
Copy code
python --version
kubectl version --client --short
kubectl get ns
ls -la manifests
🔒 Security Best Practices
Run dry-run in PRs; require approvals for --apply.

Use least-privilege service accounts in CI.

Do not store secrets in plain YAML.

Add *.kube/config, *.json to .gitignore if applicable.

🤝 Contributing
Fork the repo

Create a feature branch git checkout -b feature/xyz

Commit changes git commit -m "Add xyz"

Push and open a PR


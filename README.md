GitOps Syncer is a Python-based DevOps tool that automates Kubernetes configuration management.
It scans a local or repository manifests/ directory, validates YAML manifests, computes a change fingerprint, and applies them to clusters using kubectl.

It is:

Safe by default — dry-run mode prevents unintended changes

Lightweight — only Python standard library, no dependencies

CI/CD-ready — works inside GitHub Actions, Jenkins, or GitLab pipelines

Cross-platform — runs on Linux, macOS, or Windows

🛠 Tech & Languages
Layer	Tech	Notes
Language	Python 3.8+	Standard library only
CLI Tooling	argparse / subprocess	Used for command-line interface and shell calls
Kubernetes	kubectl	Applies manifests, supports dry-run and context switching
Container	Docker	Optional for packaging
CI/CD	GitHub Actions	Example workflow provided below
🌐 Architecture
<p align="center"> <img src="https://raw.githubusercontent.com/your-org/gitops-syncer-assets/main/architecture.png" alt="GitOps Syncer Architecture" width="650" /> </p>

Flow:

Developer commits YAML manifests under manifests/.

GitOps Syncer scans the folder for .yaml or .yml files.

Computes a SHA-256 hash to detect configuration changes.

Executes kubectl apply -f <manifest> for each file.

Runs in dry-run mode by default; add --apply for real deployments.

Integrates easily with GitHub Actions or any CI/CD platform.

📦 Repository Structure
project-gitops-syncer/
├─ gitops_syncer.py          # Main CLI tool
├─ manifests/                # Kubernetes manifests
│   └─ test-configmap.yaml   # Example file
├─ Dockerfile                # Optional container
├─ .github/
│   └─ workflows/
│       └─ gitops-sync.yml   # Example CI/CD workflow
└─ README.md                 # Documentation

▶️ Run in Google Colab or Terminal
🧩 Setup
!apt-get update -y && apt-get install -y kubectl
mkdir -p manifests
cat > manifests/test-configmap.yaml <<'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-cm
data:
  foo: bar
YAML

🖥️ Terminal (CLI)
python gitops_syncer.py --manifests=manifests --apply

📘 Jupyter / Google Colab
import gitops_syncer
gitops_syncer.main(['--manifests', 'manifests', '--apply'])

🔗 CLI Commands
Command	Description
--manifests	Path to folder with YAML manifests
--context	Optional Kubernetes context
--apply	Apply manifests (otherwise dry-run)
--parallel	Number of threads for parallel apply (default: 4)
💻 Examples

Dry-run validation:

python gitops_syncer.py --manifests=manifests


Apply to the cluster:

python gitops_syncer.py --manifests=manifests --apply


Use a specific kubecontext:

python gitops_syncer.py --manifests=manifests --context=staging --apply

🧪 Output Example
Found 1 manifest(s). dry_run=False
Manifests hash: 4a2c6e14eac3...
$ kubectl apply -f manifests/test-configmap.yaml
configmap/test-cm created
--- manifests/test-configmap.yaml (rc=0) ---
configmap/test-cm created
Sync complete: 1 succeeded, 0 failed.
[interactive] finished with return code: 0

🧩 GitHub Actions (CI/CD Workflow)
name: GitOps Sync
on:
  push:
    branches: [ main ]
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
      - name: Install kubectl
        run: sudo apt-get install -y kubectl
      - name: Validate manifests (dry-run)
        run: python gitops_syncer.py --manifests=manifests
      - name: Apply manifests (main branch only)
        if: github.ref == 'refs/heads/main'
        run: python gitops_syncer.py --manifests=manifests --apply

🐳 Docker

Build image:

docker build -t gitops-syncer:latest .


Run container:

docker run -v ~/.kube:/root/.kube -v $(pwd)/manifests:/app/manifests gitops-syncer:latest --apply

☸️ Kubernetes Deployment

Deploy GitOps Syncer as a scheduled job or CronJob in your cluster:

apiVersion: batch/v1
kind: CronJob
metadata:
  name: gitops-syncer
spec:
  schedule: "*/30 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: gitops-syncer
              image: ghcr.io/your-org/gitops-syncer:latest
              args: ["--manifests=/data/manifests", "--apply"]
              volumeMounts:
                - name: manifests
                  mountPath: /data/manifests
          restartPolicy: OnFailure
          volumes:
            - name: manifests
              configMap:
                name: app-manifests

📊 Metrics Integration

While GitOps Syncer itself is lightweight, you can wrap it in a CI/CD job that emits custom metrics such as:

Metric	Description	Example
gitops_sync_runs_total	Number of GitOps sync executions	42
gitops_sync_failures_total	Failed apply operations	2
gitops_sync_last_duration_seconds	Duration of the last sync job	5.32

Prometheus scrape config snippet:

- job_name: "gitops-syncer"
  static_configs:
    - targets: ["gitops-syncer.default.svc.cluster.local:8080"]

🔐 Production Notes

Always run dry-run first before applying live changes.

Limit cluster permissions using RBAC or a dedicated service account.

Protect your main branch with approvals before auto-apply.

Optional: run inside a Kubernetes CronJob for recurring syncs.

👤 Author

Siddharth Raut — Cloud & DevOps Engineer
📧 Email: siduk2500@gmail.com

💼 LinkedIn: siddharth-raut-

📝 License

MIT License © 2025 Siddharth Raut

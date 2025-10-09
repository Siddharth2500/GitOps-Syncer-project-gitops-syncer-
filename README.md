#!/usr/bin/env python3
# =============================================================================
# GitOps Syncer — Full Single-File Project
#
# A lightweight GitOps automation tool written in Python 3 for applying
# Kubernetes manifests from a local 'manifests/' directory.
#
# =============================================================================
# README (Documentation Section)
# =============================================================================
#
# 📘 OVERVIEW
# ------------
# GitOps Syncer helps you synchronize Kubernetes manifests defined in a Git
# repository with a live cluster. It reads YAML manifests from a folder,
# validates and applies them using `kubectl`. By default, it runs safely in
# dry-run mode. You can apply real changes with the `--apply` flag.
#
# 🔧 TECH STACK
# -------------
# • Language: Python 3.8+
# • Tooling: Uses `kubectl` for applying manifests
# • Dependencies: None (standard library only)
# • OS Support: Linux, macOS, Windows (with `kubectl`)
# • Execution Modes: CLI, Jupyter Notebook, or Google Colab
#
# 📁 PROJECT STRUCTURE
# --------------------
# project-gitops-syncer/
# ├── gitops_syncer.py          # This file
# ├── manifests/                # Folder containing YAML manifests
# │   └── test-configmap.yaml   # Example manifest
#
# 🚀 FEATURES
# ------------
# ✅ Dry-run mode by default
# ✅ Parallel apply for faster sync
# ✅ Manifest hash detection for change tracking
# ✅ Works in Colab/Jupyter without crashing
# ✅ Perfect for CI/CD pipelines (GitHub Actions, Jenkins, etc.)
#
# 🧰 REQUIREMENTS
# ----------------
# 1. Python 3.8+ installed (`python3 --version`)
# 2. `kubectl` CLI installed and in your PATH
# 3. Kubeconfig configured (`kubectl config get-contexts`)
#
# 📦 INSTALLATION
# ----------------
# Clone and enter the repo:
#   git clone https://github.com/yourusername/project-gitops-syncer.git
#   cd project-gitops-syncer
#
# Optional (create a virtual environment):
#   python3 -m venv venv
#   source venv/bin/activate
#
# 🧩 SAMPLE MANIFEST
# -------------------
# Create a test manifest for validation:
#   mkdir -p manifests
#   echo "
#   apiVersion: v1
#   kind: ConfigMap
#   metadata:
#     name: test-cm
#   data:
#     foo: bar
#   " > manifests/test-configmap.yaml
#
# 🧰 USAGE
# ---------
# In Terminal:
#   python gitops_syncer.py --manifests=manifests --apply
#
# In Jupyter / Colab:
#   import gitops_syncer
#   gitops_syncer.main(['--manifests', 'manifests', '--apply'])
#
# Or:
#   !python gitops_syncer.py --manifests=manifests --apply
#
# 💡 OUTPUT EXAMPLE
# ------------------
# Found 1 manifest(s). dry_run=False
# Manifests hash: 2a4be134...
# $ kubectl apply -f manifests/test-configmap.yaml
# configmap/test-cm created
# --- manifests/test-configmap.yaml (rc=0) ---
# configmap/test-cm created
# Sync complete: 1 succeeded, 0 failed.
# [interactive] finished with return code: 0
#
# ⚙️ COMMAND OPTIONS
# -------------------
# --manifests   Directory containing YAML manifests (default: manifests)
# --context     Kubernetes context (optional)
# --apply       Actually apply manifests (default: dry-run)
# --parallel    Threads for parallel apply (default: 4)
#
# 🧩 GITHUB ACTIONS EXAMPLE
# --------------------------
# name: GitOps Sync
#
# on:
#   push:
#     branches: [ main ]
#
# jobs:
#   apply:
#     runs-on: ubuntu-latest
#     steps:
#       - uses: actions/checkout@v4
#       - uses: actions/setup-python@v5
#         with:
#           python-version: '3.10'
#       - run: sudo apt-get install -y kubectl
#       - run: python gitops_syncer.py --manifests=manifests
#       - run: python gitops_syncer.py --manifests=manifests --apply
#
# 🧠 DESIGN PRINCIPLES
# ---------------------
# 1. Minimal dependencies (only standard library)
# 2. Explicit behavior (dry-run by default)
# 3. Transparent execution (prints every kubectl command)
#
# 🔐 SAFETY NOTES
# ----------------
# • Always test with dry-run before applying.
# • Use limited-permission kubeconfigs in CI/CD.
# • The script never stores secrets or credentials.
#
# 🪪 LICENSE
# -----------
# MIT License (c) 2025
#
# =============================================================================
# END OF README
# =============================================================================


from __future__ import annotations
import argparse
import os
import subprocess
import sys
import hashlib
from concurrent.futures import ThreadPoolExecutor
from typing import List, Tuple


# ---------------- Utilities ----------------
def run_cmd(cmd: str, capture: bool = False) -> subprocess.CompletedProcess:
    print(f"$ {cmd}")
    return subprocess.run(cmd, shell=True, capture_output=capture, text=True)

def list_manifests(manifests_dir: str) -> List[str]:
    files = []
    if not os.path.isdir(manifests_dir):
        return files
    for root, _, filenames in os.walk(manifests_dir):
        for fn in filenames:
            if fn.endswith(('.yml', '.yaml')):
                files.append(os.path.join(root, fn))
    return sorted(files)

def compute_repo_hash(manifests: List[str]) -> str:
    h = hashlib.sha256()
    for p in manifests:
        rel = os.path.normpath(p).encode('utf-8')
        h.update(rel)
        try:
            with open(p, 'rb') as f:
                h.update(f.read())
        except Exception:
            continue
    return h.hexdigest()

def apply_manifest(path: str, kubecontext: str | None = None, dry_run: bool = True) -> subprocess.CompletedProcess:
    cmd = f"kubectl apply -f \"{path}\""
    if kubecontext:
        cmd += f" --context {kubecontext}"
    if dry_run:
        cmd += " --dry-run=client"
    return run_cmd(cmd, capture=True)


# ---------------- Core Logic ----------------
def sync(manifests_dir: str = "manifests", kubecontext: str | None = None, dry_run: bool = True, parallel: int = 4) -> int:
    manifests = list_manifests(manifests_dir)
    if not manifests:
        print(f"No manifests found in '{manifests_dir}'. Nothing to do.")
        return 0

    print(f"Found {len(manifests)} manifest(s). dry_run={dry_run}")
    repo_hash = compute_repo_hash(manifests)
    print("Manifests hash:", repo_hash)

    results: list[Tuple[str, int, str, str]] = []

    def worker(path: str):
        try:
            res = apply_manifest(path, kubecontext=kubecontext, dry_run=dry_run)
            rc = res.returncode
            out = res.stdout or ""
            err = res.stderr or ""
        except Exception as e:
            rc = 1
            out = ""
            err = str(e)
        results.append((path, rc, out, err))

    with ThreadPoolExecutor(max_workers=max(1, parallel)) as ex:
        for m in manifests:
            ex.submit(worker, m)

    failed = 0
    for path, rc, out, err in results:
        print(f"--- {path} (rc={rc}) ---")
        if out:
            print(out.strip())
        if err:
            print(err.strip(), file=sys.stderr)
        if rc != 0:
            failed += 1

    print(f"Sync complete: {len(manifests) - failed} succeeded, {failed} failed.")
    return 1 if failed else 0


# ---------------- CLI Parser ----------------
def build_arg_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser(description="GitOps Syncer (safe for Jupyter/Colab)")
    p.add_argument('--manifests', default='manifests', help='directory containing Kubernetes manifests')
    p.add_argument('--context', default=None, help='kubecontext (optional)')
    p.add_argument('--apply', action='store_true', help='apply manifests (otherwise dry-run)')
    p.add_argument('--parallel', type=int, default=4, help='parallel threads for apply')
    return p

def in_interactive() -> bool:
    try:
        from IPython import get_ipython
        if get_ipython() is not None:
            return True
    except Exception:
        pass
    return bool(getattr(sys, 'ps1', False))


# ---------------- Main Entry ----------------
def main(argv: List[str] | None = None) -> int:
    parser = build_arg_parser()
    args, _unknown = parser.parse_known_args(argv)
    rc = sync(
        manifests_dir=args.manifests,
        kubecontext=args.context,
        dry_run=not args.apply,
        parallel=args.parallel
    )
    if in_interactive():
        print(f"[interactive] finished with return code: {rc}")
    return rc


if __name__ == "__main__":
    code = main()
    if not in_interactive():
        sys.exit(code)

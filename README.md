# llm-cc-in-k8s

Personal learning notes on **confidential computing for LLM serving on Kubernetes / GKE** — hardware TEEs (AMD SEV-SNP, Intel TDX), remote attestation, confidential GPUs (NVIDIA H100/Blackwell CC mode), the Google Cloud confidential surface, and how these compose into a verifiable third-party model-serving architecture.

Published site: <https://marinette101.github.io/llm-cc-in-k8s/>

Available in English and 简体中文 (use the language switcher in the site header).

## Build locally

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
mkdocs serve          # http://127.0.0.1:8000
mkdocs build --strict # fail on broken links / nav entries
```

Content lives in `docs/en/` and `docs/zh/` (same filenames per locale). Deployment is `mkdocs gh-deploy` via `.github/workflows/publish-docs.yml` on every push to `main`.

# Industrial Maintenance AI Agent

Diagnóstico asistido de la bomba centrífuga **P-204** (PLANT-A) con RAG, herramientas MCP, LangGraph, seguridad y gobierno. **Solo datos sintéticos**; ningún valor es un límite de seguridad industrial. Plan completo: [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md).

## Setup
```bash
python3 -m venv .venv && .venv/bin/pip install -e ".[dev]"
cp .env.example .env            # por defecto: todo en modo mock, sin red
```
## Seed (datos sintéticos)
```bash
.venv/bin/python scripts/generate_dataset.py && 
.venv/bin/python scripts/generate_eval_cases.py
.venv/bin/python scripts/ingest.py                      # embeddings -> store configurado
DATABASE_URL=... .venv/bin/python scripts/apply_migrations.py --seed   # solo STORE_MODE=supabase
```
## Ejecutar
```bash
.venv/bin/uvicorn app.api.main:app --reload
JWT_SECRET=... .venv/bin/python scripts/mint_token.py --role technician --plants PLANT-A --user ana@example.com
```
## Test y calidad
```bash
.venv/bin/ruff check app handlers scripts tests
.venv/bin/python -m pytest                 # offline; los tests de Postgres requieren TEST_DATABASE_URL
.venv/bin/python scripts/run_eval.py       # gates golden/adversarial
```
## Build y deploy
```bash
docker build -f Dockerfile.lambda -t maintenance-ai:lambda .
docker build -f Dockerfile.ec2 -t maintenance-ai:ec2 .
# Terraform: ver infrastructure/terraform/ (bootstrap -> environments/dev|workshop).
# NUNCA ejecutar `terraform apply` sin confirmación del operador. Post-deploy: scripts/smoke_test.py
```
## Guía por versión
Cada versión V01–V15 tiene su README en [labs/](labs/). Documentación adicional: [docs/](docs/).

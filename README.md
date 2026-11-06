# Industrial Maintenance AI Agent

Diagnóstico asistido de la bomba centrífuga **P-204** (PLANT-A) con RAG, herramientas MCP, LangGraph, seguridad y gobierno. **Solo datos sintéticos**; ningún valor es un límite de seguridad industrial. Plan completo: [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md).

> **¿Windows?** Las instalaciones y comandos equivalentes están en [docs/instalacion-windows.md](docs/instalacion-windows.md). Este README usa Linux (probado en Ubuntu con zsh).

## Índice

1. [Qué necesitas según lo que quieras hacer](#1-qué-necesitas-según-lo-que-quieras-hacer)
2. [Instalar las herramientas (Linux)](#2-instalar-las-herramientas-linux)
3. [Ejecutar en local (modo mock)](#3-ejecutar-en-local-modo-mock)
4. [Preparar Supabase](#4-preparar-supabase)
5. [Provisionar AWS con Terraform](#5-provisionar-aws-con-terraform)
6. [Secretos y verificación](#6-secretos-y-verificación)
7. [Problemas conocidos y su solución](#7-problemas-conocidos-y-su-solución)
8. [Limpiar y dejar de pagar](#8-limpiar-y-dejar-de-pagar)
9. [Test, calidad y guía por versión](#9-test-calidad-y-guía-por-versión)

---

## 1. Qué necesitas según lo que quieras hacer

| Objetivo | Necesitas | Cuesta |
|---|---|---|
| Probar el agente en local (mock) | Python 3.12+ | Nada |
| Usar Postgres real con pgvector | + cuenta de Supabase | Plan gratuito |
| Desplegar en AWS | + cuenta de AWS, AWS CLI, Terraform, Docker | Céntimos al mes sin EC2 |
| Usar un LLM real | + llave de OpenAI | Pago por uso |

El taller completo funciona **en modo mock sin AWS ni Supabase**. Las secciones 4 a 8 solo hacen falta para las partes reales (V02 con pgvector, V05 y V14).

---

## 2. Instalar las herramientas (Linux)

Cada herramienta se puede instalar de más de una forma. Elige una por herramienta y comprueba la versión al final.

### 2.1 Python 3.12 o superior y Git

```bash
sudo apt update && sudo apt install -y python3 python3-venv python3-pip git unzip curl
python3 --version        # debe ser >= 3.12
```

### 2.2 AWS CLI v2

Es la herramienta con la que te autenticas y compruebas lo que Terraform crea. Usa la versión **2**; la v1 (`pip install awscli`) no es la que documenta AWS.

**Opción A · Instalador oficial en tu carpeta de usuario (sin `sudo`).** Es la que usamos en este proyecto:

```bash
cd "$(mktemp -d)"
curl -fsSL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
unzip -q awscliv2.zip
./aws/install -i ~/.local/aws-cli -b ~/.local/bin
```

Deja el binario en `~/.local/bin/aws`. Esa carpeta no siempre está en el `PATH`, así que añádela:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc     # en bash: ~/.bashrc
source ~/.zshrc
aws --version
```

Si tu CPU es ARM (Raspberry Pi, algunos servidores), cambia `x86_64` por `aarch64` en la URL.

**Opción B · Instalador oficial para todo el sistema (con `sudo`).** Mismo zip, y en lugar de la última línea:

```bash
sudo ./aws/install
```

**Opción C · Snap:**

```bash
sudo snap install aws-cli --classic
```

**Opción D · Homebrew** (macOS, o Linux si tienes Linuxbrew):

```bash
brew install awscli
```

> Al final de `aws configure` el asistente puede preguntar si quieres configurar *skills* y un servidor MCP de AWS para agentes de IA. **Es opcional y el proyecto no lo necesita.** Si aceptas, modifica los archivos de configuración de Claude Code, Codex, Gemini CLI y OpenCode con un servidor global (`aws-mcp`) que usa tus credenciales de AWS. Para quitarlo de Claude Code: `claude mcp remove aws-mcp -s user`.

### 2.3 Terraform (≥ 1.10)

Terraform describe la infraestructura como código. El proyecto exige la **versión 1.10 o superior** (bloqueo de estado nativo en S3, sin DynamoDB).

**Opción A · Snap** (la que usamos; la más corta):

```bash
sudo snap install terraform --classic
terraform version
```

Queda en `/snap/bin/terraform`. Ten en cuenta que ese paquete de snap lo mantiene la comunidad (*snapcrafters*), no HashiCorp. Funciona, pero si prefieres el paquete oficial, usa la opción B.

**Opción B · Repositorio apt oficial de HashiCorp** (Ubuntu/Debian):

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
```

**Opción C · Homebrew:**

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

**Opción D · Binario sin `sudo`.** Descarga el zip de https://developer.hashicorp.com/terraform/install, descomprímelo y mueve `terraform` a `~/.local/bin`.

### 2.4 Docker

Hace falta para construir la imagen de la Lambda y subirla a ECR.

```bash
sudo apt install -y docker.io
sudo usermod -aG docker "$USER"      # cierra sesión y vuelve a entrar para que surta efecto
docker --version
```

En Linux también puedes instalar Docker Engine desde el repositorio oficial de Docker, o Docker Desktop.

### 2.5 Comprobar todo

```bash
python3 --version && git --version && aws --version && terraform version && docker --version
```

---

## 3. Ejecutar en local (modo mock)

### Setup
```bash
python3 -m venv .venv && .venv/bin/pip install -e ".[dev]"
cp .env.example .env            # por defecto: todo en modo mock, sin red
```

Si prefieres no escribir `.venv/bin/` cada vez, activa el entorno: `source .venv/bin/activate`. Cuando veas `(.venv)` en el prompt ya puedes usar `python` y `pip` directamente. **Sin activarlo, `python` puede no existir** en tu shell.

### Seed (datos sintéticos)
```bash
.venv/bin/python scripts/generate_dataset.py && .venv/bin/python scripts/generate_eval_cases.py

DATABASE_URL=... .venv/bin/python scripts/apply_migrations.py --seed   # solo STORE_MODE=supabase

.venv/bin/python scripts/ingest.py                      # embeddings -> store configurado
```

### Ejecutar
```bash
.venv/bin/uvicorn app.api.main:app --reload

JWT_SECRET=... .venv/bin/python scripts/mint_token.py --role technician --plants PLANT-A --user ana@example.com
```

> `.env` sirve **solo en local** y git lo ignora. En AWS la app lee sus secretos de Secrets Manager (`SECRETS_BACKEND=aws`) y nunca ve tu `.env`.

---

## 4. Preparar Supabase

1. **Crea el proyecto** en https://supabase.com. Guarda la contraseña de la base de datos y el **project ref** (el `<ref>` de `https://<ref>.supabase.co`; no es secreto).
2. **Cadena de conexión:** *Project Settings → Database → Connection string*.
   - Migraciones y scripts: *session pooler* o conexión directa, puerto **5432**.
   - Lambda: *transaction pooler*, puerto **6543**.
3. **Aplica migraciones y datos** (puerto 5432):
   ```bash
   read -s "DATABASE_URL?database url (5432): "; export DATABASE_URL
   .venv/bin/python scripts/apply_migrations.py --seed
   ```
4. **Ingiere los embeddings.** `EMBEDDING_MODE` debe coincidir con el `embedding_mode` que pondrás en Terraform; si no, las búsquedas no encontrarán resultados coherentes:
   ```bash
   STORE_MODE=supabase EMBEDDING_MODE=mock .venv/bin/python scripts/ingest.py
   ```
5. **Crea los 3 usuarios demo.** Necesita tres valores:
   - `SUPABASE_URL`: `https://<ref>.supabase.co` (no es secreto).
   - `SUPABASE_SERVICE_ROLE_KEY`: en *Project Settings → API Keys*. Con el formato nuevo está en *Secret keys* (`sb_secret_...`); con el legacy, en *Legacy API Keys* como `service_role`. Es una credencial raíz que se salta la seguridad: no la pegues en chats, no la subas a git ni a CI.
   - `DEMO_PASSWORD`: **la inventas tú**. Es la contraseña común de los usuarios demo. Guárdala en tu gestor de contraseñas.
   ```bash
   read -s "SUPABASE_SERVICE_ROLE_KEY?service role key: "; export SUPABASE_SERVICE_ROLE_KEY
   read -s "DEMO_PASSWORD?demo password: "; export DEMO_PASSWORD
   export SUPABASE_URL=https://<ref>.supabase.co
   .venv/bin/python scripts/create_demo_users.py
   ```
6. **Datos de JWT** para Terraform: `jwt_issuer = https://<ref>.supabase.co/auth/v1`. Si tu proyecto usa llaves asimétricas, añade también `jwt_jwks_url = https://<ref>.supabase.co/auth/v1/.well-known/jwks.json`.

### Cómo cargar un secreto sin dejarlo en el historial

```zsh
read -s "NOMBRE?mensaje: "; export NOMBRE        # zsh
read -s -p "mensaje: " NOMBRE; export NOMBRE      # bash
```

- Lo que va después del `?` en zsh es **el texto del prompt, no el valor**. No escribas ahí la clave.
- Comprueba que se cargó sin mostrarla: `echo ${#NOMBRE}` debe dar un número mayor que 0.
- La variable vive solo en esa terminal. Ejecuta el script en la misma.
- Si una clave se te filtra, rótala desde el dashboard del servicio.

---

## 5. Provisionar AWS con Terraform

### Qué crea

| Servicio | Para qué |
|---|---|
| S3 | Estado de Terraform (bootstrap) |
| IAM y OIDC de GitHub | 4 roles para CI: plan, apply, build y deploy |
| ECR | Imagen del contenedor de la Lambda |
| Lambda (API y worker) | La aplicación, desplegada por digest de imagen |
| API Gateway REST | Entrada pública con usage plan y API key |
| SQS (acciones y DLQ) | Cola de acciones |
| Secrets Manager | Credenciales de la app |
| CloudWatch y SNS | Logs, 4 alarmas, avisos por email |
| EC2 `t3.small` | Opcional (`enable_ec2`). Es lo que más cuesta |

### ¿Por qué Terraform y no la CLI?

Terraform es repetible, muestra un `plan` antes de cambiar nada, resuelve el orden de dependencias entre recursos y `terraform destroy` lo borra todo sin dejar huérfanos. Hacer esto a mano con la CLI son cientos de comandos y es fácil olvidar un recurso y seguir pagando. La CLI se usa para **comprobar** lo creado y para tareas puntuales, no para crear la infraestructura.

### ¿Por qué hace falta un bucket S3?

Terraform guarda en un archivo de **estado** qué recursos creó y su ID real. El CI (GitHub Actions) también ejecuta `plan` y `apply` desde sus propias máquinas, así que el estado tiene que estar en un sitio compartido. El bucket tiene versionado, cifrado y acceso público bloqueado. El bucket se crea en el *bootstrap*, con estado local, y a partir de ahí el entorno `workshop` guarda ahí el suyo.

### 5.1 Credenciales de AWS

```bash
aws configure
```

Pide cuatro cosas:
- **AWS Access Key ID** y **Secret Access Key:** se crean en la consola, en *IAM → Users → tu usuario → Security credentials → Create access key*.
- **Default region name:** `us-east-1`.
- **Default output format:** `json`.

Si tu cuenta usa IAM Identity Center, ejecuta `aws configure sso` en su lugar. Comprueba que entras a la cuenta correcta:

```bash
aws sts get-caller-identity
```

**Permisos.** El bootstrap crea un proveedor OIDC, roles y políticas, es decir, IAM. Un usuario sin permisos de IAM verá `AccessDenied` en `iam:ListOpenIDConnectProviders`. Para un taller en una cuenta propia, asigna `AdministratorAccess` al usuario desde la consola (*IAM → Users → tu usuario → Add permissions*) y **quítalo al terminar el bootstrap**. Si la cuenta no es tuya, pide esos permisos al administrador.

### 5.2 Bootstrap (una sola vez) ⚠️

```bash
cd infrastructure/terraform/bootstrap
cp terraform.tfvars.example terraform.tfvars
```

Edita `terraform.tfvars` y pon **tu** usuario y repo de GitHub (`github_repo = "<usuario>/<repo>"`). Esa variable solo existe en el bootstrap, no en el entorno `workshop`.

Comprueba si la cuenta ya tiene el proveedor OIDC de GitHub:

```bash
aws iam list-open-id-connect-providers
```

Si ya aparece uno con `token.actions.githubusercontent.com`, descomenta `create_github_oidc_provider = false` en el tfvars. Si la lista está vacía, déjalo comentado. Después:

```bash
terraform init
terraform plan          # revísalo
terraform apply         # escribe "yes" solo si el plan es el esperado
terraform output        # anota state_bucket y account_id
```

Crea 17 recursos: el bucket de estado, el proveedor OIDC y los 4 roles con sus políticas.

### 5.3 Variables y entornos en GitHub

En *Settings → Secrets and variables → Actions → Variables* añade: `AWS_REGION`, `AWS_ACCOUNT_ID`, `TF_STATE_BUCKET`, `JWT_ISSUER` y, si aplica, `JWT_JWKS_URL`.

En *Settings → Environments* crea `workshop-infra` y `workshop-deploy` y activa **Required reviewers** en ambos. Es el control humano del repositorio.

### 5.4 Entorno `workshop`

Ten claro **en qué carpeta estás**: los comandos de Terraform actúan sobre el directorio actual, y rutas como `../environments/workshop` solo valen desde `bootstrap`.

```bash
cd infrastructure/terraform/environments/workshop
cp backend.hcl.example backend.hcl
cp terraform.tfvars.example terraform.tfvars
```

En `backend.hcl` pon `bucket = "maintenance-ai-tfstate-<ACCOUNT_ID>"`. Después:

```bash
terraform init -backend-config=backend.hcl
```

En `terraform.tfvars` ajusta estos valores:

| Variable | Valor |
|---|---|
| `jwt_issuer` | `https://<ref>.supabase.co/auth/v1`. **Sin** `/.well-known/jwks.json` al final. Esa ruta va en `jwt_jwks_url`. |
| `llm_mode` y `embedding_mode` | `"mock"`, o `"openai"` si ya tienes la llave. `embedding_mode` debe coincidir con tu ingesta. |
| `enable_ec2` | `false` para evitar el coste de la instancia. |
| `api_reserved_concurrency` y `worker_reserved_concurrency` | `-1` en cuentas nuevas (ver [problema 7.2](#72-lambda-reservedconcurrentexecutions-decreases-accounts-unreservedconcurrentexecution)). |
| `lambda_image_uri` | Se rellena en el paso siguiente. |

### 5.5 Imagen en ECR y primer apply ⚠️

`lambda_image_uri` exige una imagen que ya exista en ECR, pero el repositorio lo crea esta misma pila. Se resuelve en tres pasos:

```bash
# 1) Crear solo el repositorio ECR (desde environments/workshop)
terraform apply -target=module.ecr

# 2) Construir y subir la imagen (desde la RAÍZ del repo, donde está Dockerfile.lambda)
cd ../../../..
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
docker build --provenance=false --sbom=false -f Dockerfile.lambda \
  -t $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/maintenance-ai-workshop:v1 .
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/maintenance-ai-workshop:v1
```

`--provenance=false --sbom=false` es obligatorio: sin él, `docker buildx` sube un índice OCI con una entrada de procedencia que Lambda rechaza ([problema 7.3](#73-lambda-no-acepta-la-imagen)).

El `push` imprime `digest: sha256:...` (64 caracteres hexadecimales). **Cópialo completo**, un carácter menos y Lambda no encontrará la imagen. Ponlo en el tfvars, con **tu** número de cuenta:

```
lambda_image_uri = "<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/maintenance-ai-workshop@sha256:<digest>"
```

```bash
# 3) Plan completo y apply (desde environments/workshop)
cd infrastructure/terraform/environments/workshop
terraform plan
terraform apply
terraform output        # anota api_url, app_secret_id, usage_plan_id
```

Con `enable_ec2 = false`, el paso 1 crea 2 recursos (el repositorio ECR y su política de ciclo de vida) y el apply completo unos 39 más. Con `enable_ec2 = true` se añaden la instancia, su rol y su grupo de seguridad.

---

## 6. Secretos y verificación

### 6.1 Cargar los secretos en Secrets Manager

Terraform solo crea el secreto **vacío**. Los valores los pones tú, sin pasarlos por la línea de comandos. Desde la raíz del repo:

```bash
source .venv/bin/activate
read -s "OPENAI_API_KEY?openai key (opcional): "; export OPENAI_API_KEY
read -s "DATABASE_URL?database url (puerto 6543): "; export DATABASE_URL
read -s "JWT_SECRET?jwt secret: "; export JWT_SECRET
python scripts/seed_app_secret.py --secret-id maintenance-ai/workshop/app --region us-east-1
```

- `DATABASE_URL` va con el *transaction pooler* (6543).
- `JWT_SECRET` es obligatorio para el script. Si tu proyecto usa llaves asimétricas, dale un valor cualquiera y configura `jwt_jwks_url` en Terraform.
- `MCP_SHARED_SECRET` se genera solo.
- `.env` no se mueve a ningún sitio: copia las llaves que necesites a mano con los comandos de arriba.

### 6.2 Variables finales en GitHub

Con los *outputs* añade en GitHub: `API_URL` y el secreto `API_KEY` (la clave del usage plan). Si usas EC2, también `EC2_INSTANCE_ID`. Obtén la clave con:

```bash
aws apigateway get-api-keys --include-values --query 'items[].{name:name,value:value}' --output table
```

No pegues esa clave en chats ni en commits.

### 6.3 Smoke test

```bash
python scripts/smoke_test.py --base-url <API_URL> --api-key <KEY> \
  --secret-id maintenance-ai/workshop/app --route /diagnose/quick
```

Comprueba `/health`, que sin token devuelva 401, un diagnóstico de P-204 con citas y que el acceso entre plantas dé 403.

### 6.4 Despliegues posteriores

El workflow **Deploy** (`workflow_dispatch`, entorno `workshop`) construye la imagen, la escanea, espera a un revisor, la despliega por digest y hace *rollback* si el smoke test falla. Terraform ignora los cambios de `image_uri` (`ignore_changes`): **el CI es el dueño de la versión de la imagen**, y Terraform de la configuración.

---

## 7. Problemas conocidos y su solución

Todos ocurrieron al provisionar este proyecto.

### 7.1 `aws: command not found` o `python: command not found`
- `aws`: `~/.local/bin` no está en el `PATH`. Ver [2.2](#22-aws-cli-v2).
- `python`: no activaste el entorno virtual. Ejecuta `source .venv/bin/activate`.

### 7.2 `Lambda ReservedConcurrentExecutions ... decreases account's UnreservedConcurrentExecution below its minimum value of [10]`
Las cuentas nuevas de AWS tienen un límite de **10 ejecuciones concurrentes en total**, y AWS exige dejar 10 sin reservar, así que no se puede reservar ninguna. Compruébalo con `aws lambda get-account-settings --query AccountLimit`. En `terraform.tfvars`:
```
api_reserved_concurrency    = -1
worker_reserved_concurrency = -1
```
Sin reserva no hay tope de conexiones a Supabase, pero con 10 ejecuciones totales no es un riesgo real. Si pides un aumento de cuota a AWS, vuelve a poner 5 y 2.

### 7.3 Lambda no acepta la imagen
`docker buildx` añade por defecto una entrada de procedencia y deja una **imagen índice** (`application/vnd.oci.image.index.v1+json`). Reconstruye con `--provenance=false --sbom=false`. Comprueba que la etiqueta sea un manifiesto simple:
```bash
aws ecr batch-get-image --repository-name maintenance-ai-workshop --image-ids imageTag=v1 --query 'images[0].imageManifestMediaType' --output text
# debe ser application/vnd.oci.image.manifest.v1+json
```

### 7.4 `ResourceConflictException: Function already exist`
Una Lambda quedó *tainted* porque el apply anterior falló a medias (por ejemplo por el límite de concurrencia). Terraform intenta reemplazarla y AWS aún tiene el nombre ocupado. Si la función existente está sana (`State: Active`), quita la marca en lugar de recrearla:
```bash
terraform untaint module.lambda_api.aws_lambda_function.this
terraform plan        # confirma que ya no la destruye
terraform apply
```
Para el worker, usa `module.lambda_worker.aws_lambda_function.this`.

### 7.5 `Backend initialization required, please run "terraform init"`
Estás en una carpeta donde aún no ejecutaste `init`. El entorno `workshop` necesita el bucket del bootstrap, así que aplica el bootstrap primero y luego `terraform init -backend-config=backend.hcl`.

### 7.6 `cd: no such file or directory: ../environments/workshop`
Las rutas relativas dependen de la carpeta actual. Usa la ruta completa desde la raíz: `cd infrastructure/terraform/environments/workshop`.

### 7.7 `failed to read dockerfile: open Dockerfile.lambda`
`docker build` se ejecutó fuera de la raíz del repo. `Dockerfile.lambda` está en la raíz.

### 7.8 `AccessDenied ... iam:ListOpenIDConnectProviders`
El usuario de AWS no tiene permisos de IAM. Ver [5.1](#51-credenciales-de-aws).

### 7.9 La API rechaza todos los tokens
Revisa que `jwt_issuer` termine en `/auth/v1` y no lleve `/.well-known/jwks.json`. Con un `iss` distinto, ninguna validación coincide.

### 7.10 Un `apply` falló a medias
Un `apply` parcial no deshace lo creado. Ejecuta `terraform plan` para ver qué falta y qué está marcado como *tainted*, y repite el `apply` tras corregir la causa.

---

## 8. Limpiar y dejar de pagar ⚠️

Sin EC2 el coste es mínimo (sobre todo Secrets Manager, unos 0,40 USD al mes), pero al terminar el taller:

```bash
cd infrastructure/terraform/environments/workshop
terraform destroy        # revisa el plan de destrucción
```

Después, si ya no lo necesitas, el bootstrap. Vacía antes el bucket de estado, incluidas todas las versiones, y confirma siempre el plan. Por último, **quita `AdministratorAccess`** al usuario de AWS y borra las llaves de acceso que ya no uses.

---

## 9. Test, calidad y guía por versión

```bash
.venv/bin/ruff check app handlers scripts tests
.venv/bin/python -m pytest                 # offline; los tests de Postgres requieren TEST_DATABASE_URL
.venv/bin/python scripts/run_eval.py       # gates golden/adversarial
```

### Build local
```bash
docker build -f Dockerfile.lambda -t maintenance-ai:lambda .
docker build -f Dockerfile.ec2 -t maintenance-ai:ec2 .
# NUNCA ejecutar `terraform apply` sin confirmación del operador.
```

### Guía por versión
Cada versión V01–V15 tiene su README en [labs/](labs/). Documentación adicional: [docs/](docs/). Guía de AWS y Supabase con más detalle: [manual-aws-supabase.md](manual-aws-supabase.md).

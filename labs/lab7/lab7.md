---
layout: lab
title: "Práctica 7: Crear pipeline que haga plan en pull requests y apply en merges"
permalink: /lab7/lab7/
images_base: /labs/lab7/img
duration: "60 minutos"
objective:
  - "Crear un pipeline CI/CD reproducible que valide y planifique Terraform en Pull Requests, aplique infraestructura tras fusionar a main y mantenga un estado remoto protegido en Google Cloud Storage."
prerequisites:
  - "Prácticas 1 a 6 completadas; carpeta terraform-gcp-intermediate en el Escritorio de Windows."
  - "VS Code con terminal Git Bash, Terraform >= 1.5, gcloud CLI, Git y cuenta GitHub con permiso para crear repositorios."
  - "Proyecto GCP con facturación y permisos para crear Service Accounts, Workload Identity Pools/Providers, administrar IAM, buckets GCS y redes VPC."
  - "Permiso en GitHub para configurar Actions y variables del repositorio; API de Compute Engine habilitada."
introduction:
  - "Usarás GitHub Actions para ejecutar fmt, validate y plan al proponer cambios, y apply al fusionarlos a main. Configurarás autenticación sin llaves JSON mediante Workload Identity Federation (OIDC), guardarás el estado en GCS y retirarás la infraestructura con un segundo Pull Request. La red VPC será un recurso pequeño y fácil de validar."
slug: lab7
lab_number: 7
final_result: >
  Habrás creado un repositorio y un pipeline funcional con autenticación OIDC mediante Workload Identity Federation,
  estado remoto GCS y validación de Pull Requests. Comprobarás un despliegue tras merge y una eliminación controlada
  mediante un segundo Pull Request, sin dejar recursos VPC de la práctica.
notes:
  - "Todos los comandos de sistema se ejecutan en Git Bash integrado en VS Code, dentro de terraform-gcp-intermediate/lab7, salvo que el paso indique GitHub o Google Cloud Console."
  - "La estimación de 60 minutos supone permisos IAM y herramientas disponibles; los tiempos de Google Cloud y GitHub Actions pueden variar."
  - "No generes llaves JSON de Service Account ni concedas Editor; aplica permisos específicos. La identidad de GitHub se limita al repositorio autorizado."
  - "El bucket remoto de estado se crea antes del primer terraform init; se conserva al terminar salvo que el instructor indique una limpieza administrativa adicional."
  - "El workflow solo admite Pull Requests del mismo repositorio para evitar ejecutar infraestructura no confiable con credenciales de GCP."
  - "En los pasos de construcción dirigida escribe exactamente los bloques indicados y guarda antes de continuar."
references:
  - text: "Terraform: Backend GCS"
    url: https://developer.hashicorp.com/terraform/language/backend/gcs
  - text: "Terraform: State Locking"
    url: https://developer.hashicorp.com/terraform/language/state/locking
  - text: "Google Cloud: Workload Identity Federation para pipelines"
    url: https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines
  - text: "Google GitHub Actions: auth"
    url: https://github.com/google-github-actions/auth
  - text: "GitHub Actions: eventos de workflow"
    url: https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows
  - text: "GitHub Actions: variables de configuración"
    url: https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-variables
  - text: "Terraform Google Provider"
    url: https://registry.terraform.io/providers/hashicorp/google/latest/docs
prev: /lab6/lab6/
next: /lab8/lab8/
---

---

## 🔎 Tarea 1. Preparar el repositorio y la estación de trabajo — 7 min

Crearás un repositorio público exclusivo de GitHub para la práctica, comprobarás las herramientas instaladas y prepararás `lab7` dentro del directorio central del curso, sin modificar las prácticas anteriores.

### Tarea 1.1. Verificar herramientas y preparar lab7

Comprobar VS Code, Git Bash y las herramientas de línea de comandos antes de comenzar la configuración de GitHub y Google Cloud.

- {% include step_label.html %} En **VS Code**, abre la carpeta principal del curso antes de comenzar. Sigue esta secuencia en la interfaz:

  1. Selecciona **File > Open Folder...**.
  2. Abre la carpeta **Escritorio > terraform-gcp-intermediate**.
  3. En el menú superior, selecciona **Terminal > New Terminal**.
  4. Si la terminal no es Git Bash, abre el selector de perfil y elige **Git Bash**.
  5. Comprueba que en el explorador aparezcan las carpetas de las prácticas anteriores. **No abras ni modifiques `lab6`.**

  > **Salida esperada:** Debes ver la raíz del curso abierta y una terminal Bash con acceso al proyecto.
  {: .lab-note .output .compact}

- {% include step_label.html %} Comprueba las herramientas desde **Git Bash**: `git` administra versiones, `terraform` define infraestructura y `gcloud` configura IAM y GCS. No necesitas instalar GitHub CLI porque todas las acciones de Pull Request se realizan desde el navegador.

  ```bash
  terraform version
  gcloud version | head -n 3
  git --version
  ```

  > **Salida esperada:** Terraform, gcloud y Git muestran versión; GitHub CLI es opcional.
  {: .lab-note .output .compact}

- {% include step_label.html %} Confirma qué proyecto y qué usuario GCP están activos **antes** de crear permisos o buckets. Conserva el ID en una variable de la terminal para reutilizarlo. Si aparece otro proyecto, corrige `gcloud config set project` antes de continuar.

  ```bash
  PROJECT_ID="$(gcloud config get-value project)"
  PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
  echo "Proyecto: $PROJECT_ID / número: $PROJECT_NUMBER"
  gcloud auth list --filter='status:ACTIVE' --format='value(account)'
  gcloud auth application-default print-access-token >/dev/null && echo "ADC disponible"
  ```

  > **Nota:** Si no aparece `ADC disponible`, ejecuta `gcloud auth application-default login` antes del `terraform plan` local. El inicio de sesión de gcloud y ADC son mecanismos distintos.
  {: .lab-note .info .compact}

  > **Salida esperada:** El ID, número de proyecto y cuenta corresponden al curso, y la terminal confirma `ADC disponible`.
  {: .lab-note .output .compact}


### Tarea 1.2. Crear y clonar un repositorio

Crear el repositorio público del laboratorio, establecer la carpeta `lab7` dentro del directorio del curso y clonar su contenido en esa ubicación.

- {% include step_label.html %} En tu navegador, crea **un repositorio nuevo de GitHub** para alojar exclusivamente la infraestructura de esta práctica. No utilices el repositorio Jekyll donde publicas los laboratorios.

  1. Entra a [github.com](https://github.com/) e inicia sesión.
  2. En la esquina superior derecha, selecciona el icono **+** y después **New repository**. También puedes abrir [New repository](https://github.com/new).
  3. Completa el formulario con estos valores:

  | Campo en GitHub | Valor que debes seleccionar |
  |---|---|
  | **Owner** | Tu cuenta personal de GitHub o la organización autorizada |
  | **Repository name** | `lab7-terraform-gcp` |
  | **Description** | `Pipeline Terraform para GCP - Práctica 7` (opcional) |
  | **Visibility** | **Public** |
  | **Add a README file** | **Activado** |
  | **Add .gitignore** | **None**; lo crearás en VS Code |
  | **Choose a license** | **None** |

  > **Advertencia:** Al ser un repositorio **público**, cualquiera puede consultar los archivos y el historial de Git. Nunca subas credenciales JSON, tokens, contraseñas ni archivos de estado; usa únicamente variables de configuración no secretas y autenticación OIDC.
  {: .lab-note .warning .compact}

  4. Selecciona **Create repository**.
  5. Confirma que aparece `lab7-terraform-gcp`, que el repositorio es **Public** y que contiene `README.md` en la rama `main`.

  > **Nota:** Si el nombre ya existe en la cuenta elegida, usa un sufijo identificador, pero ajusta también `GH_REPO` y la URL de `git clone` en los pasos posteriores.
  {: .lab-note .important .compact}

  > **Salida esperada:** El repositorio público aparece en tu cuenta con la rama principal y un README.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **VS Code**, abre la terminal **Git Bash** y entra a la carpeta principal del curso ubicada en el Escritorio. Ejecuta `mkdir -p lab7` para **crear el directorio de la Práctica 7**, entra con `cd lab7` y confirma la ruta antes de clonar. Así tendrás `terraform-gcp-intermediate/lab7`, sin usar carpetas de las prácticas anteriores.

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab7"
  cd "$DESKTOP/terraform-gcp-intermediate/lab7"
  pwd
  ```

  > **Salida esperada:** La ruta termina en `/terraform-gcp-intermediate/lab7` y la carpeta no contiene archivos de otras prácticas.
  {: .lab-note .output .compact}

- {% include step_label.html %} En el repositorio GitHub, abre **Code > Local > HTTPS** y copia la URL. Regresa a **Git Bash**, ya ubicado en `lab7`, y clona el repositorio **dentro de esta carpeta vacía** mediante `git clone URL .` (el punto final es obligatorio). Sustituye `OWNER` y comprueba que el remoto coincida con tu cuenta antes de continuar.

  ```bash
  git clone "https://github.com/OWNER/lab7-terraform-gcp.git" .
  git status --short
  git remote -v
  ```

  > **Salida esperada:** `lab7` contiene `.git/` y `README.md`; `git status --short` no muestra cambios y `origin` corresponde al repositorio nuevo.
  {: .lab-note .output .compact}

- {% include step_label.html %} Crea el directorio del workflow y los archivos base. Agrega `.gitignore` antes de crear estado o credenciales; Git no debe rastrear `.terraform/`, planes ni archivos locales con valores internos.

  ```bash
  mkdir -p .github/workflows
  touch providers.tf main.tf variables.tf outputs.tf backend.tf .github/workflows/terraform.yml
  cat > .gitignore <<'EOF'
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  *.tfvars
  *.tfvars.json
  *.json.credentials
  .lab7-setup.env
  credentials*.json
  gha-creds-*.json
  EOF
  ```

  > **Salida esperada:** Los archivos HCL y el workflow existen; `.gitignore` excluye estado, planes y credenciales.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **Git Bash**, verifica la identidad del repositorio y establece las variables que usaremos con GitHub. Reemplaza `OWNER` por el propietario real de tu repositorio, respeta mayúsculas y minúsculas y no añadas espacios.

  ```bash
  GH_OWNER="OWNER"
  GH_REPO="lab7-terraform-gcp"
  GH_FULL_REPO="${GH_OWNER}/${GH_REPO}"
  git remote -v
  echo "$GH_FULL_REPO"
  ```

  > **Salida esperada:** La URL de `origin` y `GH_FULL_REPO` señalan exactamente al repositorio creado.
  {: .lab-note .output .compact}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}


---

## ☁️ Tarea 2. Preparar Workload Identity Federation y GCS — 18 min

Habilitarás el acceso de GitHub Actions a Google Cloud sin claves JSON, configurarás permisos limitados y prepararás un bucket GCS para compartir el estado entre ejecuciones.

### Tarea 2.1. Verificar permisos y habilitar APIs

Confirmar que el proyecto permita crear identidades, asignar permisos y acceder a las APIs que utilizará el pipeline.

- {% include step_label.html %} En **Git Bash**, habilita las APIs indispensables. La cuenta del participante debe poder administrar estos servicios; si aparece `PERMISSION_DENIED`, detente y solicita al instructor la preconfiguración del laboratorio.

  ```bash
  gcloud services enable \
    iam.googleapis.com iamcredentials.googleapis.com sts.googleapis.com \
    cloudresourcemanager.googleapis.com compute.googleapis.com \
    storage.googleapis.com --project="$PROJECT_ID"
  ```

  > **Salida esperada:** Las APIs quedan habilitadas; la operación no muestra errores de permisos.
  {: .lab-note .output .compact}

- {% include step_label.html %} Define IDs cortos y repetibles para los recursos de identidad. La Service Account operará Terraform y el Workload Identity Pool aceptará tokens OIDC del repositorio. Ninguno requiere una llave JSON descargable.

  ```bash
  SA_NAME="lab7-terraform"
  SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
  POOL_ID="lab7-github"
  PROVIDER_ID="github-oidc"
  echo "$SA_EMAIL / $POOL_ID / $PROVIDER_ID"
  ```

  > **Salida esperada:** Las variables muestran el email y los identificadores que se usarán en los siguientes comandos.
  {: .lab-note .output .compact}

- {% include step_label.html %} Crea la Service Account y limita sus funciones. El recurso de demostración será únicamente una VPC, por lo que basta el rol de redes para esa tarea; el acceso al bucket de estado se concederá por separado a nivel de bucket.

  ```bash
  gcloud iam service-accounts create "$SA_NAME" \
    --display-name="Lab7 Terraform GitHub Actions" --project="$PROJECT_ID"
  gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:${SA_EMAIL}" --role="roles/compute.networkAdmin"
  ```

  > **Salida esperada:** La cuenta existe y tiene `roles/compute.networkAdmin`; no se concede `roles/editor`.
  {: .lab-note .output .compact}


### Tarea 2.2. Configurar la confianza OIDC de GitHub

Crear el Workload Identity Pool y el OIDC Provider, restringir la confianza al repositorio autorizado y habilitar la suplantación de la Service Account.

- {% include step_label.html %} Crea el **Workload Identity Pool** dentro del proyecto. El pool representa un conjunto de identidades externas autorizadas para solicitar credenciales temporales de Google Cloud. Si ya existe por un intento anterior, confírmalo con `list` antes de reintentar.

  ```bash
  gcloud iam workload-identity-pools create "$POOL_ID" \
  --project="$PROJECT_ID" --location="global" \
  --display-name="Lab7 GitHub Actions"
  ```

  > **Salida esperada:** El comando termina correctamente y el pool aparece habilitado.
  {: .lab-note .output .compact}

- {% include step_label.html %} Crea un **OIDC Provider** para `https://token.actions.githubusercontent.com`. La condición `assertion.repository` limita la confianza al repositorio elegido, evitando admitir tokens de otros repositorios del mismo propietario. Utiliza valores reales para `GH_FULL_REPO`.

  ```bash
  gcloud iam workload-identity-pools providers create-oidc "$PROVIDER_ID" \
  --project="$PROJECT_ID" --location="global" \
  --workload-identity-pool="$POOL_ID" \
  --display-name="GitHub OIDC Lab7" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository == '${GH_FULL_REPO}'"
  ```

  > **Salida esperada:** El provider se crea y solo admite tokens cuyo atributo `repository` coincida con el repositorio de la práctica.
  {: .lab-note .output .compact}

- {% include step_label.html %} Comprueba con `describe` que el OIDC Provider quedó habilitado y contiene la condición de repositorio correcta. Una diferencia en `GH_OWNER` o `GH_REPO` ocasionaría que los tokens de GitHub fueran rechazados; corrige el provider antes de ejecutar el pipeline.

  ```bash
  gcloud iam workload-identity-pools providers describe "$PROVIDER_ID" \
  --project="$PROJECT_ID" --location="global" \
  --workload-identity-pool="$POOL_ID" \
  --format="yaml(name,state,attributeCondition)"
  ```

  > **Salida esperada:** Se confirma `ACTIVE` y la condición coincide exactamente con el propietario y repositorio configurados.
  {: .lab-note .output .compact}

- {% include step_label.html %} Asigna a las identidades permitidas el rol `roles/iam.workloadIdentityUser` **sobre la Service Account**, no sobre todo el proyecto. El principalSet utiliza el número de proyecto y el atributo de repositorio del provider.

  ```bash
  gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/attribute.repository/${GH_FULL_REPO}"
  ```

  > **Salida esperada:** La política IAM de la Service Account contiene el principalSet exacto del repositorio.
  {: .lab-note .output .compact}

- {% include step_label.html %} Obtén el identificador completo del provider. Este valor se configurará como variable de GitHub; evita escribirlo manualmente porque debe incluir número del proyecto, ubicación global, pool y provider.

  ```bash
  WIF_PROVIDER="$(gcloud iam workload-identity-pools providers describe "$PROVIDER_ID" \
    --project="$PROJECT_ID" --location="global" \
    --workload-identity-pool="$POOL_ID" --format='value(name)')"
  echo "$WIF_PROVIDER"
  ```

  > **Salida esperada:** Se muestra una ruta `projects/NUM/locations/global/workloadIdentityPools/.../providers/...`.
  {: .lab-note .output .compact}


### Tarea 2.3. Crear el backend GCS y preparar variables de GitHub

Crear el bucket versionado para estado remoto y configurar las variables necesarias para conectar el workflow con Google Cloud.

- {% include step_label.html %} Crea el bucket de estado **antes** de inicializar Terraform. El nombre GCS es global, por eso añadimos un sufijo aleatorio. Activa el versionado para conservar generaciones anteriores. Guarda el nombre para usarlo en el workflow.

  ```bash
  STATE_BUCKET="tfstate-lab7-${PROJECT_ID}-${RANDOM}"
  gcloud storage buckets create "gs://${STATE_BUCKET}" \
  --project="$PROJECT_ID" --location="us-central1" \
  --uniform-bucket-level-access
  gcloud storage buckets update "gs://${STATE_BUCKET}" --versioning
  echo "$STATE_BUCKET"
  ```

  > **Salida esperada:** Se creó un bucket exclusivo para lab7 y el versionado de objetos está habilitado.
  {: .lab-note .output .compact}

- {% include step_label.html %} Otorga acceso a los objetos del bucket únicamente a la Service Account del pipeline. Para la práctica utilizaremos `roles/storage.objectAdmin` **a nivel del bucket**; nunca concedas acceso administrativo a todos los buckets del proyecto.

  ```bash
  gcloud storage buckets add-iam-policy-binding "gs://${STATE_BUCKET}" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectAdmin"
  gcloud storage buckets describe "gs://${STATE_BUCKET}"   --format='yaml(versioning)'
  ```

  > **Salida esperada:** El bucket tiene versionado activo y la Service Account dispone de acceso a objetos de ese bucket.
  {: .lab-note .output .compact}

- {% include step_label.html %} En GitHub, configura las cuatro **Repository variables** que utilizará el workflow para identificar tu proyecto, la Service Account y el backend GCS.

  1. Abre tu repositorio en el navegador y selecciona **Settings**.
  2. En la barra lateral, entra a **Secrets and variables > Actions**.
  3. Selecciona la pestaña **Variables** (no **Secrets**) y haz clic en **New repository variable**.
  4. Crea una variable por cada fila de esta tabla. Para obtener los valores exactos, ejecuta el comando Bash siguiente:

  | **Name** en GitHub | **Value** que debes copiar |
  |---|---|
  | `GCP_PROJECT_ID` | Contenido de `$PROJECT_ID` |
  | `GCP_WIF_PROVIDER` | Ruta completa mostrada en `$WIF_PROVIDER` |
  | `GCP_SERVICE_ACCOUNT` | Correo de `$SA_EMAIL` |
  | `GCP_STATE_BUCKET` | Nombre de `$STATE_BUCKET`, **sin `gs://`** |

  5. En cada formulario, pega el nombre en **Name**, su valor en **Value** y selecciona **Add variable**.
  6. Regresa a **Actions > Variables** y confirma que aparecen las cuatro entradas. No pegues llaves JSON ni contraseñas.

  ```bash
  printf 'GCP_PROJECT_ID=%s\nGCP_WIF_PROVIDER=%s\nGCP_SERVICE_ACCOUNT=%s\nGCP_STATE_BUCKET=%s\n' \
    "$PROJECT_ID" "$WIF_PROVIDER" "$SA_EMAIL" "$STATE_BUCKET"
  ```

  > **Salida esperada:** Las cuatro variables aparecen registradas en GitHub Actions con nombres y valores exactos.
  {: .lab-note .output .compact}

- {% include step_label.html %} Después de registrar las variables de GitHub, anota el nombre del bucket y la información del provider en un archivo **local ignorado por Git**. Esto evita perder las variables de Git Bash si cierras VS Code y hace reproducibles las validaciones posteriores. El archivo no contiene llaves privadas.

  ```bash
  printf 'PROJECT_ID=%q\nPROJECT_NUMBER=%q\nSTATE_BUCKET=%q\nSA_EMAIL=%q\nWIF_PROVIDER=%q\n' \
    "$PROJECT_ID" "$PROJECT_NUMBER" "$STATE_BUCKET" "$SA_EMAIL" "$WIF_PROVIDER" > .lab7-setup.env
  cat .lab7-setup.env
  ```

  > **Salida esperada:** Existe `.lab7-setup.env` con los datos de configuración; permanece ignorado gracias al `.gitignore`.
  {: .lab-note .output .compact}




{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}


---

## 🧩 Tarea 3. Construir la configuración Terraform — 8 min

Definirás una VPC de prueba mediante Terraform, con variables, outputs y backend GCS. Inicializarás la configuración en Git Bash y revisarás el plan antes de subir los archivos al repositorio.

### Tarea 3.1. Definir Provider, variables y recurso de prueba

Crear la configuración del proveedor Google, declarar las variables requeridas y definir el recurso de red que GitHub Actions administrará.

- {% include step_label.html %} En **VS Code**, abre `lab7/providers.tf` y declara la versión mínima de Terraform y la dependencia del proveedor Google. El proyecto se suministrará mediante una variable, lo que permite reutilizar el mismo código en distintas cuentas del curso.

  ```hcl
  terraform {
    required_version = ">= 1.5.0"
    required_providers {
      google = {
        source  = "hashicorp/google"
        version = "~> 6.0"
      }
    }
  }
  
  provider "google" {
    project = var.project_id
    region  = var.region
  }
  ```

  > **Salida esperada:** El archivo define Google Provider sin incluir credenciales o llaves privadas.
  {: .lab-note .output .compact}

- {% include step_label.html %} En el explorador de **VS Code**, abre `lab7/variables.tf` y crea dos bloques `variable`, uno debajo del otro, **sin copiar código de otro archivo**.

  | Variable | Tipo | Valor predeterminado | Resultado requerido |
  |---|---|---|---|
  | `project_id` | `string` | **No incluir `default`** | Obliga a indicar el proyecto antes de planificar |
  | `region` | `string` | `us-central1` | Define la región del recurso |

  1. Escribe `variable "project_id" { ... }` y declara `type = string`.
  2. Debajo de `project_id`, agrega `region` con el valor de la tabla.
  3. Guarda con **Ctrl+S**. Revisa la sintaxis con la referencia [Input variables](https://developer.hashicorp.com/terraform/language/values/variables).

  > **Nota:** En el paso siguiente crearás `network_name`, que será utilizado por `main.tf`.
  {: .lab-note .important .compact}

  > **Salida esperada:** El archivo declara `project_id` obligatorio y `region` con el valor regional requerido.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de las variables anteriores, agrega `network_name` con el valor por defecto `lab7-cicd-vpc`. La VPC será el único recurso gestionado por el workflow; crear un recurso pequeño mantiene el laboratorio enfocado en CI/CD y simplifica la limpieza.

  ```hcl
  variable "network_name" {
    description = "Nombre de la VPC creada por el pipeline"
    type        = string
    default     = "lab7-cicd-vpc"
  }
  ```

  > **Salida esperada:** La configuración dispone de un nombre de red estable y reconocible.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `lab7/main.tf`, crea la VPC personalizada. Establece `auto_create_subnetworks = false` para no desplegar subredes automáticas en otras regiones. El atributo `description` permitirá comparar el plan con la infraestructura aplicada.

  ```hcl
  resource "google_compute_network" "pipeline" {
    name                    = var.network_name
    auto_create_subnetworks = false
    description             = "VPC administrada por GitHub Actions - lab7"
  }
  ```

  > **Salida esperada:** Se declara una VPC sin subredes automáticas y con descripción identificable.
  {: .lab-note .output .compact}


### Tarea 3.2. Outputs, backend y validación inicial

Definir salidas y backend GCS, inicializar Terraform en Git Bash y comprobar el plan antes de ejecutar GitHub Actions.

- {% include step_label.html %} Crea el output para mostrar el nombre de la VPC creada por Terraform.

  1. En el explorador de **VS Code**, abre `lab7/outputs.tf`.
  2. Declara un bloque **`output "network_name"`**.
  3. Dentro del bloque, asigna a `value` la expresión `google_compute_network.pipeline.name`.
  4. Agrega `description = "Nombre de la VPC administrada en lab7"`.
  5. Guarda con **Ctrl+S** y compara la estructura con [Output values](https://developer.hashicorp.com/terraform/language/values/outputs).

  > **Nota:** El nombre `pipeline` debe coincidir exactamente con el identificador del recurso declarado en `main.tf`.
  {: .lab-note .important .compact}

  > **Salida esperada:** El output muestra el nombre de la VPC que administrará el pipeline.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `lab7/backend.tf`, agrega el backend remoto GCS **sin incrustar el nombre de bucket**. Este valor se inyectará con `terraform init -backend-config` tanto en el entorno local como en GitHub Actions. No uses `var.*` dentro del backend, porque Terraform no acepta variables en esa sección.

  ```hcl
  terraform {
    backend "gcs" {
      prefix = "terraform/state/lab7"
    }
  }
  ```

  > **Salida esperada:** El backend usa el prefijo exclusivo de lab7 y espera el nombre del bucket durante init.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **Git Bash**, ejecuta `fmt` y `validate` después de completar ambos ejercicios dirigidos. Inicializa el backend usando el bucket de estado creado anteriormente y pasa el ID del proyecto a `validate` solo al planificar, no al inicializar.

  ```bash
  terraform fmt -recursive
  terraform init -backend-config="bucket=${STATE_BUCKET}"
  terraform validate
  terraform plan -var="project_id=${PROJECT_ID}"
  ```

  > **Salida esperada:** Terraform inicializa GCS, valida archivos HCL y prepara un plan para crear una VPC.
  {: .lab-note .output .compact}


{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}


---

## ⚙️ Tarea 4. Configurar GitHub Actions — 10 min

Construirás un workflow YAML que ejecutará formato, validación y plan en Pull Requests, y aplicará los cambios únicamente después de fusionarlos a `main`.

### Tarea 4.1. Definir eventos, permisos y contexto de ejecución

Configurar los eventos `pull_request` y `push`, los permisos OIDC del workflow y el job que revisará los cambios propuestos.

- {% include step_label.html %} En **VS Code**, abre `.github/workflows/terraform.yml`. Identifica los eventos que deben activar GitHub Actions, antes de escribir el YAML completo del siguiente paso.

  | Elemento del workflow | Configuración esperada |
  |---|---|
  | `name` | `Terraform lab7 CI/CD` |
  | `on.pull_request.branches` | `main` |
  | `on.push.branches` | `main` |

  1. En el explorador, despliega `.github` > `workflows` y selecciona `terraform.yml`.
  2. Declara el nombre del workflow y los eventos de la tabla.
  3. Verifica que `pull_request` y `push` queden **dentro de `on`**, con sangría YAML de espacios, no tabulaciones.
  4. Guarda con **Ctrl+S**. Contrasta con [Workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).

  > **Nota:** El próximo paso reemplazará este borrador por el encabezado definitivo; aquí debes comprender el disparador de cada evento.
  {: .lab-note .important .compact}

  > **Salida esperada:** El workflow se activará ante PR hacia main y push producido al fusionar cambios.
  {: .lab-note .output .compact}

- {% include step_label.html %} Sustituye el contenido provisional de `.github/workflows/terraform.yml` por el encabezado siguiente. `id-token: write` permite solicitar un token OIDC temporal, `contents: read` permite checkout y `concurrency` impide que dos runs de este workflow operen simultáneamente.

  {%raw%}
  ```yaml
  name: Terraform lab7 CI/CD

  on:
    pull_request:
      branches: [main]
    push:
      branches: [main]

  permissions:
    contents: read
    id-token: write

  concurrency:
    group: lab7-terraform-${{ github.repository }}
    cancel-in-progress: false

  env:
    TF_IN_AUTOMATION: "true"
    TF_INPUT: "false"
    GOOGLE_PROJECT: ${{ vars.GCP_PROJECT_ID }}
    TF_STATE_BUCKET: ${{ vars.GCP_STATE_BUCKET }}
  ```
  {%endraw%}

  > **Importante:** Pega el bloque en **VS Code**, no mediante `cat <<EOF`: Git Bash puede alterar las expresiones `{% raw %}${{ ... }}{% endraw %}`. Si usas terminal, emplea `cat <<'YAML'`.
  {: .lab-note .important .compact}

  > **Salida esperada:** El YAML tiene eventos, permisos mínimos de token y variables comunes definidos.
  {: .lab-note .output .compact}

- {% include step_label.html %} Agrega el job `plan` debajo de `env`. **No** ejecutes autenticación de GCP sobre Pull Requests procedentes de forks: el condicional exige que el repositorio origen y destino sean el mismo. Usa `working-directory: .` porque Terraform vive en la raíz del nuevo repositorio.

  {%raw%}
  ```yaml
  jobs:
    plan:
      if: github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository
      runs-on: ubuntu-latest
      defaults:
        run:
          working-directory: .
      steps:
        - name: Checkout
          uses: actions/checkout@v4
        - name: Validar variables del repositorio
          env:
            WIF_PROVIDER: ${{ vars.GCP_WIF_PROVIDER }}
            SA_EMAIL: ${{ vars.GCP_SERVICE_ACCOUNT }}
          run: |
            test -n "$GOOGLE_PROJECT" && test "$GOOGLE_PROJECT" != '$' || { echo "::error::GCP_PROJECT_ID inválido"; exit 1; }
            test -n "$TF_STATE_BUCKET" && test "$TF_STATE_BUCKET" != '$' || { echo "::error::GCP_STATE_BUCKET inválido"; exit 1; }
            [[ "$SA_EMAIL" =~ ^[^[:space:]@]+@[^[:space:]@]+\.iam\.gserviceaccount\.com$ ]] || { echo "::error::GCP_SERVICE_ACCOUNT inválido"; exit 1; }
            [[ "$WIF_PROVIDER" =~ ^projects/[0-9]+/locations/global/workloadIdentityPools/[^/]+/providers/[^/]+$ ]] || { echo "::error::Falta GCP_WIF_PROVIDER o tiene formato inválido"; exit 1; }
        - name: Google Cloud OIDC
          uses: google-github-actions/auth@v2
          with:
            project_id: ${{ vars.GCP_PROJECT_ID }}
            workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
            service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}
        - name: Terraform CLI
          uses: hashicorp/setup-terraform@v3
          with:
            terraform_version: "1.11.4"
        - name: Inicializar backend GCS
          run: terraform init -input=false -backend-config="bucket=${TF_STATE_BUCKET}"
        - name: Validar formato
          run: terraform fmt -check -recursive
        - name: Validar configuración
          run: terraform validate
        - name: Terraform Plan
          run: terraform plan -input=false -no-color -lock-timeout=60s -var="project_id=${GOOGLE_PROJECT}"
  ```
  {%endraw%}

  > **Validación previa:** Los cuatro valores provienen de **Repository variables** con nombres `GCP_PROJECT_ID`, `GCP_STATE_BUCKET`, `GCP_WIF_PROVIDER` y `GCP_SERVICE_ACCOUNT`. No reemplaces `vars` por `secrets` si configuraste las variables como indica la Tarea 2.3.
  {: .lab-note .important .compact}

  > **Salida esperada:** El job Plan solo se ejecutará para PR originados dentro del mismo repositorio.
  {: .lab-note .output .compact}


### Tarea 4.2. Implementar el Apply del merge

Agregar el job `apply` condicionado a `main`, comprobar la indentación YAML y verificar que GitHub Actions esté habilitado.

- {% include step_label.html %} Revisa en `.github/workflows/terraform.yml` cómo debe declararse el job `apply` debajo del job `plan`.

  1. Identifica la clave principal `jobs:` y el bloque existente `plan:`.
  2. Planea un segundo job llamado `apply:` **al mismo nivel de indentación que `plan:`**.
  3. Configura la condición `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`.
  4. Incluye en tu diseño estas fases: Checkout, autenticación WIF, Terraform Setup, `init`, `validate`, `plan` y `apply`.
  5. Revisa [GitHub Actions expressions](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions). En el paso siguiente insertarás el código funcional completo.

  > **Nota:** La condición evita que una revisión de Pull Request ejecute el despliegue; el `apply` solo corresponde a un `push` en `main`.
  {: .lab-note .important .compact}

  > **Salida esperada:** Apply se limita a push a main y no se ejecuta desde Pull Requests.
  {: .lab-note .output .compact}

- {% include step_label.html %} Añade el siguiente bloque YAML como **segundo job**, al mismo nivel que `plan`. Conserva exactamente la indentación de dos espacios. El `plan` se vuelve a calcular en la revisión fusionada y el `apply` ejecuta únicamente la configuración de `main`.

  {%raw%}
  ```yaml
  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: .
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Validar variables del repositorio
        env:
          WIF_PROVIDER: ${{ vars.GCP_WIF_PROVIDER }}
          SA_EMAIL: ${{ vars.GCP_SERVICE_ACCOUNT }}
        run: |
          test -n "$GOOGLE_PROJECT" && test "$GOOGLE_PROJECT" != '$' || { echo "::error::GCP_PROJECT_ID inválido"; exit 1; }
          test -n "$TF_STATE_BUCKET" && test "$TF_STATE_BUCKET" != '$' || { echo "::error::GCP_STATE_BUCKET inválido"; exit 1; }
          [[ "$SA_EMAIL" =~ ^[^[:space:]@]+@[^[:space:]@]+\.iam\.gserviceaccount\.com$ ]] || { echo "::error::GCP_SERVICE_ACCOUNT inválido"; exit 1; }
          [[ "$WIF_PROVIDER" =~ ^projects/[0-9]+/locations/global/workloadIdentityPools/[^/]+/providers/[^/]+$ ]] || { echo "::error::Falta GCP_WIF_PROVIDER o tiene formato inválido"; exit 1; }
      - name: Google Cloud OIDC
        uses: google-github-actions/auth@v2
        with:
          project_id: ${{ vars.GCP_PROJECT_ID }}
          workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
          service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}
      - name: Terraform CLI
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.11.4"
      - name: Inicializar backend GCS
        run: terraform init -input=false -backend-config="bucket=${TF_STATE_BUCKET}"
      - name: Validar formato
        run: terraform fmt -check -recursive
      - name: Validar configuración
        run: terraform validate
      - name: Revisar plan de main
        run: terraform plan -input=false -no-color -lock-timeout=60s -var="project_id=${GOOGLE_PROJECT}"
      - name: Aplicar infraestructura
        run: terraform apply -input=false -auto-approve -lock-timeout=60s -var="project_id=${GOOGLE_PROJECT}"
  ```
  {%endraw%}

  > **Salida esperada:** El archivo contiene dos jobs separados y la fase Apply no usa claves de servicio JSON.
  {: .lab-note .output .compact}

- {% include step_label.html %} En Git Bash, revisa que el workflow no tenga tabulaciones, bloques duplicados ni errores obvios de indentación. Comprueba en **Settings > Secrets and variables > Actions > Variables** que `GCP_PROJECT_ID`, `GCP_WIF_PROVIDER`, `GCP_SERVICE_ACCOUNT` y `GCP_STATE_BUCKET` tienen valores. No son Secrets en esta práctica. Los pasos **Validar variables del repositorio** detendrán la ejecución con el nombre del valor faltante.

  ```bash
  git diff --check
  git status --short
  sed -n '1,45p' .github/workflows/terraform.yml
  grep -nE '\$\{\{ (vars|github)\.' .github/workflows/terraform.yml
  ```

  > **Salida esperada:** El YAML contiene un encabezado, un único `jobs:` y los jobs `plan` y `apply` correctamente indentados.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **GitHub**, revisa que el repositorio público tenga habilitado **GitHub Actions** antes de enviar el primer Pull Request.

  1. Abre `lab7-terraform-gcp` en el navegador y selecciona **Settings**.
  2. En la barra lateral, entra a **Actions > General**.
  3. En **Actions permissions**, comprueba que estén permitidas las acciones utilizadas: `actions/checkout`, `google-github-actions/auth` y `hashicorp/setup-terraform`.
  4. Si la organización aplica una lista de acciones aprobadas, pide autorización al administrador; **no modifiques políticas de seguridad fuera de tus permisos**.
  5. Verifica que el repositorio tenga visible la pestaña **Actions** y que no existan restricciones que impidan ejecutar workflows desde ramas de tu mismo repositorio.

  > **Nota:** No habilites ejecuciones de código de forks no confiables para solucionar fallos de autenticación.
  {: .lab-note .important .compact}

  > **Salida esperada:** El repositorio permite la ejecución del workflow y las acciones aprobadas.
  {: .lab-note .output .compact}


{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}


---

## 🚀 Tarea 5. Ejecutar Pull Request y merge controlado — 12 min

Publicarás cambios desde una rama, revisarás los checks de GitHub Actions y fusionarás el Pull Request únicamente si el plan es correcto. Después comprobarás que la VPC existe en GCP.

### Tarea 5.1. Preparar la rama de cambio

Crear la rama `feature/lab7-vpc`, revisar que no se suban archivos sensibles y publicar el primer Pull Request.

- {% include step_label.html %} En **Git Bash**, verifica que la rama principal está actualizada y crea `feature/lab7-vpc`. Las modificaciones de infraestructura se proponen en ramas separadas; el pipeline revisa el cambio antes de desplegarlo.

  ```bash
  git switch main
  git pull --ff-only origin main
  git switch -c feature/lab7-vpc
  ```

  > **Salida esperada:** La terminal indica que estás en `feature/lab7-vpc`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Confirma que únicamente subirás HCL y YAML. En particular, revisa que no se incluyan `.terraform/`, archivos `.tfstate`, planes ni credenciales. Luego crea un commit descriptivo.

  ```bash
  git status --short
  git add .gitignore providers.tf main.tf variables.tf outputs.tf backend.tf .terraform.lock.hcl .github/workflows/terraform.yml
  git diff --cached --stat
  git commit -m "feat(lab7): Terraform CI/CD con OIDC y GCS"
  ```

  > **Salida esperada:** El commit incluye infraestructura y workflow, sin archivos de credenciales.
  {: .lab-note .output .compact}

- {% include step_label.html %} Publica la rama de trabajo en GitHub. Al terminar, confirma que la URL remota coincide con tu repositorio y que el push no creó accidentalmente una rama en otro proyecto.

  ```bash
  git push -u origin feature/lab7-vpc
  ```

  > **Salida esperada:** GitHub recibe la rama y muestra la opción de abrir un Pull Request.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **GitHub**, abre un Pull Request para solicitar la incorporación del código Terraform a la rama `main`.

  1. Entra al repositorio y selecciona la pestaña **Pull requests**.
  2. Haz clic en **New pull request** (o en **Compare & pull request** si GitHub lo muestra).
  3. Comprueba los selectores antes de continuar:

  | Selector | Valor esperado |
  |---|---|
  | **base** (rama destino) | `main` |
  | **compare** (rama origen) | `feature/lab7-vpc` |

  4. Comprueba que los archivos cambiados sean los de `lab7` y el workflow.
  5. Escribe un título como `feat: pipeline Terraform GCP para lab7`.
  6. En la descripción, explica que **Plan valida el PR** y que **Apply desplegará tras el merge**.
  7. Selecciona **Create pull request**. **Todavía no fusiones las ramas.**

  > **Salida esperada:** Se crea un PR hacia main y empieza la ejecución del job Plan.
  {: .lab-note .output .compact}


### Tarea 5.2. Revisar el plan antes del merge

Examinar el resultado de formato, validación y plan en la interfaz de GitHub antes de aprobar los cambios.

- {% include step_label.html %} En el Pull Request abierto, inspecciona la ejecución automática de **Plan** antes de aprobar el cambio.

  1. Abre la pestaña **Checks** del PR. Si aún no aparece, entra a **Actions** y selecciona el run correspondiente al evento `pull_request`.
  2. Identifica el workflow **Terraform lab7 CI/CD** y abre el job **plan**.
  3. Revisa estas fases en orden:

  | Fase | Qué debes comprobar |
  |---|---|
  | Checkout | El repositorio se descargó correctamente |
  | Google Cloud OIDC | La autenticación WIF fue aceptada |
  | Terraform CLI | Terraform está disponible en el runner |
  | Inicializar backend GCS | El bucket y el estado remoto son accesibles |
  | Validar formato / configuración | `fmt` y `validate` terminan correctamente |
  | Terraform Plan | El plan termina sin errores y muestra los cambios previstos |

  4. Si aparece un símbolo rojo, abre **la primera fase fallida** y lee el detalle; no hagas merge hasta corregirla.
  5. Verifica que el check final esté **verde (Success)**.

  > **Salida esperada:** El PR muestra un check Plan correcto y un plan que contempla crear solo la VPC de lab7.
  {: .lab-note .output .compact}

- {% include step_label.html %} Revisa la salida del paso **Terraform Plan** para confirmar que GitHub Actions administrará solamente el recurso previsto.

  1. En el log del job `plan`, desplázate hasta la línea `Plan:`.
  2. Confirma que el recurso mencionado sea `google_compute_network.pipeline`.
  3. Para el primer despliegue con estado vacío, la salida debe indicar **`1 to add, 0 to change, 0 to destroy`**.
  4. Si aparecen recursos ajenos, reemplazos inesperados o destrucciones, **detén el proceso y revisa backend, bucket y prefijo**.
  5. Regresa al PR solo cuando la configuración y el resultado coincidan.

  > **Salida esperada:** El plan revisado solo afecta `lab7-cicd-vpc` y no toca recursos externos.
  {: .lab-note .output .compact}

- {% include step_label.html %} Deja una explicación técnica breve como comentario en el Pull Request, sin modificar el código.

  1. Vuelve a la pestaña **Conversation** del PR.
  2. En el cuadro de comentario, explica **por qué `terraform plan` ocurre durante `pull_request`**.
  3. Explica **por qué `terraform apply` solo ocurre después del `push` a `main`**.
  4. Indica que el archivo de estado persiste en el **backend GCS**, no dentro de la máquina temporal de GitHub Actions.
  5. Selecciona **Comment** y confirma que el comentario se publicó.

  > **Nota:** Puedes apoyarte en [Workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).
  {: .lab-note .important .compact}

  > **Salida esperada:** El comentario distingue revisión del plan, despliegue tras merge y persistencia en GCS.
  {: .lab-note .output .compact}


### Tarea 5.3. Fusionar y validar el despliegue

Fusionar el Pull Request, inspeccionar el job `apply` y comprobar la existencia de la VPC con gcloud.

- {% include step_label.html %} Cuando **Plan esté en verde**, fusiona el Pull Request con la rama `main`.

  1. Regresa a la pestaña **Conversation** del PR.
  2. Confirma que el check **plan** está correcto y que GitHub indique que la rama puede fusionarse.
  3. Selecciona **Merge pull request**.
  4. Comprueba que la rama destino sea **`main`** y pulsa **Confirm merge**.
  5. Si hay reglas de revisión obligatoria, solicita la aprobación correspondiente. **No eludas la protección del repositorio.**
  6. Comprueba que el PR muestra **Merged**; la fusión provoca un nuevo `push` a `main` y activa otro workflow.

  > **Salida esperada:** El PR figura como Merged y se inicia la ejecución Apply en Actions.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **GitHub > Actions**, identifica la ejecución de **Apply** creada después del merge.

  1. Entra a la pestaña **Actions** del repositorio.
  2. Selecciona **Terraform lab7 CI/CD** y localiza el run cuya rama es `main` y cuyo evento es **push**.
  3. Abre el job **apply**; su nombre debe ser distinto del job `plan` usado en el PR.
  4. Verifica que terminen correctamente OIDC, `terraform init`, `fmt`, `validate`, `plan` y `apply`.
  5. Busca una confirmación equivalente a **`Apply complete! Resources: 1 added, 0 changed, 0 destroyed.`**.
  6. No continúes con la limpieza mientras el job siga pendiente o haya fallado.

  > **Nota:** El job en `main` calcula un plan nuevo antes de aplicar; no reutiliza literalmente el plan anterior del PR.
  {: .lab-note .important .compact}

  > **Salida esperada:** El run Apply termina en verde y el estado se actualiza en GCS.
  {: .lab-note .output .compact}

- {% include step_label.html %} Desde **Git Bash** de la VM Windows, comprueba la red sin modificarla. Usa el Project ID asignado para evitar mezclar resultados con VPC de laboratorios previos.

  ```bash
  gcloud compute networks describe lab7-cicd-vpc \
    --project="$PROJECT_ID" \
    --format="table(name,autoCreateSubnetworks,description)"
  gcloud storage ls "gs://${STATE_BUCKET}/terraform/state/lab7/"
  ```

  > **Salida esperada:** Aparece `lab7-cicd-vpc` y al menos un objeto de estado dentro del prefijo de lab7.
  {: .lab-note .output .compact}


{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}


---

## 🧹 Tarea 6. Limpiar mediante el mismo pipeline — 5 min

Retirarás la infraestructura mediante un segundo Pull Request, confirmarás que el plan propone destruir solamente la VPC y conservarás el backend remoto para auditoría.

### Tarea 6.1. Retirar la VPC mediante un nuevo Pull Request

Preparar una modificación que elimine solo el recurso de prueba y solicitar su aplicación mediante el mismo pipeline.

- {% include step_label.html %} Actualiza tu clon local a `main` y crea una rama de limpieza. El recurso debe retirarse mediante Terraform: borrar la VPC con `gcloud` provocaría una divergencia entre la infraestructura real y el estado remoto.

  ```bash
  git switch main
  git pull --ff-only origin main
  git switch -c cleanup/lab7-vpc
  ```

  > **Salida esperada:** La nueva rama contiene el workflow y el estado remoto continúa intacto.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **VS Code**, prepara la limpieza **sin borrar archivos de configuración ni destruir recursos manualmente**.

  1. Abre `lab7/main.tf` y localiza el bloque `resource "google_compute_network" "pipeline"`.
  2. Elimina **solo ese bloque completo**, incluidas sus llaves de apertura y cierre.
  3. Abre `lab7/outputs.tf` y elimina el bloque `output "network_name"`, porque hace referencia al recurso retirado.
  4. Conserva `providers.tf`, `variables.tf`, `backend.tf`, `.github/workflows/terraform.yml` y `.gitignore` sin cambios.
  5. Guarda ambos archivos con **Ctrl+S**. Puedes dejarlos vacíos: Terraform seguirá leyendo el provider y el backend remoto.

  > **Advertencia:** No elimines el bucket GCS ni ejecutes `gcloud compute networks delete`; la eliminación debe registrarse en el estado de Terraform.
  {: .lab-note .warning .compact}

  > **Salida esperada:** Los archivos `main.tf` y `outputs.tf` quedan vacíos o contienen solo comentarios; Terraform conserva la configuración del provider y backend.
  {: .lab-note .output .compact}

- {% include step_label.html %} Ejecuta validación, crea el commit y publica la rama. Abre un nuevo PR hacia main, verifica que su plan indique **1 to destroy** para `google_compute_network.pipeline`, y solo entonces fusiona. Si aparece cualquier recurso ajeno, no hagas merge.

  ```bash
  terraform fmt -recursive
  terraform validate
  git add main.tf outputs.tf
  git commit -m "chore(lab7): retirar VPC de prueba"
  git push -u origin cleanup/lab7-vpc
  ```

  1. En GitHub, entra a **Pull requests > New pull request** y selecciona **base: `main`** y **compare: `cleanup/lab7-vpc`**.
  2. Selecciona **Create pull request**. En **Checks**, abre el job `plan` y comprueba **`0 to add, 0 to change, 1 to destroy`**.
  3. Solo si el recurso es `google_compute_network.pipeline`, pulsa **Merge pull request > Confirm merge**.
  4. En **Actions**, abre el nuevo run de `push` sobre `main` y verifica que `apply` termine correctamente.

  > **Salida esperada:** El segundo PR se fusiona después de mostrar la destrucción exclusiva de la VPC, y el job Apply finaliza correctamente.
  {: .lab-note .output .compact}


### Tarea 6.2. Confirmar ausencia de recursos y preservar el estado

Comprobar la eliminación de la VPC en Google Cloud y preservar el bucket de estado para auditoría de la práctica.

- {% include step_label.html %} Verifica en **GitHub > Actions** que el nuevo `apply` de main finalizó correctamente y comprueba desde Git Bash que la VPC ya no existe. Si deseas conservar una evidencia técnica, revisa que el objeto del estado permanezca en GCS, ahora sin el recurso VPC.

  ```bash
  gcloud compute networks list --project="$PROJECT_ID" \
    --filter="name=lab7-cicd-vpc" --format="value(name)"
  gcloud storage ls "gs://${STATE_BUCKET}/terraform/state/lab7/"
  ```

  > **Salida esperada:** La búsqueda de la VPC no devuelve resultados y el backend de Terraform permanece disponible para auditoría.
  {: .lab-note .output .compact}


{% capture r6 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r6 %}

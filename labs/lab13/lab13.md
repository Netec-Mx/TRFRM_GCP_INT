---
course_id: terrgcpint
lab_id: l013
tracking: true
position: 13
layout: lab
title: "Práctica 13: Crear un wrapper básico con Terragrunt"
permalink: /lab13/lab13/
images_base: /labs/lab13/img
duration: "26 minutos"
objective:
  - "Crear un módulo Terraform reutilizable para Cloud Storage y administrarlo mediante Terragrunt desde tres entornos lógicos, con configuración compartida y estado local independiente."
  - "Planificar, desplegar, verificar y destruir los buckets de dev, test y prod sin afectar recursos de otras prácticas."
prerequisites:
  - "Prácticas 1 a 12 realizadas y directorio terraform-gcp-intermediate disponible en el Escritorio de Windows."
  - "VM Windows con VS Code, Git Bash, Terraform >= 1.6, Google Cloud CLI y Terragrunt instalado o acceso para descargarlo."
  - "Proyecto GCP de capacitación con facturación, Cloud Storage API, permisos para crear y eliminar buckets y credenciales ADC configuradas."
introduction:
  - "Crearás un único módulo Terraform que define un bucket GCS y lo consumirás desde tres unidades Terragrunt, dev, test y prod. root.hcl compartirá project_id y location; cada unidad tendrá su nombre de bucket y backend local independiente. Comprobarás las salidas y el estado por entorno, y eliminarás exclusivamente los tres buckets del laboratorio."
slug: lab13
lab_number: 13
final_result: >
  Habrás implementado un wrapper Terragrunt con configuración raíz reutilizable, tres entornos que consumen un único módulo Terraform y estados aislados; comprobarás los tres buckets en Google Cloud y realizarás una limpieza controlada sin dejar recursos activos del laboratorio.
notes:
  - "Todos los comandos locales se ejecutan desde Git Bash integrado en VS Code; cada cambio de directorio se indica expresamente."
  - "Los entornos dev, test y prod son simulaciones dentro de un mismo proyecto GCP: separar estados no equivale a aislar permisos, redes o facturación."
  - "No se reutiliza el backend GCS de las prácticas anteriores. Cada entorno usa backend local con estado en su directorio live correspondiente; no compartas archivos tfstate."
  - "El nombre de cada bucket es globalmente único; se deriva del Project ID y del entorno. Valida que no exista antes de aplicar."
  - "No se utilizarán credenciales JSON ni se concederán roles amplios; si faltan permisos, solicita apoyo del instructor."
  - "Los 30 minutos suponen herramientas, autenticación, permisos y API preparados; descargas y aprovisionamiento pueden aumentar la duración."
references:
  - text: "Terragrunt: documentación oficial"
    url: https://terragrunt.gruntwork.io/docs/
  - text: "Terragrunt: configuración include y root.hcl"
    url: https://terragrunt.gruntwork.io/docs/features/includes/
  - text: "Terragrunt: funciones integradas"
    url: https://terragrunt.gruntwork.io/docs/reference/built-in-functions/
  - text: "Terragrunt: comandos run"
    url: https://terragrunt.gruntwork.io/docs/reference/cli/commands/run/
  - text: "Terraform: desarrollo de módulos"
    url: https://developer.hashicorp.com/terraform/language/modules/develop
  - text: "Google Provider: storage_bucket"
    url: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/storage_bucket
  - text: "Cloud Storage: nombres de buckets"
    url: https://cloud.google.com/storage/docs/buckets#naming
prev: /lab12/lab12/
---


---

## 🔎 Tarea 1. Preparar Terragrunt y construir el módulo base — 7 min

{% assign tracking_task_id = "t001" %}

Prepararás `lab13` sin modificar laboratorios anteriores, comprobarás acceso a Google Cloud y crearás un módulo Terraform reutilizable que expone el nombre del bucket.

### Tarea 1.1. Crear lab13 y verificar herramientas

Trabajar exclusivamente desde el directorio central del curso, con Git Bash, Terraform, Terragrunt y gcloud disponibles.

- {% include step_label.html id="s001" %} En **VS Code**, abre **File > Open Folder...**, selecciona `terraform-gcp-intermediate` del Escritorio y abre **Terminal > New Terminal** con perfil **Git Bash**.

  > **Salida esperada:** El Explorer muestra `lab1` a `lab12` y la terminal integrada presenta el prompt de Git Bash.
  {: .lab-note .output .compact}

- {% include step_label.html id="s002" %} En **Git Bash**, localiza el Escritorio real de Windows, crea `lab13` y prepara las carpetas del módulo y los entornos.

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab13/modules/bucket"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab13/live/"{dev,test,prod}
  cd "$DESKTOP/terraform-gcp-intermediate/lab13"
  pwd
  ```

  > **Salida esperada:** La ruta termina en `/terraform-gcp-intermediate/lab13` y existen `modules/bucket`, `live/dev`, `live/test` y `live/prod`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s003" %} Comprueba versiones, cuenta y proyecto. Guarda el proyecto activo en una variable de entorno para que `root.hcl` pueda leerlo.

  ```bash
  terraform version
  terragrunt --version
  gcloud version | head -n 3
  ```
  ```bash
  export GOOGLE_CLOUD_PROJECT="$(gcloud config get-value project)"
  printf 'Proyecto: %s\n' "$GOOGLE_CLOUD_PROJECT"
  gcloud auth list --filter='status:ACTIVE' --format='value(account)'
  gcloud auth application-default print-access-token >/dev/null && echo 'ADC disponible' 
  ```

  > **Importante:** Si falta Terragrunt puedes instalarlo con Chocolately `choco install terragrunt` ó consulta [Releases oficiales](https://github.com/gruntwork-io/terragrunt/releases), descarga el ejecutable Windows amd64 aprobado por tu instructor y verifica `terragrunt --version`. Si ADC falla, ejecuta `gcloud auth application-default login`.
  {: .lab-note .important .compact}

  > **Salida esperada:** Terraform, Terragrunt y gcloud responden; aparece el Project ID y `ADC disponible`.
  {: .lab-note .output .compact}

### Tarea 1.2. Construir el módulo Terraform bucket

Separar el contrato del módulo en `providers.tf`, `variables.tf`, `main.tf` y `outputs.tf`, sin incrustar el ID del proyecto ni credenciales.

- {% include step_label.html id="s004" %} En Git Bash, crea los cuatro archivos del módulo y un `.gitignore` en `lab13`.

  ```bash
  touch modules/bucket/providers.tf modules/bucket/variables.tf \
        modules/bucket/main.tf modules/bucket/outputs.tf
  ```
  ```bash
  cat > .gitignore <<'EOF'
  .terragrunt-cache/
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  *.tfvars
  *.tfvars.json
  *.json.credentials
  EOF
  ```
  > **Salida esperada:** Los cuatro archivos están dentro de `modules/bucket`; `.gitignore` excluye caché, estado y datos locales.
  {: .lab-note .output .compact}

- {% include step_label.html id="s005" %} En **VS Code**, abre `lab13/modules/bucket/providers.tf` y declara el proveedor Google con restricciones de versión.

  ```hcl
  terraform {
    required_version = ">= 1.6.0"
    required_providers {
      google = {
        source  = "hashicorp/google"
        version = ">= 5.0, < 8.0"
      }
    }
  }
  ```

  > **Salida esperada:** El módulo declara `hashicorp/google` y no contiene un bloque `provider` con credenciales.
  {: .lab-note .output .compact}

- {% include step_label.html id="s006" %} En `modules/bucket/variables.tf`, define las tres entradas que recibirán los entornos a través de Terragrunt.

  ```hcl
  variable "name" {
    description = "Nombre globalmente unico del bucket"
    type        = string
  }
  
  variable "location" {
    description = "Ubicacion GCP del bucket"
    type        = string
  }
  
  variable "project_id" {
    description = "Proyecto GCP de capacitacion"
    type        = string
  }
  ```

  > **Salida esperada:** El módulo exige `name`, `location` y `project_id`; ninguna variable tiene un identificador de proyecto fijo.
  {: .lab-note .output .compact}

- {% include step_label.html id="s007" %} En `modules/bucket/main.tf`, crea el recurso GCS. Conservarás `force_destroy = false` para no borrar objetos por accidente.

  ```hcl
  resource "google_storage_bucket" "bucket" {
    name                        = var.name
    location                    = var.location
    project                     = var.project_id
    uniform_bucket_level_access = true
    public_access_prevention    = "enforced"
    force_destroy               = false
  
    labels = {
      curso = "terraform"
      lab   = "lab13"
    }
  }
  ```

  > **Salida esperada:** El bucket consume las variables del módulo, tiene UBLA y prevención de acceso público habilitados.
  {: .lab-note .output .compact}

- {% include step_label.html id="s008" %} En `modules/bucket/outputs.tf`, crea un output llamado `bucket_name` cuyo valor sea `google_storage_bucket.bucket.name`.

  > **Importante:** Coloca el bloque `output` como declaración independiente en `outputs.tf`, no dentro de otro bloque. Debe devolver el nombre real del recurso.
  {: .lab-note .important .compact}

  > **Salida esperada:** El módulo expone `bucket_name` con `description` y `value`, según la referencia oficial de [Outputs](https://developer.hashicorp.com/terraform/language/values/outputs).
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## 🧩 Tarea 2. Configurar valores compartidos de Terragrunt — 5 min

{% assign tracking_task_id = "t002" %}

Crearás `root.hcl` para centralizar el proyecto y la ubicación, y prepararás la configuración reutilizable de los tres entornos sin convertir la raíz en una unidad ejecutable.

### Tarea 2.1. Crear root.hcl con entradas compartidas

Definir los valores comunes una sola vez y explicar la diferencia entre las entradas compartidas y las específicas de un entorno.

- {% include step_label.html id="s009" %} En **Git Bash**, desde `lab13`, crea `root.hcl` en la raíz del laboratorio; no lo nombres `terragrunt.hcl`.

  ```bash
  touch root.hcl
  pwd
  ls -l root.hcl
  ```

  > **Salida esperada:** `root.hcl` existe en `lab13` y no hay un `terragrunt.hcl` ejecutable en la raíz.
  {: .lab-note .output .compact}

- {% include step_label.html id="s010" %} En **VS Code**, abre `lab13/root.hcl` y declara las entradas compartidas. Terragrunt tomará el proyecto desde la variable del shell.

  ```hcl
  inputs = {
    project_id = get_env("GOOGLE_CLOUD_PROJECT")
    location   = "us-central1"
  }

  ```

  > **Importante:** El Project ID de Google Cloud debe estar configurado en la sesión Git Bash con `export GOOGLE_CLOUD_PROJECT="$(gcloud config get-value project)"`.
  {: .lab-note .important .compact}

  > **Salida esperada:** `root.hcl` define `project_id` desde `GOOGLE_CLOUD_PROJECT` y `location = "us-central1"`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s011" %} Identifica cuál variable del módulo se comparte (`project_id`, `location`) y cuál debe cambiar por entorno (`name`).

  > **Nota:** Usa [Terragrunt Includes](https://terragrunt.gruntwork.io/docs/features/includes/) para revisar cómo se heredan los valores `inputs`.
  {: .lab-note .info .compact}

  > **Salida esperada:** Puedes explicar por qué `name` no pertenece al archivo común y por qué no se duplicará `project_id` en cada entorno.
  {: .lab-note .output .compact}

### Tarea 2.2. Definir la estrategia de estados independientes

Evitar que varios entornos administren un mismo estado y que este se pierda al limpiar la caché de Terragrunt.

- {% include step_label.html id="s012" %} Crea los tres directorios de trabajo si no existen y comprueba la estructura antes de definir los backends locales.

  ```bash
  mkdir -p live/dev live/test live/prod
  find live -maxdepth 1 -type d | sort
  ```

  > **Salida esperada:** Existen `live/dev`, `live/test` y `live/prod` dentro de lab13.
  {: .lab-note .output .compact}

- {% include step_label.html id="s013" %} Revisa la regla de estado que agregarás a **cada** `terragrunt.hcl`: el backend `local` guardará el estado en el directorio propio de la unidad, no en `.terragrunt-cache`.

  ```hcl
  remote_state {
    backend = "local"
    config = {
      path = "${get_terragrunt_dir()}/terraform.tfstate"
    }
  }
  ```

  > **Importante:** El mismo bloque se incorporará en los tres archivos de entorno; esta repetición evita depender de rutas de caché y mantiene el estado fuera de los módulos descargados.
  {: .lab-note .important .compact}

  > **Salida esperada:** Queda definido el patrón de un archivo `terraform.tfstate` distinto para cada unidad de `live/`.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## ☁️ Tarea 3. Configurar los entornos dev, test y prod — 6 min

{% assign tracking_task_id = "t003" %}

Crearás tres unidades Terragrunt que heredan la configuración de `root.hcl`, apuntan al mismo módulo y proporcionan un nombre de bucket único por entorno.

### Tarea 3.1. Definir dev y test con el módulo compartido

Escribir configuraciones que difieren solo en el nombre del bucket y almacenan su estado local en cada directorio.

- {% include step_label.html id="s014" %} Desde **Git Bash** en `lab13`, crea los archivos de configuración de los tres entornos.

  ```bash
  touch live/dev/terragrunt.hcl live/test/terragrunt.hcl live/prod/terragrunt.hcl
  find live -name 'terragrunt.hcl' | sort
  ```

  > **Salida esperada:** Aparecen los tres archivos `terragrunt.hcl` bajo `live/dev`, `live/test` y `live/prod`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s015" %} En `live/dev/terragrunt.hcl`, agrega la configuración completa de desarrollo. La ruta del módulo parte del archivo `root.hcl`, y el estado se guarda en `live/dev`.

  ```hcl
  include "root" {
    path = find_in_parent_folders("root.hcl")
  }
  
  terraform {
    source = "${get_parent_terragrunt_dir("root")}/modules/bucket"
  }
  
  remote_state {
    backend = "local"
    config = {
      path = "${get_terragrunt_dir()}/terraform.tfstate"
    }
    generate = {
      path      = "backend.tf"
      if_exists = "overwrite_terragrunt"
    }
  }
  
  inputs = {
    name = "lab13-dev-${get_env("GOOGLE_CLOUD_PROJECT")}" 
  }
  ```

  > **Salida esperada:** El entorno dev hereda `project_id` y `location` y configura el bucket `lab13-dev-PROJECT_ID`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s016" %} En `live/test/terragrunt.hcl`, usa el mismo módulo, el mismo include y un estado separado; cambia solo el nombre lógico del bucket.

  ```hcl
  include "root" {
    path = find_in_parent_folders("root.hcl")
  }
  
  terraform {
    source = "${get_parent_terragrunt_dir("root")}/modules/bucket"
  }
  
  remote_state {
    backend = "local"
    config = {
      path = "${get_terragrunt_dir()}/terraform.tfstate"
    }
    generate = {
      path      = "backend.tf"
      if_exists = "overwrite_terragrunt"
    }
  }
  
  inputs = {
    name = "lab13-test-${get_env("GOOGLE_CLOUD_PROJECT")}" 
  }
  ```

  > **Salida esperada:** El entorno test crea un nombre distinto sin duplicar las definiciones Terraform.
  {: .lab-note .output .compact}

- {% include step_label.html id="s017" %} En Git Bash, revisa los nombres de ambos buckets con el Project ID real, sin crear recursos todavía.

  ```bash
  printf 'DEV  lab13-dev-%s\n' "$GOOGLE_CLOUD_PROJECT"
  printf 'TEST lab13-test-%s\n' "$GOOGLE_CLOUD_PROJECT"
  ```

  > **Salida esperada:** Los nombres contienen prefijos diferentes y terminan en el Project ID globalmente único.
  {: .lab-note .output .compact}

### Tarea 3.2. Completar prod y comprobar rutas de ejecución

Definir el tercer entorno y verificar la separación de los estados antes del primer despliegue.

- {% include step_label.html id="s018" %} En `live/prod/terragrunt.hcl`, reproduce el esquema de `dev`, sustituyendo únicamente `lab13-dev-` por `lab13-prod-` en `inputs.name`.

  > **Importante:** El `source` debe seguir apuntando a `get_parent_terragrunt_dir("root")/modules/bucket`, no a un directorio dentro de otro entorno. Consulta las [funciones de Terragrunt](https://terragrunt.gruntwork.io/docs/reference/built-in-functions/).
  {: .lab-note .important .compact}

  > **Salida esperada:** El archivo prod contiene `include "root"`, `terraform.source`, `remote_state` local y `inputs.name = "lab13-prod-${get_env("GOOGLE_CLOUD_PROJECT")}"`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s019" %} Comprueba en **VS Code** la estructura: raíz `root.hcl`, cuatro archivos `.tf` dentro del módulo y tres unidades `terragrunt.hcl`.

  ```bash
  find modules/bucket -maxdepth 1 -name '*.tf' | sort
  find live -name 'terragrunt.hcl' | sort
  ```

  > **Salida esperada:** Se encuentran cuatro archivos HCL Terraform y tres configuraciones de entorno, sin duplicar el módulo.
  {: .lab-note .output .compact}

- {% include step_label.html id="s020" %} Desde Git Bash, verifica que el proyecto activo siga siendo correcto y que ninguno de los tres nombres de bucket esté ocupado en tu proyecto.

  ```bash
  test -n "$GOOGLE_CLOUD_PROJECT" && echo "Proyecto: $GOOGLE_CLOUD_PROJECT"
  for env in dev test prod; do
    name="lab13-${env}-${GOOGLE_CLOUD_PROJECT}"
    if gcloud storage buckets describe "gs://${name}" --format='value(name)' >/dev/null 2>&1; then
      echo "Ya existe: $name - detente y revisa"
    else
      echo "No encontrado: $name"
    fi
  done
  ```

  > **Nota:** Que un bucket no aparezca en tu proyecto no garantiza que el nombre esté disponible globalmente. Si `apply` informa un conflicto, consulta con el instructor antes de modificar nombres.
  {: .lab-note .info .compact}

  > **Salida esperada:** Los tres nombres propuestos están disponibles o se identifica explícitamente algún nombre existente antes del despliegue.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## 🚀 Tarea 4. Desplegar y validar los tres entornos — 8 min

{% assign tracking_task_id = "t004" %}

Inicializarás y planificarás cada unidad Terragrunt antes de aplicar; después comprobarás que cada estado administra un solo bucket y que los tres existen en GCP.

### Tarea 4.1. Inicializar, planificar y desplegar dev

Verificar primero la unidad de desarrollo y leer su plan completo antes de crear recursos.

- {% include step_label.html id="s021" %} En **Git Bash**, entra desde `lab13` a `live/dev` e inicializa el módulo Terraform a través de Terragrunt.

  ```bash
  cd live/dev
  terragrunt init
  ```

  > **Salida esperada:** Terragrunt prepara el módulo y Terraform termina con `Terraform has been successfully initialized!`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s022" %} En `live/dev`, ejecuta `plan` y confirma que aparece **un solo bucket** `lab13-dev-PROJECT_ID`.

  ```bash
  terragrunt plan
  ```

  > **Importante:** Si el plan muestra recursos inesperados o el Project ID no coincide, no continúes con `apply`.
  {: .lab-note .important .compact}

  > **Salida esperada:** El plan propone `1 to add, 0 to change, 0 to destroy` para `google_storage_bucket.bucket`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s023" %} Aplica el plan de `dev` desde la misma carpeta y confirma después de revisar los cambios.

  ```bash
  terragrunt apply
  terragrunt output bucket_name
  ```

  > **Salida esperada:** Se crea el bucket de dev y el output muestra `lab13-dev-PROJECT_ID`.
  {: .lab-note .output .compact}

### Tarea 4.2. Planificar y desplegar test y prod

Repetir el mismo flujo para los otros dos entornos, manteniendo sus estados y nombres separados.

- {% include step_label.html id="s024" %} Regresa a `live/`, entra a `test` y ejecuta `init`, `plan` y `apply` secuencialmente, después de revisar el plan.

  ```bash
  cd ../test
  terragrunt init
  ```
  ```bash
  terragrunt plan
  terragrunt apply
  terragrunt output bucket_name
  ```

  > **Salida esperada:** Se crea `lab13-test-PROJECT_ID`, sin cambiar ni destruir el bucket de dev.
  {: .lab-note .output .compact}

- {% include step_label.html id="s025" %} Desde `live/test`, cambia a `prod` y repite inicialización, plan y aplicación con verificación previa.

  ```bash
  cd ../prod
  terragrunt init
  ```
  ```bash
  terragrunt plan
  terragrunt apply
  terragrunt output bucket_name
  ```

  > **Importante:** No utilices `-auto-approve` al aprender el flujo de Terragrunt; revisa siempre la operación antes de aceptarla.
  {: .lab-note .important .compact}

  > **Salida esperada:** Se crea `lab13-prod-PROJECT_ID`; la configuración de producción es solo una simulación dentro del proyecto del curso.
  {: .lab-note .output .compact}

- {% include step_label.html id="s026" %} En Git Bash, desde `live/prod`, consulta los tres nombres directamente con Cloud Storage sin listar indiscriminadamente todos los recursos de capacitación.

  ```bash
  for env in dev test prod; do
    gcloud storage buckets describe "gs://lab13-${env}-${GOOGLE_CLOUD_PROJECT}" \
      --format='value(name)'
  done
  ```

  > **Salida esperada:** Las consultas confirman tres buckets diferentes correspondientes a dev, test y prod.
  {: .lab-note .output .compact}

### Tarea 4.3. Examinar outputs y estados independientes

Verificar la diferencia entre reutilizar un módulo y compartir un estado, que debe permanecer aislado por entorno.

- {% include step_label.html id="s027" %} Regresa a `live` y muestra los tres archivos de estado. Explica por qué se mantienen separados pese a que el módulo es común.

  ```bash
  cd ..
  find dev test prod -maxdepth 1 -name 'terraform.tfstate' -print
  for env in dev test prod; do
    printf '%s: ' "$env"
    (cd "$env" && terragrunt output -raw bucket_name)
  done
  ```

  > **Importante:** El estado Terraform puede contener información sensible. Verifica su existencia y outputs, pero no lo subas a Git ni lo publiques.
  {: .lab-note .important .compact}

  > **Salida esperada:** Hay un estado en cada carpeta y cada output corresponde al bucket específico de su entorno.
  {: .lab-note .output .compact}

- {% include step_label.html id="s028" %} Comprueba desde las tres carpetas que cada unidad conoce exactamente un recurso Terraform y no tiene cambios pendientes.

  ```bash
  for env in dev test prod; do
    echo "--- $env ---"
    (cd "$env" && terragrunt state list && terragrunt plan -detailed-exitcode)
  done
  ```

  > **Nota:** `terraform plan -detailed-exitcode` devuelve 0 sin cambios, 2 cuando hay cambios y 1 ante errores. Detente si alguno devuelve 1 o 2.
  {: .lab-note .info .compact}

  > **Salida esperada:** Cada estado contiene `google_storage_bucket.bucket` y cada plan informa `No changes`.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

## 🧹 Tarea 5. Eliminar buckets y comprobar la limpieza — 4 min

{% assign tracking_task_id = "t005" %}

Destruirás los tres entornos de forma controlada, revisando individualmente los planes, y comprobarás que los buckets de esta práctica ya no existan.

### Tarea 5.1. Destruir los tres entornos con planes revisados

Retirar cada recurso desde su propio estado para evitar afectar infraestructura administrada por otras prácticas.

- {% include step_label.html id="s029" %} Desde `lab13/live`, revisa los tres planes de destrucción individualmente. Debe aparecer un bucket por entorno.

  ```bash
  for env in dev test prod; do
    echo "--- Destroy plan: $env ---"
    (cd "$env" && terragrunt plan -destroy) || break
  done
  ```

  > **Advertencia:** No utilices `terragrunt --all destroy` para omitir la revisión: la ejecución múltiple puede aprobar automáticamente operaciones. Solo continúa si los tres planes son correctos.
  {: .lab-note .warning .compact}

  > **Salida esperada:** Cada plan indica `0 to add, 0 to change, 1 to destroy` y solo menciona su bucket `lab13-*`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s030" %} Ejecuta `destroy` en cada entorno, uno a la vez, confirmando únicamente la eliminación del bucket que corresponda.

  ```bash
  (cd dev && terragrunt destroy)
  (cd test && terragrunt destroy)
  (cd prod && terragrunt destroy)
  ```

  > **Importante:** Los buckets deben estar vacíos, dado que no cargamos objetos y usamos `force_destroy = false`. No borres buckets ajenos si alguna operación falla.
  {: .lab-note .important .compact}

  > **Salida esperada:** Cada comando finaliza con `Destroy complete! Resources: 1 destroyed.`
  {: .lab-note .output .compact}

### Tarea 5.2. Comprobar estados vacíos y ausencia de buckets

Corroborar en Terraform y Google Cloud que solo quedaron archivos fuente del laboratorio, sin recursos activos.

- {% include step_label.html id="s031" %} En `lab13/live`, consulta los estados de los tres entornos. No debe haber recursos administrados después de la destrucción.

  ```bash
  for env in dev test prod; do
    echo "--- Estado $env ---"
    (cd "$env" && terragrunt state list)
  done
  ```

  > **Salida esperada:** Los tres listados están vacíos y no muestran `google_storage_bucket.bucket`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s032" %} Comprueba específicamente que ninguno de los tres buckets existe y conserva los archivos HCL sin borrar el módulo reutilizable.

  ```bash
  for env in dev test prod; do
    name="lab13-${env}-${GOOGLE_CLOUD_PROJECT}"
    if gcloud storage buckets describe "gs://${name}" >/dev/null 2>&1; then
      echo "REVISAR: $name sigue existiendo"
    else
      echo "Sin bucket: $name"
    fi
  done
  find ../modules/bucket -maxdepth 1 -name '*.tf' -print | sort
  ```

  > **Nota:** Una consulta fallida puede deberse también a permisos o conectividad; si hay dudas, consulta Cloud Storage Console antes de asumir que se eliminó.
  {: .lab-note .info .compact}

  > **Salida esperada:** Los tres buckets de la práctica ya no existen; los cuatro archivos fuente del módulo permanecen disponibles.
  {: .lab-note .output .compact}




{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

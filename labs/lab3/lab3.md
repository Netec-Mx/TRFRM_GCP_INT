---
course_id: terrgcpint
lab_id: l003
tracking: true
position: 3
layout: lab
title: "Práctica 3: Migrar el estado local a Google Cloud Storage (GCS) con locking"
permalink: /lab3/lab3/
images_base: /labs/lab3/img
duration: "45 minutos"
objective:
  - "Crear un estado local con una VPC en GCP, migrarlo al backend GCS y verificar conservación del estado, versionado de objetos y bloqueo de operaciones concurrentes."
prerequisites:
  - "Prácticas 1 y 2 realizadas; carpeta terraform-gcp-intermediate creada en el Escritorio de Windows."
  - "VM Windows con VS Code, Git Bash integrado, Terraform y Google Cloud CLI instalados."
  - "Proyecto GCP con facturación, Compute Engine API y Cloud Storage API habilitadas y permisos para administrar VPC y buckets."
  - "Cuenta de gcloud y credenciales ADC configuradas; permisos de lectura/escritura y eliminación de objetos en el bucket creado."
introduction:
  - "Desplegarás una VPC con estado local, crearás un bucket de Cloud Storage con Object Versioning y migrarás el estado existente mediante terraform init -migrate-state. Comprobarás versiones y locking con dos terminales y limpiarás los recursos sin dejar un backend apuntando a un bucket eliminado."
slug: lab3
lab_number: 3
final_result: >
  Habrás creado una VPC real con estado local; migrado su estado a un bucket GCS versionado; comprobado que Terraform sigue administrando la misma infraestructura sin recrearla; observado versiones y bloqueo del estado; y retirado los recursos de forma controlada, conservando el código del laboratorio.
notes:
  - "Todos los comandos de terminal se ejecutan en Git Bash integrado en VS Code, dentro de terraform-gcp-intermediate/lab3, salvo cuando se indiquen dos terminales."
  - "El backend GCS necesita un bucket ya existente y credenciales ADC válidas. Su configuración no admite referencias var.* ni valores local.*."
  - "Conserva el nombre de bucket en .lab3-bucket.env; nunca subas estados, planes o credenciales a Git."
  - "El archivo terraform.tfstate puede contener información sensible; no lo publiques y elimina sus copias temporales después del laboratorio."
  - "La estimación de 30 minutos supone permisos y APIs preparados; despliegue, cambios de estado y limpieza pueden ampliar el tiempo."
references:
  - text: "Terraform - Backend GCS y soporte de locking"
    url: https://developer.hashicorp.com/terraform/language/backend/gcs
  - text: "Terraform - Migración y configuración de backends"
    url: https://developer.hashicorp.com/terraform/language/backend
  - text: "Terraform - State Locking"
    url: https://developer.hashicorp.com/terraform/language/state/locking
  - text: "Google Cloud - Object Versioning"
    url: https://cloud.google.com/storage/docs/using-object-versioning
  - text: "gcloud storage rm y limpieza de versiones"
    url: https://cloud.google.com/sdk/gcloud/reference/storage/rm
  - text: "Google Provider - google_compute_network"
    url: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_network
prev: /lab2/lab2/
next: /lab4/lab4/
---

---

## 🔎 Tarea 1. Preparar lab3 y generar un estado local real — 8 min

{% assign tracking_task_id = "t001" %}

Prepararás un estado real desde el backend local para demostrar una migración con recursos existentes.

### Tarea 1.1. Crear la carpeta y verificar herramientas

Crear un directorio exclusivo de esta práctica y confirmar que VS Code utiliza Git Bash con Terraform y gcloud disponibles.

- {% include step_label.html id="s001" %} Abre **Terminal > New Terminal**, selecciona **Git Bash** y crea el directorio `lab3` en el Escritorio.

  > **Nota:** La ruta real puede estar redirigida a OneDrive; el comando consulta la ubicación de Windows.
  {: .lab-note .info .compact}

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab3"
  cd "$DESKTOP/terraform-gcp-intermediate/lab3"
  ```

  > **Salida esperada:** El directorio actual termina en `/terraform-gcp-intermediate/lab3`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s002" %} Comprueba las herramientas y el proyecto activo desde **Git Bash**.

  > **Nota:** Si falta ADC, ejecuta `gcloud auth application-default login` y regresa a esta carpeta.
  {: .lab-note .info .compact}

  ```bash
  terraform version
  gcloud auth list --filter="status:ACTIVE" --format="value(account)"
  gcloud config get-value project
  ```

  > **Salida esperada:** Terraform muestra su versión, y gcloud la cuenta y el proyecto correctos.
  {: .lab-note .output .compact}

### Tarea 1.2. Definir el proveedor y una VPC con configuración local

Declarar una sola VPC sencilla para disponer de un recurso real administrado inicialmente con estado local.

- {% include step_label.html id="s003" %} En **Git Bash**, crea los archivos de configuración y exclusiones locales.

  > **Nota:** Deja `backend.tf` sin crear por ahora; así Terraform usará el backend local durante el primer despliegue.
  {: .lab-note .info .compact}

  ```bash
  touch providers.tf main.tf variables.tf terraform.tfvars outputs.tf .gitignore
  ```

  > **Salida esperada:** Aparecen los archivos vacíos en el explorador de VS Code.
  {: .lab-note .output .compact}

- {% include step_label.html id="s004" %} En **VS Code**, pega el proveedor Google y la restricción de versiones en `lab3/providers.tf`.

  > **Nota:** Si el proveedor autorizado en el curso tiene otra versión, ajusta esta restricción antes de ejecutar `init`.
  {: .lab-note .info .compact}

  ```hcl
  terraform {
    required_version = ">= 1.5.0"
    required_providers {
      google = {
        source  = "hashicorp/google"
        version = ">= 5.0, < 8.0"
      }
    }
  }
  
  provider "google" {
    project = var.project_id
    region  = var.region
  }
  ```

  > **Salida esperada:** El archivo declara el provider `hashicorp/google` y utiliza variables del módulo raíz.
  {: .lab-note .output .compact}

- {% include step_label.html id="s005" %} En `lab3/variables.tf`, declara `project_id`, `region` y la variable `network_name` según los requisitos.

  > **Nota:** **Completa `network_name` debajo de `region`**: debe ser de tipo `string`, describir el nombre de la VPC y no tener valor predeterminado. Referencia: https://developer.hashicorp.com/terraform/language/values/variables
  {: .lab-note .info .compact}

  ```hcl
  variable "project_id" {
    description = "ID del proyecto Google Cloud"
    type        = string
  }

  variable "region" {
    description = "Región de GCP"
    type        = string
    default     = "us-central1"
  }

  variable "network_name" {
    description = "Red de GCP"
    type        = string
  }
  ```

  > **Salida esperada:** Quedan declaradas tres variables; `network_name` será proporcionada mediante `terraform.tfvars`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s006" %} En `lab3/main.tf`, define el recurso VPC con red personalizada y una descripción inicial.

  > **Nota:** Conserva el recurso `google_compute_network.vpc`: la descripción se modificará después para probar Object Versioning.
  {: .lab-note .info .compact}

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = var.network_name
    auto_create_subnetworks = false
    description             = "Estado local inicial - laboratorio 3"
  }
  ```

  > **Salida esperada:** La VPC depende de `var.network_name` y no crea subredes automáticas.
  {: .lab-note .output .compact}

- {% include step_label.html id="s007" %} En `lab3/terraform.tfvars`, configura el proyecto real y el nombre único de la red.

  > **Nota:** Sustituye **TU_PROJECT_ID** por el resultado de `gcloud config get-value project` antes de continuar.
  {: .lab-note .info .compact}

  ```hcl
  project_id   = "TU_PROJECT_ID"
  region       = "us-central1"
  network_name = "lab3-vpc-estado-remoto"
  ```

  > **Salida esperada:** El archivo contiene el ID real del proyecto, la región y el nombre de VPC.
  {: .lab-note .output .compact}

### Tarea 1.3. Inicializar, desplegar y comprobar el estado local

Ejecutar por primera vez Terraform sin backend remoto y comprobar que el archivo de estado contiene una VPC creada.

- {% include step_label.html id="s008" %} En `lab3/outputs.tf`, expón el identificador y el nombre de la VPC.

  > **Nota:** Guarda todos los archivos con **Ctrl + S** antes de comenzar el flujo Terraform.
  {: .lab-note .info .compact}

  ```hcl
  output "vpc_id" {
    value = google_compute_network.vpc.id
  }
  
  output "vpc_name" {
    value = google_compute_network.vpc.name
  }
  ```

  > **Salida esperada:** Los outputs referencian el recurso declarado en `main.tf`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s009" %} En `.gitignore`, excluye estados, planes, credenciales y el archivo local con el bucket.

  > **Nota:** No elimines el archivo `.terraform.lock.hcl` del directorio; permite reproducir la selección de proveedor.
  {: .lab-note .info .compact}

  ```gitignore
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  .lab3-bucket.env
  .lab3-vpc-*.txt
  .lab3-state-*.txt
  crash.log
  ```

  > **Salida esperada:** Git no incorporará estados ni el archivo de variables de entorno del bucket.
  {: .lab-note .output .compact}

- {% include step_label.html id="s010" %} En **Git Bash**, inicializa Terraform y verifica el formato y la configuración.

  > **Nota:** No debe existir aún un bloque `backend "gcs"` dentro del directorio.
  {: .lab-note .info .compact}

  > **Salida esperada:** Terraform informa que la configuración es válida y prepara el backend local.
  {: .lab-note .output .compact}

- {% include step_label.html id="s011" %} Revisa el plan y despliega únicamente la VPC del laboratorio.

  > **Nota:** Confirma con `yes` solo si el plan muestra **1 recurso para crear**, sin cambios ajenos al laboratorio.
  {: .lab-note .info .compact}

  > **Salida esperada:** Aparece `Apply complete! Resources: 1 added` y el output con el nombre de VPC.
  {: .lab-note .output .compact}

- {% include step_label.html id="s012" %} Confirma que `terraform.tfstate` existe y contiene la VPC creada.

  > **Nota:** **No copies ni publiques** el contenido completo de `terraform.tfstate`; puede incluir datos internos o sensibles.
  {: .lab-note .info .compact}

  ```bash
  test -f terraform.tfstate && echo "OK: estado local existente"
  terraform state list
  terraform output vpc_name
  ```

  > **Salida esperada:** La lista contiene `google_compute_network.vpc` y el output `lab3-vpc-estado-remoto`.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## ☁️ Tarea 2. Crear un bucket GCS con versionado habilitado — 5 min

{% assign tracking_task_id = "t002" %}

Crearás un bucket exclusivo con acceso uniforme y Object Versioning antes de migrar el archivo tfstate.

### Tarea 2.1. Elegir un nombre persistente y crear el bucket

Preparar un bucket nuevo y exclusivo para el estado de la práctica; guardar su nombre para las sesiones posteriores.

- {% include step_label.html id="s013" %} En **Git Bash**, identifica el proyecto y crea un nombre de bucket con sufijo aleatorio.

  > **Nota:** El nombre del bucket debe ser globalmente único, en minúsculas y sin espacios.
  {: .lab-note .info .compact}

  ```bash
  PROJECT_ID="$(gcloud config get-value project)"
  BUCKET_NAME="tfstate-lab3-${PROJECT_ID,,}-$(date +%s)-$RANDOM"
  BUCKET_LOCATION="us-central1"
  PREFIX="terraform/state/lab3"
  printf "Bucket: gs://%s\n" "$BUCKET_NAME"
  ```

  > **Salida esperada:** La terminal muestra el identificador del nuevo bucket.
  {: .lab-note .output .compact}

- {% include step_label.html id="s014" %} Guarda las variables en `lab3/.lab3-bucket.env` para poder recuperarlas si cierras VS Code.

  > **Nota:** Este archivo está excluido en `.gitignore`. No contiene credenciales; conserva el nombre exacto del bucket.
  {: .lab-note .info .compact}

  ```bash
  printf "export BUCKET_NAME=%q\nexport BUCKET_LOCATION=%q\nexport PREFIX=%q\n" "$BUCKET_NAME" "$BUCKET_LOCATION" "$PREFIX" > .lab3-bucket.env
  cat .lab3-bucket.env
  ```

  > **Salida esperada:** El archivo contiene tres declaraciones `export` y puede cargarse con `source .lab3-bucket.env`. 
  {: .lab-note .output .compact}

- {% include step_label.html id="s015" %} En **Git Bash**, crea el bucket con acceso uniforme y bloqueo de acceso público preventivo.

  > **Nota:** Si la organización impone restricciones de ubicación, consulta al instructor antes de cambiar `us-central1`.
  {: .lab-note .info .compact}

  ```bash
  gcloud storage buckets create "gs://$BUCKET_NAME" --project="$PROJECT_ID" --location="$BUCKET_LOCATION" --uniform-bucket-level-access --public-access-prevention
  ```

  > **Salida esperada:** El bucket es creado sin errores y puede consultarse en Cloud Storage.
  {: .lab-note .output .compact}

- {% include step_label.html id="s016" %} Verifica los detalles del bucket que acabas de crear.

  > **Nota:** No utilices buckets existentes que pertenezcan a otras prácticas o participantes.
  {: .lab-note .info .compact}

  ```bash
  gcloud storage buckets describe "gs://$BUCKET_NAME" --format="yaml(name,location,iamConfiguration)"
  ```

  > **Salida esperada:** Aparece la ubicación `US-CENTRAL1` y el acceso uniforme habilitado.
  {: .lab-note .output .compact}

### Tarea 2.2. Habilitar Object Versioning y validar las propiedades

Activar la conservación de generaciones de objeto para recuperar versiones anteriores del estado en caso de error.

- {% include step_label.html id="s017" %} Habilita Object Versioning desde **Git Bash**.

  > **Nota:** El versionado conserva generaciones previas, pero puede generar costos de almacenamiento si no se eliminan.
  {: .lab-note .info .compact}

  ```bash
  gsutil versioning set on "gs://$BUCKET_NAME"
  ```

  > **Salida esperada:** La actualización del bucket termina correctamente.
  {: .lab-note .output .compact}

- {% include step_label.html id="s018" %} Consulta nuevamente las propiedades para verificar que `versioning` esté habilitado.

  > **Nota:** Si aparece `enabled: true`, el bucket está listo para recibir el estado de Terraform.
  {: .lab-note .info .compact}

  ```bash
  gsutil versioning get "gs://$BUCKET_NAME"
  ```

  > **Salida esperada:** La salida muestra `Enabled`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s019" %} Comprueba que el bucket se encuentra vacío antes de migrar el estado.

  > **Nota:** Una salida vacía o un aviso de cero objetos es normal antes de migrar.
  {: .lab-note .info .compact}

  ```bash
  gcloud storage ls "gs://$BUCKET_NAME/"
  ```

  > **Salida esperada:** Todavía no existe un archivo `default.tfstate` en el bucket.
  {: .lab-note .output .compact}

{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## 🔄 Tarea 3. Configurar el backend remoto y migrar el estado — 6 min

{% assign tracking_task_id = "t003" %}

Configurarás el backend GCS y migrarás el estado sin recrear la VPC desplegada.

### Tarea 3.1. Configurar backend.tf y conservar una referencia del estado

Preparar la conexión al bucket sin redefinir variables dentro del bloque backend ni perder el estado ya creado.

- {% include step_label.html id="s020" %} En **Git Bash**, recupera el nombre del bucket guardado en el archivo local.

  > **Nota:** El bucket debe coincidir con el creado en la tarea anterior.
  {: .lab-note .info .compact}

  ```bash
  source .lab3-bucket.env
  printf "Destino: gs://%s/%s\n" "$BUCKET_NAME" "$PREFIX"
  ```

  > **Salida esperada:** La terminal muestra la ruta de destino del backend remoto.
  {: .lab-note .output .compact}

- {% include step_label.html id="s021" %} Antes de la migración, registra los identificadores del estado y de la VPC en archivos temporales.

  > **Nota:** Estos archivos son solo referencias locales de validación; no los subas a Git.
  {: .lab-note .info .compact}

  ```bash
  terraform output -raw vpc_id > .lab3-vpc-before.txt
  terraform state list > .lab3-state-before.txt
  ```

  > **Salida esperada:** Los archivos muestran el identificador de la VPC y el recurso Terraform administrado.
  {: .lab-note .output .compact}

- {% include step_label.html id="s022" %} En **VS Code**, abre `lab3/backend.tf` (créalo si no existe) y configura un backend GCS.

  > **Importante:** **Construcción dirigida:** agrega un bloque `terraform` con `backend "gcs"` que contenga `bucket` con el nombre **literal** mostrado en Git Bash y `prefix = "terraform/state/lab3"`. No uses `var.*`. Documentación: https://developer.hashicorp.com/terraform/language/backend/gcs
  {: .lab-note .important .compact}

  > **Salida esperada:** `backend.tf` contiene un único backend GCS, con nombre real de bucket y prefijo válido.
  {: .lab-note .output .compact}

### Tarea 3.2. Migrar y comprobar que el estado apunta a la misma VPC

Realizar la migración explícita y demostrar que se conserva el recurso existente sin crear una segunda red.

- {% include step_label.html id="s023" %} Ejecuta el comando de migración en **Git Bash**, desde `lab3`.

  > **Nota:** Si Terraform solicita copiar el estado existente, responde `yes`; no uses `-reconfigure`, que no migra el estado.
  {: .lab-note .info .compact}

  ```bash
  terraform init -migrate-state
  ```

  > **Salida esperada:** Terraform termina con `Successfully configured the backend "gcs"!`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s024" %} Consulta los recursos administrados después de la migración.

  > **Nota:** No ejecutes `terraform apply` todavía: primero confirmaremos que el recurso sigue siendo el original.
  {: .lab-note .info .compact}

  ```bash
  terraform state list
  terraform output -raw vpc_id
  ```

  > **Salida esperada:** Continúa apareciendo `google_compute_network.vpc` y el identificador de VPC esperado.
  {: .lab-note .output .compact}

- {% include step_label.html id="s025" %} Comprueba que el identificador del recurso es exactamente el mismo antes y después.

  > **Nota:** `diff` sin diferencias prueba que no se recreó el recurso durante la migración.
  {: .lab-note .info .compact}

  ```bash
  terraform output -raw vpc_id > .lab3-vpc-after.txt
  diff -u .lab3-vpc-before.txt .lab3-vpc-after.txt && echo "OK: misma VPC"
  ```

  > **Salida esperada:** Aparece `OK: misma VPC`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s026" %} Ejecuta un plan de control para verificar que Terraform no quiere recrear la infraestructura.

  > **Nota:** Espera `No changes. Your infrastructure matches the configuration.`. Si aparecen cambios, detente y revisa.
  {: .lab-note .info .compact}

  ```bash
  terraform plan
  ```

  > **Salida esperada:** El plan no propone crear, modificar ni destruir recursos.
  {: .lab-note .output .compact}

- {% include step_label.html id="s027" %} En Git Bash, localiza el objeto de estado del workspace `default` dentro de GCS.

  > **Nota:** El backend GCS utiliza `default.tfstate` para el workspace `default`.
  {: .lab-note .info .compact}

  ```bash
  gcloud storage ls "gs://$BUCKET_NAME/$PREFIX/"
  ```

  > **Salida esperada:** La ruta del objeto remoto incluye `terraform/state/lab3/default.tfstate`.
  {: .lab-note .output .compact}

{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## 🔐 Tarea 4. Verificar versiones del estado y bloqueo de operaciones — 7 min

{% assign tracking_task_id = "t004" %}

Comprobarás generaciones de estado y una condición de bloqueo entre dos terminales Git Bash.

### Tarea 4.1. Consultar el estado remoto y las generaciones de objeto

Observar el archivo `default.tfstate` y registrar su historial de generaciones antes y después de un cambio aplicado.

- {% include step_label.html id="s028" %} Lista las generaciones disponibles de `default.tfstate` desde Git Bash.

  > **Nota:** No compartas el contenido del estado: basta con visualizar nombre, tamaños y generaciones.
  {: .lab-note .info .compact}

  ```bash
  STATE_OBJECT="gs://$BUCKET_NAME/$PREFIX/default.tfstate"
  gcloud storage ls --all-versions --long "$STATE_OBJECT"
  ```

  > **Salida esperada:** Aparece al menos una generación del archivo `default.tfstate`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s029" %} En `lab3/main.tf`, cambia solamente el texto de `description` de la VPC, manteniendo el resto igual.

  > **Importante:** **Construcción dirigida:** dentro de `resource "google_compute_network" "vpc"`, reemplaza el valor de `description` por `Estado remoto versionado - laboratorio 3`. No cambies `name` ni `auto_create_subnetworks`. Documentación: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_network
  {: .lab-note .important .compact}

  > **Salida esperada:** El archivo conserva la misma VPC y únicamente presenta una descripción nueva.
  {: .lab-note .output .compact}

- {% include step_label.html id="s030" %} Verifica con Terraform que la descripción se actualizará sin reemplazar la VPC.

  > **Nota:** El resumen esperado es `Plan: 0 to add, 1 to change, 0 to destroy` si la API admite la actualización.
  {: .lab-note .info .compact}

  ```bash
  terraform fmt
  terraform validate
  terraform plan
  ```

  > **Salida esperada:** El plan muestra una actualización in-place de la descripción; no se crea una VPC adicional.
  {: .lab-note .output .compact}

### Tarea 4.2. Comprobar versionado tras un cambio real

Realizar una modificación segura sobre la VPC y comprobar que Cloud Storage conserva la generación anterior del estado.

- {% include step_label.html id="s031" %} Aplica la actualización y confirma de manera interactiva.

  > **Nota:** Responde `yes` solo si el plan no muestra un reemplazo ni recursos adicionales.
  {: .lab-note .info .compact}

  ```bash
  terraform apply
  ```

  > **Salida esperada:** Terraform termina con una VPC modificada y sin destrucciones.
  {: .lab-note .output .compact}

- {% include step_label.html id="s032" %} Consulta otra vez las generaciones del objeto de estado.

  > **Nota:** El versionado es del objeto en GCS; `terraform plan` por sí solo no garantiza una nueva generación.
  {: .lab-note .info .compact}

  ```bash
  gcloud storage ls --all-versions --long "$STATE_OBJECT"
  ```

  > **Salida esperada:** Deben existir varias generaciones del mismo objeto de estado.
  {: .lab-note .output .compact}

- {% include step_label.html id="s033" %} Comprueba que Terraform sigue gestionando un único recurso y que el estado está actualizado.

  > **Nota:** Si la actualización se aplicó correctamente, el segundo plan no debe proponer cambios.
  {: .lab-note .info .compact}

  ```bash
  terraform state list
  terraform plan
  ```

  > **Salida esperada:** Hay un solo recurso `google_compute_network.vpc` y el plan está sin cambios.
  {: .lab-note .output .compact}

### Tarea 4.3. Demostrar locking sin alterar el estado manualmente

Mantener una sesión interactiva que retiene el lock mientras una segunda terminal intenta planificar el mismo estado.

- {% include step_label.html id="s034" %} Abre una **Terminal A** en VS Code, entra en `lab3`, realiza una modificación menor en la descripción o etiquetas de tu archivo main.tf y ejecuta `terraform apply`.

  > **Importante:** **Importante:** Deja abierta la terminal en la pantalla de confirmación que solicita escribir yes/no; no escribas nada ni canceles el comando hasta terminar la prueba. Esto mantendrá el bloqueo activo en Google Cloud Storage durante la sesión.
  {: .lab-note .important .compact}

  > **Salida esperada:**  La Terminal A se pausa y muestra el prompt de confirmación interactivo de **Terraform (Enter a value:).**
  {: .lab-note .output .compact}

- {% include step_label.html id="s035" %} Abre una **Terminal B** en el **mismo directorio lab3** y lanza un plan sin espera de locking.

  > **Nota:** Si la consola A retiene el lock, la Terminal B debe mostrar `Error acquiring the state lock` y el objeto `.tflock`. No uses `-lock=false`.
  {: .lab-note .info .compact}

  ```bash
  terraform plan -lock-timeout=0s
  ```

  > **Salida esperada:** Se observa el bloqueo concurrente; si no ocurre, no afirmes que la prueba lo demostró.
  {: .lab-note .output .compact}

- {% include step_label.html id="s036" %} En la **Terminal A**, escribe `no` para cancelar el apply de forma segura y repite el plan desde la **Terminal B**.

  > **Nota:** Ejecuta `no` **en el prompt de confirmación de terraform**, no cierres la shell de VS Code. Esto liberará automáticamente el candado en tu bucket de GCS. Si el bloqueo persiste, revisa sesiones abiertas antes de actuar.
  {: .lab-note .info .compact}

  ```bash
  terraform plan
  ```

  > **Salida esperada:** Una vez liberado el lock, Terraform ejecuta el plan y muestra `No changes`.
  {: .lab-note .output .compact}

{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

## 🧹 Tarea 5. Destruir la VPC y retirar el bucket remoto — 4 min

{% assign tracking_task_id = "t005" %}

Destruirás la infraestructura, desconectarás el backend y eliminarás exclusivamente el bucket de esta práctica.

### Tarea 5.1. Destruir infraestructura y verificar la ausencia de recursos

Liberar la VPC creada para la práctica manteniendo el backend GCS disponible durante todo el proceso de destrucción.

- {% include step_label.html id="s037" %} Desde Git Bash, confirma el proyecto y el recurso exacto que vas a eliminar.

  > **Nota:** No ejecutes destrucción si la lista contiene recursos que no pertenecen a esta práctica.
  {: .lab-note .info .compact}

  ```bash
  gcloud config get-value project
  terraform state list
  ```

  > **Salida esperada:** Solo figura `google_compute_network.vpc`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s038" %} Ejecuta `terraform destroy` con confirmación interactiva.

  > **Advertencia:** **Advertencia:** revisa que el plan destruya solo la VPC de `lab3`; después confirma con `yes`.
  {: .lab-note .warning .compact}

  ```bash
  terraform destroy
  ```

  > **Salida esperada:** Aparece `Destroy complete! Resources: 1 destroyed.`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s039" %} Confirma que ya no hay recursos administrados y que el backend remoto sigue accesible.

  > **Nota:** No elimines el bucket si cualquiera de estos comandos falla: el estado debe seguir accesible.
  {: .lab-note .info .compact}

  ```bash
  terraform state list
  terraform plan
  ```

  > **Salida esperada:** No hay recursos listados y el plan informa que no hay cambios.
  {: .lab-note .output .compact}

### Tarea 5.2. Desconectar el backend y eliminar el bucket de la práctica

Completar la limpieza sin dejar Terraform configurado hacia un bucket eliminado ni borrar estados de otros laboratorios.

- {% include step_label.html id="s040" %} En **VS Code**, elimina únicamente el archivo `lab3/backend.tf` y reinicializa el backend local vacío.

  > **Nota:** Hazlo solo **después** de comprobar que no existen recursos. `-reconfigure` no migra estado; aquí se usa para dejar de apuntar al bucket.
  {: .lab-note .info .compact}

  ```bash
  rm backend.tf
  terraform init -reconfigure
  ```

  > **Salida esperada:** Terraform queda inicializado con backend local. El estado remoto anterior permanece en GCS hasta eliminar el bucket.
  {: .lab-note .output .compact}

- {% include step_label.html id="s041" %} Consulta el nombre exacto del bucket y elimina **solo el bucket exclusivo de lab3** con todas sus versiones.

  > **Advertencia:** **Advertencia:** `--recursive` también elimina las generaciones antiguas y el bucket. Revisa el nombre antes de confirmar. No uses buckets compartidos.
  {: .lab-note .warning .compact}

  ```bash
  source .lab3-bucket.env
  echo "BUCKET A ELIMINAR: gs://$BUCKET_NAME"
  gcloud storage rm --recursive "gs://$BUCKET_NAME/"
  ```

  > **Salida esperada:** Cloud Storage informa la eliminación de los objetos versionados y del bucket.
  {: .lab-note .output .compact}

- {% include step_label.html id="s042" %} Verifica que el bucket no existe y conserva los archivos fuente para estudiar la migración.

  > **Nota:** Un error de consulta no prueba por sí solo la eliminación; confirma en la consola de Cloud Storage si hay dudas.
  {: .lab-note .info .compact}

  ```bash
  gcloud storage buckets describe "gs://$BUCKET_NAME" 2>&1 || echo "OK: bucket ausente o sin acceso"
  terraform validate
  ```

  > **Salida esperada:** La configuración local es válida y la VPC y el bucket de la práctica ya no están disponibles.
  {: .lab-note .output .compact}

{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

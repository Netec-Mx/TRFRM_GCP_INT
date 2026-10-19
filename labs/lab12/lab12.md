---
layout: lab
title: "Práctica 12: Estimar costos con Infracost"
permalink: /lab12/lab12/
images_base: /labs/lab12/img
duration: "25 minutos"
objective:
  - "Estimar el costo mensual de recursos Compute Engine y Cloud Storage definidos con Terraform mediante Infracost, conservar una línea base y comparar modificaciones sin desplegar infraestructura."
prerequisites:
  - "Prácticas 1 a 11 completadas; carpeta terraform-gcp-intermediate creada en el Escritorio de Windows."
  - "VM Windows con VS Code, Git Bash, Terraform y Google Cloud CLI; conexión a Internet para Infracost y proveedores."
  - "Cuenta gratuita de Infracost preparada; para obtener un plan de Terraform con Google Provider, proyecto GCP y ADC con permisos de lectura habilitados."
introduction:
  - "Definirás una VPC, una VM Debian y un bucket GCS sin ejecutarlos. Calcularás el costo mensual estimado con Infracost, generarás el plan JSON de Terraform como evidencia complementaria y guardarás la estimación inicial. Después cambiarás tipo de máquina y clase de almacenamiento para observar la diferencia de costos antes de decidir un despliegue."
slug: lab12
lab_number: 12
final_result: >
  Proyecto lab12 con configuración Terraform de GCP, estimación de costos en tabla y JSON, plan JSON opcional de contraste y comparativa Infracost entre la línea base y la configuración modificada; sin recursos desplegados ni cargos por infraestructura del laboratorio.
notes:
  - "Ejecuta comandos desde Git Bash integrado en VS Code dentro de terraform-gcp-intermediate/lab12, salvo indicación contraria."
  - "No ejecutes terraform apply ni terraform destroy; un plan y una estimación no crean infraestructura GCP."
  - "Infracost requiere iniciar sesión; la cuenta gratuita debe estar preparada para cumplir los 25 minutos de clase."
  - "Los importes son estimaciones sujetas a tarifas, regiones, descuentos y consumo. GCS depende de GB, operaciones y recuperación; sin datos de uso no se representa necesariamente todo el costo."
  - "Guarda infracost-base.json ANTES de modificar el código; si lo regeneras después, perderás la referencia para diff."
  - "No publiques planes JSON, identificadores sensibles ni credenciales. El plan JSON puede incluir valores de configuración que requieren revisión."
references:
  - text: "Infracost: documentación oficial"
    url: https://www.infracost.io/docs/
  - text: "Infracost: CLI"
    url: https://www.infracost.io/docs/features/cli_commands/
  - text: "Infracost: archivos de uso"
    url: https://www.infracost.io/docs/features/usage_based_resources/
  - text: "Infracost: versiones para Windows"
    url: https://github.com/infracost/infracost/releases
  - text: "Terraform: comando show -json"
    url: https://developer.hashicorp.com/terraform/cli/commands/show
  - text: "Google Cloud: precios"
    url: https://cloud.google.com/pricing
prev: /lab11/lab11/
next: /lab13/lab13/
---

---

## 🔎 Tarea 1. Preparar el proyecto Terraform para estimación — 6 min

Crearás `lab12` sin alterar las prácticas anteriores y definirás una VM, una VPC y un bucket de ejemplo. Los recursos permanecerán únicamente en el código.

### Tarea 1.1. Preparar directorios y herramientas

Abrir la raíz del curso, crear una carpeta independiente y verificar las herramientas disponibles en Git Bash.

- {% include step_label.html %} En **VS Code**, abre **File > Open Folder...** y selecciona la carpeta `terraform-gcp-intermediate` del Escritorio; abre **Terminal > New Terminal** y elige **Git Bash**.

  > **Salida esperada:** El Explorer muestra `lab1` a `lab11` y la terminal muestra un prompt Bash.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **Git Bash**, identifica la ubicación real del Escritorio de Windows y crea exclusivamente `lab12`.

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '
  ')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab12/reports"
  cd "$DESKTOP/terraform-gcp-intermediate/lab12"
  pwd
  ```

  > **Salida esperada:** La ruta termina en `/terraform-gcp-intermediate/lab12` y existe la carpeta `reports`, incluso si el Escritorio usa OneDrive.
  {: .lab-note .output .compact}

- {% include step_label.html %} Crea los archivos de Terraform y un `.gitignore` que excluya estado, planes y reportes de la publicación accidental.

  ```bash
  touch providers.tf variables.tf main.tf terraform.tfvars .gitignore
  cat > .gitignore <<'EOF'
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  tfplan.json
  *.tfvars
  reports/
  infracost-base.json
  infracost-nuevo.json
  EOF
  terraform version
  gcloud version | head -n 3
  ```

  > **Salida esperada:** Existen los archivos y las herramientas muestran versiones; `.gitignore` protege los artefactos locales.
  {: .lab-note .output .compact}

### Tarea 1.2. Escribir infraestructura representativa

Declarar un ejemplo que combine cómputo y almacenamiento. Conservaremos el código legible en archivos separados, con la infraestructura propia del laboratorio.

- {% include step_label.html %} En **VS Code**, abre `lab12/providers.tf` y declara Terraform y el Google Provider. No incluyas claves JSON.

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
    zone    = var.zone
  }
  ```

  > **Salida esperada:** La configuración del proveedor depende de variables y no contiene credenciales.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `variables.tf`, define proyecto, región, zona, tipo de máquina y clase de almacenamiento. Estas dos últimas entradas se compararán después.

  ```hcl
  variable "project_id" {
    description = "ID del proyecto GCP usado para validar el plan"
    type        = string
  }
  variable "region" {
    description = "Región de recursos de ejemplo"
    type        = string
    default     = "us-central1"
  }
  variable "zone" {
    description = "Zona de la VM de ejemplo"
    type        = string
    default     = "us-central1-a"
  }
  variable "machine_type" {
    description = "Tipo de VM para estimación"
    type        = string
    default     = "e2-medium"
  }
  variable "storage_class" {
    description = "Clase de almacenamiento GCS"
    type        = string
    default     = "STANDARD"
  }
  ```

  > **Salida esperada:** Existen cinco variables, con `machine_type = "e2-medium"` y `storage_class = "STANDARD"` como valores iniciales.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `main.tf`, agrega la VPC de ejemplo y la VM. **Construcción dirigida 1/4:** conecta `machine_type` con `var.machine_type` y fija un disco `pd-standard` de 20 GB.

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = "lab12-infracost-vpc"
    auto_create_subnetworks = true
  }
  
  resource "google_compute_instance" "vm" {
    name         = "lab12-infracost-vm"
    machine_type = var.machine_type
    zone         = var.zone
    tags         = ["demo", "costos"]
  
    boot_disk {
      initialize_params {
        image = "projects/debian-cloud/global/images/family/debian-12"
        size  = 20
        type  = "pd-standard"
      }
    }
  
    network_interface {
      network = google_compute_network.vpc.id
      access_config {}
    }
  
    labels = { env = "dev" }
  }
  ```

  > **Nota:** La red es exclusiva de `lab12` y no depende de que exista la VPC `default`. La IP externa y el disco pueden representar cargos adicionales.
  {: .lab-note .info .compact}

  > **Salida esperada:** El recurso Compute Engine usa `e2-medium`, disco de 20 GB e IP externa, vinculado a la VPC de ejemplo.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Debajo de la VM**, añade el bucket GCS y prepara `terraform.tfvars` con el proyecto real. El bucket no se creará.

  ```hcl
  resource "google_storage_bucket" "bucket" {
    name                        = "lab12-infracost-ejemplo-no-deploy"
    location                    = var.region
    storage_class               = var.storage_class
    uniform_bucket_level_access = true
    force_destroy               = false
    labels                      = { env = "dev" }
  
    lifecycle_rule {
      condition { age = 30 }
      action { type = "Delete" }
    }
  }
  ```

  > **Nota:** En `terraform.tfvars` escribe `project_id = "TU_PROJECT_ID"` y sustituye el marcador por tu ID obtenido con `gcloud config get-value project`. El nombre del bucket es ilustrativo; no necesitas reservarlo.
  {: .lab-note .info .compact}

  > **Salida esperada:** El código contiene una VPC, una VM y un bucket con clase `STANDARD`, sin ejecutar `apply`.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## 🔎 Tarea 2. Instalar y autenticar Infracost — 5 min

Comprobarás que Infracost CLI esté disponible en Windows y vincularás una cuenta del servicio, necesaria para consultar estimaciones.

### Tarea 2.1. Comprobar o instalar la CLI

Evitar instalaciones innecesarias y utilizar una opción aprobada para la VM Windows.

- {% include step_label.html %} En **Git Bash**, comprueba si Infracost ya está en `PATH` antes de instalarlo.

  ```bash
  if command -v infracost >/dev/null 2>&1; then
    infracost --version
  else
    echo "Infracost no disponible en PATH"
  fi
  ```

  > **Salida esperada:** Infracost muestra una versión o queda confirmado que debes instalarlo.
  {: .lab-note .output .compact}

- {% include step_label.html %} Si falta Infracost, instala la herramienta con Chocolatey en una terminal administrativa o utiliza el ejecutable oficial Windows amd64.

  **Instrucciones:**

  - Si Chocolatey está disponible y tienes autorización, ejecuta `choco install infracost -y` desde una terminal de Windows con privilegios adecuados.
  - Alternativamente, abre [Infracost Releases](https://github.com/infracost/infracost/releases), descarga la distribución Windows amd64, extrae `infracost.exe` y colócalo en una carpeta de herramientas del usuario.
  - Regresa a Git Bash y verifica `infracost --version`; si se descargó a una carpeta no incluida en `PATH`, agrega esa ruta antes de continuar.

  > **Nota:** No instales extensiones de VS Code como requisito: son opcionales.
  {: .lab-note .info .compact}

  > **Salida esperada:** `infracost --version` responde correctamente desde Git Bash.
  {: .lab-note .output .compact}

### Tarea 2.2. Acceder y verificar la cuenta

Abrir la autenticación de Infracost sin copiar claves privadas a Terraform.

- {% include step_label.html %} En el navegador, accede a [Infracost Dashboard](https://dashboard.infracost.io/) y entra o registra una cuenta utilizando el método permitido por el curso.

  > **Salida esperada:** La cuenta de Infracost está disponible; no usaste enlaces de inicio de sesión con tokens temporales.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **Git Bash**, ejecuta la autenticación interactiva de la CLI y finaliza el procedimiento abierto en el navegador.

  ```bash
  infracost auth login
  ```

  > **Nota:** No copies tokens de autenticación a `.tf` o al repositorio.
  {: .lab-note .info .compact}

  > **Salida esperada:** Infracost informa que la CLI quedó autenticada.
  {: .lab-note .output .compact}

- {% include step_label.html %} Valida que la autenticación esté vigente y anota únicamente el estado de la sesión, sin revelar credenciales.

  ```bash
  infracost auth whoami
  infracost --version
  ```

  > **Salida esperada:** Se identifica la cuenta conectada y la versión de la CLI.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## 🔎 Tarea 3. Generar el costo inicial y guardar la línea base — 5 min

Inicializarás el proyecto, obtendrás evidencia de un plan Terraform y generarás la estimación inicial que posteriormente se utilizará como referencia.

### Tarea 3.1. Inicializar Terraform y exportar el plan JSON

Comprobar la sintaxis HCL y guardar un plan solo de lectura, sin aprovisionar recursos.

- {% include step_label.html %} Desde `lab12`, normaliza el formato e inicializa el proveedor Google.

  ```bash
  terraform fmt -recursive
  terraform init
  terraform validate
  ```

  > **Salida esperada:** Terraform finaliza la inicialización y `validate` muestra `Success! The configuration is valid.`
  {: .lab-note .output .compact}

- {% include step_label.html %} Comprueba el proyecto y las credenciales ADC **solo para la generación del plan Terraform**. Si faltan, autentica la cuenta autorizada.

  ```bash
  PROJECT_ID="$(gcloud config get-value project)"
  echo "Proyecto configurado: $PROJECT_ID"
  gcloud auth application-default print-access-token >/dev/null && echo "ADC disponible"
  ```

  > **Nota:** Si falla ADC, ejecuta `gcloud auth application-default login`. El análisis directo de Infracost sobre HCL no exige esa autenticación; `terraform plan` sí puede necesitarla.
  {: .lab-note .info .compact}

  > **Salida esperada:** El proyecto activo coincide con `terraform.tfvars` y, cuando hay permisos, se muestra `ADC disponible`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Genera un plan guardado y conviértelo a JSON. **No ejecutes `terraform apply`**.

  ```bash
  terraform plan -out=lab12.tfplan
  terraform show -json lab12.tfplan > tfplan.json
  ls -lh lab12.tfplan tfplan.json
  ```

  > **Nota:** Si las políticas GCP impiden obtener el plan, registra el error y continúa con las estimaciones desde HCL: `infracost breakdown --path .`. No inventes un plan válido.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se crean `lab12.tfplan` y `tfplan.json`, con la VM, VPC y bucket propuestos para alta.
  {: .lab-note .output .compact}

### Tarea 3.2. Obtener y conservar la estimación original

Usar Infracost para examinar la VM y GCS y guardar una línea base consistente anterior a cualquier modificación.

- {% include step_label.html %} Ejecuta el reporte de costos en tabla desde HCL y, si obtuviste el plan JSON, úsalo adicionalmente para contrastar valores.

  ```bash
  infracost breakdown --path . --format table
  # Ejecutar SOLO si se generó tfplan.json correctamente:
  # infracost breakdown --path tfplan.json --format table
  ```

  > **Nota:** El archivo de plan debe consumirse como JSON, no como el binario `.tfplan`. Algunos rubros variables de GCS necesitan supuestos de uso.
  {: .lab-note .info .compact}

  > **Salida esperada:** Aparece un desglose mensual estimado de los recursos cubiertos por Infracost; los valores no tienen que ser idénticos a los de otro participante.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Antes de cambiar el HCL**, genera la línea base JSON que utilizará `infracost diff`.

  ```bash
  infracost breakdown --path . --format json --out-file infracost-base.json
  ls -lh infracost-base.json
  ```

  > **Salida esperada:** Existe `infracost-base.json` y representa `e2-medium` con GCS `STANDARD`.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Construcción dirigida 2/4.** Identifica en el reporte inicial qué recurso concentra el costo fijo estimado y qué conceptos de GCS dependen del consumo.

  **Instrucciones:**

  - Revisa la fila de `google_compute_instance.vm` y el total mensual estimado.
  - Distingue cómputo, disco e IP externa si aparecen desglosados.
  - Comprueba si el bucket muestra costo cuantificado o componentes cuyo uso no se conoce.
  - Anota brevemente la observación; no supongas importes exactos.

  > **Salida esperada:** Identificas los conceptos principales de la línea base y sabes qué datos de uso faltan para un presupuesto exhaustivo.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## 🔎 Tarea 4. Modificar infraestructura y comparar estimaciones — 6 min

Actualizarás dos atributos que impactan la estimación y compararás los resultados contra la línea base guardada, sin modificar infraestructura real.

### Tarea 4.1. Modificar VM y clase de almacenamiento

Editar el código HCL original en la ubicación correcta, conservando el resto del proyecto.

- {% include step_label.html %} En **VS Code**, abre `lab12/variables.tf` y ubica `variable "machine_type"`. **Construcción dirigida 3/4:** cambia únicamente su valor predeterminado a `e2-standard-2`.

  ```hcl
  variable "machine_type" {
    description = "Tipo de VM para estimación"
    type        = string
    default     = "e2-standard-2"
  }
  ```

  > **Salida esperada:** La VM sigue usando `var.machine_type` en `main.tf`, pero ahora apunta a `e2-standard-2`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de `machine_type`, localiza `variable "storage_class"` y cambia su valor por `NEARLINE`.

  ```hcl
  variable "storage_class" {
    description = "Clase de almacenamiento GCS"
    type        = string
    default     = "NEARLINE"
  }
  ```

  > **Salida esperada:** El bucket continúa usando `var.storage_class` y la nueva clase es `NEARLINE`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Valida el formato y confirma en Git Bash que los dos valores modificados aparecen solo en las definiciones esperadas.

  ```bash
  terraform fmt -recursive
  terraform validate
  grep -nE 'e2-standard-2|NEARLINE' variables.tf
  ```

  > **Salida esperada:** Terraform valida correctamente y `grep` muestra ambos valores en `variables.tf`.
  {: .lab-note .output .compact}

### Tarea 4.2. Calcular diferencia frente a la línea base

Generar la nueva estimación y obtener el cambio mensual mediante `infracost diff`.

- {% include step_label.html %} Ejecuta una nueva estimación JSON sin sobrescribir `infracost-base.json`.

  ```bash
  infracost breakdown --path . --format json --out-file infracost-nuevo.json
  ls -lh infracost-base.json infracost-nuevo.json
  ```

  > **Salida esperada:** Existen dos reportes diferentes: original y modificado.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Construcción dirigida 4/4.** Compara la configuración actual con la línea base mediante el comando documentado por Infracost.

  ```bash
  infracost diff --path . --compare-to infracost-base.json --format table
  ```

  > **Nota:** El cambio de clase del bucket puede necesitar cantidades de almacenamiento y operaciones para obtener una diferencia monetaria representativa.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se presenta el cambio de costo estimado frente a la configuración inicial, especialmente el impacto del tipo de VM.
  {: .lab-note .output .compact}

- {% include step_label.html %} Interpreta el resultado en lugar de asumir que todas las modificaciones generan incrementos o ahorros.

  **Instrucciones:**

  - Localiza el costo anterior y el nuevo costo mensual cuando estén disponibles.
  - Determina si el cambio de VM incrementa el costo estimado y cuál es su contribución.
  - Verifica si GCS aparece con estimación completa, parcial o sin datos de uso.
  - Relaciona la comparación con una revisión de Pull Request antes de un futuro `apply`.

  > **Salida esperada:** Puedes explicar la variación del costo mensual y reconocer rubros no cuantificados por falta de consumo.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

## 🔎 Tarea 5. Exportar y verificar reportes de costos — 3 min

Reunirás los artefactos del laboratorio y comprobarás que las estimaciones se generaron sin crear ningún recurso en GCP.

### Tarea 5.1. Guardar reportes legibles para revisión

Conservar una salida de comparación y los JSON para utilizarlos como evidencia técnica.

- {% include step_label.html %} Exporta el detalle de la nueva estimación a un archivo de texto y guarda la comparación de costos.

  ```bash
  infracost breakdown --path . --format table > reports/costo-nuevo.txt
  infracost diff --path . --compare-to infracost-base.json --format table > reports/diferencia.txt
  ls -lh reports
  ```

  > **Salida esperada:** `reports/costo-nuevo.txt` y `reports/diferencia.txt` existen y contienen reportes legibles.
  {: .lab-note .output .compact}

- {% include step_label.html %} Comprueba los archivos de la línea base y de la nueva estimación sin imprimir llaves o tokens.

  ```bash
  ls -lh infracost-base.json infracost-nuevo.json reports/*.txt
  sed -n '1,28p' reports/diferencia.txt
  ```

  > **Salida esperada:** Puedes leer el resumen de cambios y existen ambos JSON con estimaciones.
  {: .lab-note .output .compact}

### Tarea 5.2. Validar que no hubo despliegue

Confirmar que solo se ejecutaron planes y estimaciones y cerrar la actividad sin destruir infraestructura ajena.

- {% include step_label.html %} Confirma que no ejecutaste `apply` ni creaste estado con recursos administrados. Verifica desde Git Bash el contenido del directorio.

  ```bash
  pwd
  ls -la main.tf variables.tf providers.tf reports
  if [ -f terraform.tfstate ]; then
    echo "Existe un archivo de estado local; revísalo antes de finalizar"
  else
    echo "No existe estado local de recursos aplicados en lab12"
  fi
  ```

  > **Nota:** La ausencia de `terraform.tfstate` es un indicio local, no una auditoría completa de recursos de GCP. No ejecutes `terraform destroy` porque no se aplicó esta configuración.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se muestran archivos HCL y reportes; no hay despliegue realizado por esta práctica.
  {: .lab-note .output .compact}

- {% include step_label.html %} Revisa en **VS Code** los archivos finales y anota cuál cambio requiere aprobación presupuestaria antes de aplicar infraestructura.

  **Instrucciones:**

  - Comprueba que `infracost-base.json` no fue sobrescrito después de editar variables.
  - Conserva `reports/diferencia.txt` como evidencia de la variación mensual estimada.
  - Explica qué información adicional se necesita para estimar con precisión almacenamiento GCS.
  - No compartas archivos JSON de planes o autenticación en repositorios públicos sin revisarlos.

  > **Salida esperada:** Queda documentada la comparación de costos para revisión; no se han desplegado recursos GCP.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}
---
layout: lab
title: "Práctica 10: Escanear código con tfsec"
permalink: /lab10/lab10/
images_base: /labs/lab10/img
duration: "20 minutos"
objective:
  - "Analizar código Terraform de Google Cloud con tfsec para identificar configuraciones inseguras, interpretar hallazgos, corregir recursos y exportar un reporte SARIF sin desplegar infraestructura."
prerequisites:
  - "Prácticas 1 a 9 completadas y carpeta terraform-gcp-intermediate en el Escritorio de Windows."
  - "VS Code con terminal Git Bash y Terraform instalado; acceso a Internet para obtener tfsec si todavía no está instalado."
  - "Permisos para instalar tfsec en Windows o descargar el ejecutable oficial en una carpeta del usuario; no se requieren credenciales de GCP para el escaneo."
introduction:
  - "Crearás una configuración Terraform de demostración con una regla de SSH abierta a Internet y un bucket con controles insuficientes. Analizarás el código sin aplicar recursos en GCP, corregirás los hallazgos y exportarás reportes SARIF inicial y final para comparar resultados."
slug: lab10
lab_number: 10
final_result: >
  Proyecto lab10 con un ciclo reproducible de escaneo estático, correcciones de red y almacenamiento,
  y reportes SARIF de antes y después. No se crean recursos GCP.
notes:
  - "Ejecuta comandos en Git Bash integrado en VS Code dentro de terraform-gcp-intermediate/lab10, salvo indicación expresa de la interfaz."
  - "Esta práctica es exclusivamente de análisis estático: NO ejecutes terraform apply ni terraform destroy. No hay recursos creados."
  - "Los ejemplos iniciales son deliberadamente inseguros: no reutilices esos bloques en infraestructura real."
  - "Las reglas, severidades y cantidades de hallazgos pueden variar según la versión de tfsec; evalúa siempre los recursos y controles específicos."
  - "tfsec forma parte del ecosistema Trivy; se mantiene como herramienta del programa del curso y se recomienda considerar Trivy para pipelines nuevos."
references:
  - text: "tfsec: documentación de uso y formatos"
    url: https://aquasecurity.github.io/tfsec/latest/guides/usage/
  - text: "tfsec: versiones y ejecutables Windows"
    url: https://github.com/aquasecurity/tfsec/releases
  - text: "tfsec: comprobaciones GCP"
    url: https://aquasecurity.github.io/tfsec/latest/checks/google/
  - text: "Google Cloud: IAP TCP forwarding"
    url: https://cloud.google.com/iap/docs/using-tcp-forwarding
  - text: "Google Cloud Storage: Uniform bucket-level access"
    url: https://cloud.google.com/storage/docs/uniform-bucket-level-access
  - text: "Trivy y tfsec"
    url: https://github.com/aquasecurity/tfsec
prev: /lab9/lab9/
next: /lab11/lab11/
---
---

## 🔎 Tarea 1. Preparar el proyecto y verificar tfsec — 5 min

Crear la carpeta exclusiva de esta práctica y comprobar que el ejecutable de análisis estático esté disponible para Windows sin modificar las prácticas anteriores.

### Tarea 1.1. Crear directorio lab10 y archivos

Preparar los archivos Terraform y la carpeta de reportes desde la raíz del curso para aislar los ejemplos inseguros.

- {% include step_label.html %} En **VS Code**, abre la carpeta principal del curso y selecciona la terminal **Git Bash**. No abras ni modifiques los archivos de `lab9`.

  **Acciones en VS Code:**

  1. En el menú, selecciona **File > Open Folder...** y localiza **Escritorio > terraform-gcp-intermediate**.
  2. Selecciona **Terminal > New Terminal** y, si es necesario, cambia el perfil a **Git Bash**.
  3. Comprueba en **Explorer** que las carpetas `lab1` a `lab9` estén presentes.

  > **Salida esperada:** VS Code muestra la raíz del curso y la terminal activa indica un entorno Bash.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **Git Bash**, crea explícitamente `lab10`, entra en la carpeta y genera los archivos vacíos. La carpeta `reports` almacenará los resultados SARIF de ambos escaneos. El comando detecta la ruta real del Escritorio de Windows, incluso si está redirigido.

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab10/reports"
  cd "$DESKTOP/terraform-gcp-intermediate/lab10"
  ```
  ```bash
  touch providers.tf variables.tf main.tf
  ```
  ```bash
  cat > .gitignore <<'EOF'
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  *.tfvars
  credentials*.json
  EOF
  pwd
  ls -la
  ```

  > **Salida esperada:** `pwd` termina en `/terraform-gcp-intermediate/lab10` (también si Escritorio se encuentra en OneDrive); aparecen `providers.tf`, `variables.tf`, `main.tf`, `.gitignore` y `reports`; el `.gitignore` excluye estado y credenciales.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **Git Bash**, identifica la versión de Terraform y comprueba si `tfsec` ya está disponible. No ejecutes comandos de autenticación GCP: esta práctica inspecciona archivos locales.

  ```bash
  terraform version
  ```
  ```bash
  if command -v tfsec >/dev/null 2>&1; then
    tfsec --version
  else
    echo "tfsec no está instalado o no está en PATH"
  fi
  ```

  > **Salida esperada:** Terraform muestra su versión y conoces si necesitas instalar tfsec; la ausencia de tfsec se resuelve en la siguiente subtarea.
  {: .lab-note .output .compact}


### Tarea 1.2. Instalar o localizar tfsec

Utilizar el paquete de Windows cuando Chocolatey esté disponible o recuperar el ejecutable oficial sin instalar extensiones obligatorias.

- {% include step_label.html %} Si **tfsec no está instalado**, revisa si Chocolatey existe en Windows. Solo si está disponible y tienes permisos administrativos, puedes instalar el paquete desde una **terminal con privilegios de administrador**, reiniciando después Git Bash.

  ```bash
  if command -v choco >/dev/null 2>&1; then
    echo "Chocolatey disponible; opción administrativa: choco install tfsec -y"
  else
    echo "Chocolatey no disponible; utiliza la descarga oficial del siguiente paso"
  fi
  ```

  > **Nota:** No ejecutes `choco install` dentro de una terminal sin permisos elevados. Si tfsec ya funciona, omite la instalación y continúa con la verificación.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se identifica si Chocolatey está disponible; este paso no altera la máquina por sí mismo.
  {: .lab-note .output .compact}


- {% include step_label.html %} Si no cuentas con Chocolatey, descarga el ejecutable oficial de Windows desde la página de versiones y cópialo a una carpeta de herramientas del usuario.

  **Acciones o requisitos:**

  - Abre [Releases de tfsec](https://github.com/aquasecurity/tfsec/releases) y entra en la versión publicada que utilizará el curso.
  - Expande **Assets** y descarga el archivo llamado **`tfsec-windows-amd64.exe`** para Windows x64; evita descargar `tfsec-checkgen`, que es otra utilidad.
  - Crea `C:\Users\TU_USUARIO\tools\tfsec` y guarda ahí el ejecutable. Cambia su nombre a `tfsec.exe` para facilitar su uso.
  - Si Windows muestra una advertencia de seguridad, verifica la procedencia y firma del archivo antes de autorizarlo; si la política de la VM impide ejecutarlo, pide al instructor preparar la herramienta.

  > **Nota:** La instalación desde Chocolatey y la descarga directa son alternativas. Si tfsec ya está instalado, no necesitas repetir ninguna.
  {: .lab-note .info .compact}

  > **Salida esperada:** Existe un ejecutable `tfsec.exe` en la carpeta de herramientas o se instaló mediante el administrador de paquetes.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **Git Bash**, agrega temporalmente al `PATH` la ubicación del ejecutable descargado y verifica que se ejecute. Si lo instalaste con Chocolatey, bastará con comprobar la versión.

  ```bash
  if ! command -v tfsec >/dev/null 2>&1; then
    export PATH="$(cygpath -u "$USERPROFILE")/tools/tfsec:$PATH"
  fi
  command -v tfsec
  ```
  ```bash
  tfsec --version
  ```

  > **Advertencia:** Si `command -v tfsec` no muestra una ruta, revisa que el archivo se llame exactamente `tfsec.exe` y esté en la ubicación indicada. Si reabres Git Bash deberás volver a agregar la ruta o configurarla permanentemente.
  {: .lab-note .warning .compact}

  > **Salida esperada:** `command -v tfsec` muestra la ubicación del ejecutable y `tfsec --version` informa una versión de tfsec.
  {: .lab-note .output .compact}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## 🔎 Tarea 2. Crear configuraciones Terraform inseguras de ejemplo — 4 min

Escribir recursos mínimos de Google Cloud con dos tipos de riesgos: SSH accesible desde Internet y bucket con controles de acceso insuficientes. Ningún recurso se despliega.

### Tarea 2.1. Configurar Terraform y variables

Separar el Provider de las entradas para distinguir la sintaxis HCL del análisis de seguridad.

- {% include step_label.html %} En el panel **Explorer**, abre `lab10/providers.tf` y agrega el Provider de Google. El archivo declara la dependencia, pero no se conectará a Google Cloud durante el escaneo.

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

  > **Salida esperada:** `providers.tf` contiene un bloque `terraform` y otro `provider "google"`, sin credenciales ni claves JSON.
  {: .lab-note .output .compact}


- {% include step_label.html %} Abre `lab10/variables.tf` y declara un ID de proyecto ficticio para analizar únicamente el código. No necesitas usar el proyecto real de tu cuenta.

  ```hcl
  variable "project_id" {
    description = "ID de ejemplo para análisis estático; no se despliega"
    type        = string
    default     = "demo-lab10-no-deploy"
  }

  variable "region" {
    description = "Región de los recursos de demostración"
    type        = string
    default     = "us-central1"
  }
  ```

  > **Salida esperada:** El proyecto y la región tienen valores explícitos de demostración; el análisis estático puede ejecutarse sin `terraform.tfvars`.
  {: .lab-note .output .compact}


### Tarea 2.2. Definir red y bucket con riesgos reproducibles

Crear recursos intencionalmente inseguros para que tfsec muestre hallazgos identificables y posteriormente corregibles.

- {% include step_label.html %} En `lab10/main.tf`, declara una VPC y un Firewall que permite SSH desde cualquier dirección IPv4. **No despliegues** esta configuración: solo se utilizará como caso de análisis.

  ```hcl
  # INSEGURO A PROPÓSITO: no aplicar a GCP.
  resource "google_compute_network" "vpc" {
    name                    = "lab10-tfsec-vpc"
    auto_create_subnetworks = false
  }

  resource "google_compute_firewall" "ssh" {
    name      = "lab10-ssh-inseguro"
    network   = google_compute_network.vpc.name
    direction = "INGRESS"

    allow {
      protocol = "tcp"
      ports    = ["22"]
    }

    source_ranges = ["0.0.0.0/0"]
    target_tags   = ["ssh"]
  }
  ```

  > **Advertencia:** `0.0.0.0/0` permite tráfico SSH de todo Internet hacia instancias objetivo que cumplan las demás condiciones de firewall. No utilices esta regla en ambientes reales.
  {: .lab-note .warning .compact}

  > **Salida esperada:** La VPC y el recurso Firewall aparecen declarados; el origen abierto es claramente visible en `source_ranges`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Debajo del Firewall, añade un bucket sin **Uniform bucket-level access (UBLA)**. Este control no significa por sí solo que el bucket sea público, pero permite reproducir una recomendación de seguridad sobre el uso de ACL.

  ```hcl
  # INSEGURO A PROPÓSITO: UBLA deshabilitado.
  resource "google_storage_bucket" "demo" {
    name                        = "lab10-demo-no-deploy-ejemplo"
    location                    = var.region
    uniform_bucket_level_access = false
    public_access_prevention    = "inherited"
    force_destroy               = false
  }
  ```

  > **Nota:** El nombre del bucket es ilustrativo y no necesita ser globalmente único porque no ejecutaremos `terraform apply`.
  {: .lab-note .info .compact}

  > **Salida esperada:** `google_storage_bucket.demo` aparece debajo del Firewall con `uniform_bucket_level_access = false`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Para mostrar una segunda vulnerabilidad independiente, añade una asignación IAM que concede lectura pública de objetos. **No la apliques**. Se eliminará por completo durante la corrección.

  ```hcl
  # INSEGURO A PROPÓSITO: lectura pública anónima.
  resource "google_storage_bucket_iam_member" "public_read" {
    bucket = google_storage_bucket.demo.name
    role   = "roles/storage.objectViewer"
    member = "allUsers"
  }
  ```

  > **Salida esperada:** El código contiene una asignación `allUsers` claramente identificable, además de la configuración UBLA deshabilitada.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## 🔎 Tarea 3. Ejecutar tfsec e interpretar los hallazgos — 4 min

Realizar un escaneo inicial sin despliegue y registrar los problemas detectados con su recurso, regla y severidad. El resultado servirá como referencia para la corrección.

### Tarea 3.1. Lanzar el primer análisis estático

Comprobar el formato de Terraform y ejecutar tfsec desde el directorio correcto.

- {% include step_label.html %} En **Git Bash**, confirma la carpeta de trabajo y ejecuta el formato automático. Esto organiza la presentación del HCL, pero **no corrige** configuraciones inseguras.

  ```bash
  pwd
  terraform fmt -check || terraform fmt
  terraform fmt -check
  ```

  > **Nota:** `terraform fmt` solo modifica espacios, sangrías y formato; no sustituye a un escáner de seguridad.
  {: .lab-note .info .compact}

  > **Salida esperada:** `terraform fmt -check` termina correctamente y los archivos HCL conservan los ejemplos inseguros.
  {: .lab-note .output .compact}


- {% include step_label.html %} Ejecuta el primer escaneo desde `lab10`. Si tfsec detecta hallazgos puede terminar con código distinto de cero; eso es el comportamiento esperado al identificar riesgos.

  ```bash
  tfsec .
  ```
  ```bash
  tfsec . --format sarif --out reports/antes.sarif --soft-fail
  ```

  > **Advertencia:** No interpretes un código de salida distinto de cero como fallo de instalación si la consola está mostrando hallazgos de seguridad. Si aparece `command not found`, vuelve a la instalación.
  {: .lab-note .warning .compact}

  > **Salida esperada:** Se presenta al menos una alerta relacionada con las reglas o permisos inseguros; su número y severidad dependen de la versión de tfsec.
  {: .lab-note .output .compact}


### Tarea 3.2. Analizar los riesgos antes de corregir

Distinguir recomendaciones de acceso a red, IAM público y controles de almacenamiento en lugar de contar únicamente incidencias.

- {% include step_label.html %} Revisa los hallazgos impresos por tfsec y localiza el recurso `google_compute_firewall.ssh`. Identifica el fragmento `source_ranges` y explica por qué permitir SSH desde todas las direcciones aumenta el riesgo.

  **Acciones o requisitos:**

  - Busca el nombre de recurso o ubicación en `main.tf`.
  - Lee el identificador de la regla y su severidad mostrada por tu versión.
  - Anota la recomendación concreta para restringir el origen; **no cambies aún el archivo**.

  > **Salida esperada:** Identificas el firewall inseguro y puedes describir cómo reducir el alcance de su regla de ingreso.
  {: .lab-note .output .compact}


- {% include step_label.html %} Localiza los hallazgos de `google_storage_bucket.demo` y `google_storage_bucket_iam_member.public_read`. No confundas **UBLA desactivado** con **permiso de lectura concedido a `allUsers`**: son riesgos relacionados, pero diferentes.

  **Acciones o requisitos:**

  - En el bucket, comprueba el valor de `uniform_bucket_level_access`.
  - En la asignación IAM, localiza `member = "allUsers"`.
  - Relaciona cada hallazgo con el recurso específico señalado por la herramienta; si alguna regla no aparece, indícalo sin inventar resultados.

  > **Salida esperada:** Distingues la falta de UBLA de la concesión pública efectiva de permisos de lectura.
  {: .lab-note .output .compact}


- {% include step_label.html %} Registra un pequeño resumen del análisis inicial antes de editar. Usa la siguiente tabla como guía y completa las celdas con lo observado en **tu** ejecución.

  **Acciones o requisitos:**

  - **Firewall:** escribe la regla o el texto del hallazgo y la severidad observada.
  - **Bucket:** anota si tfsec señaló la configuración UBLA y cómo recomienda corregirla.
  - **IAM público:** anota si identificó `allUsers` y qué recurso afecta.
  - Cuando una regla no aparezca, escribe **No detectada en esta versión** en vez de asumir que se ejecutó.

  > **Salida esperada:** Dispones de una comparación inicial de tres controles que podrás revisar tras aplicar las correcciones.
  {: .lab-note .output .compact}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## 🔎 Tarea 4. Corregir los riesgos y repetir el escaneo — 4 min

Endurecer el código sin cambiar su intención didáctica y comparar los hallazgos antes y después. No se utilizarán exclusiones para ocultar vulnerabilidades.

### Tarea 4.1. Corregir red y acceso al almacenamiento

Modificar los atributos inseguros y retirar la asignación pública sin introducir recursos innecesarios.

- {% include step_label.html %} En `main.tf`, identifica `google_compute_firewall.ssh` y cambia **solo** el rango de origen a la red utilizada por IAP TCP forwarding. Localiza `source_ranges` y utiliza el CIDR documentado por Google Cloud.

  **Acciones o requisitos:**

  - Abre el recurso `google_compute_firewall.ssh`.
  - Sustituye `0.0.0.0/0` por **`35.235.240.0/20`**, conservando el puerto TCP 22 y `target_tags = ["ssh"]`.
  - Consulta la documentación oficial de [IAP TCP forwarding](https://cloud.google.com/iap/docs/using-tcp-forwarding) si necesitas comprobar el rango.
  - Guarda el archivo y verifica que ya no aparezca `0.0.0.0/0` en la regla.

  > **Nota:** Esta modificación limita el origen de SSH al rango de IAP; no demuestra por sí sola que exista un túnel SSH ni que los permisos de IAP estén concedidos.
  {: .lab-note .info .compact}

  > **Salida esperada:** El firewall mantiene TCP 22 y el origen se reduce a `35.235.240.0/20`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Dentro de `google_storage_bucket.demo`, habilita UBLA y exige prevención de acceso público mediante atributos explícitos. Conserva el nombre del recurso `demo` para comparar el mismo objeto lógico.

  ```hcl
  resource "google_storage_bucket" "demo" {
    name                        = "lab10-demo-no-deploy-ejemplo"
    location                    = var.region
    uniform_bucket_level_access = true
    public_access_prevention    = "enforced"
    force_destroy               = false
  }
  ```

  > **Nota:** Este bloque **reemplaza** por completo al bloque anterior `google_storage_bucket.demo`; no lo agregues como segundo recurso.
  {: .lab-note .info .compact}

  > **Salida esperada:** El bucket tiene UBLA habilitado y la prevención de acceso público marcada como `enforced`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Elimina **por completo** el recurso `google_storage_bucket_iam_member.public_read` del final de `main.tf`. Verifica que no permanezcan referencias a `allUsers` ni recursos IAM públicos activos.

  **Acciones o requisitos:**

  - Selecciona desde `resource "google_storage_bucket_iam_member" "public_read" {` hasta la llave `}` que cierra el recurso.
  - Elimina el bloque completo; no basta comentar únicamente la línea `member = "allUsers"`.
  - Guarda el archivo y utiliza la búsqueda de VS Code para confirmar que no quedan referencias activas a `allUsers`.

  > **Salida esperada:** `main.tf` solo declara VPC, Firewall y bucket; no incluye permisos públicos de lectura.
  {: .lab-note .output .compact}


### Tarea 4.2. Ejecutar el segundo análisis y evaluar

Confirmar las correcciones y distinguir los hallazgos eliminados de recomendaciones adicionales.

- {% include step_label.html %} Ejecuta `terraform fmt` para estandarizar el archivo corregido y repite el escaneo con `tfsec`. Revisa el reporte real en lugar de asumir que no habrá ninguna alerta.

  ```bash
  terraform fmt
  terraform fmt -check
  tfsec .
  ```

  > **Salida esperada:** Los hallazgos sobre SSH abierto, UBLA deshabilitado y asignación pública desaparecen cuando tu versión detecta esos controles; otros hallazgos pueden permanecer.
  {: .lab-note .output .compact}


- {% include step_label.html %} Compara el primer y segundo escaneo y explica por qué corregir es preferible a ignorar reglas. Si tfsec sigue mostrando un hallazgo diferente, revisa su contenido antes de tomar una decisión.

  **Acciones o requisitos:**

  - **Antes:** había un origen SSH amplio, UBLA deshabilitado y un miembro público.
  - **Después:** el origen está limitado a IAP, UBLA está habilitado y no existe concesión `allUsers`.
  - No añadas comentarios `tfsec:ignore` solo para conseguir una salida sin alertas. Una excepción real requiere justificación y aprobación.

  > **Salida esperada:** Puedes relacionar los cambios HCL con los hallazgos resueltos, diferenciando correcciones reales de exclusiones.
  {: .lab-note .output .compact}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

## 🔎 Tarea 5. Exportar reportes SARIF y verificar resultados — 3 min

Conservar los reportes SARIF inicial y final generados sobre la misma carpeta en momentos distintos, compararlos y verificar su formato sin desplegar recursos.

### Tarea 5.1. Generar archivos SARIF y comprobar su contenido

Exportar resultados de los dos estados del código y verificar el formato de salida compatible con herramientas CI/CD.

- {% include step_label.html %} En **Git Bash**, exporta el **estado corregido** a `reports/despues.sarif`. La opción `--soft-fail` solo se utiliza para que la exportación no se detenga ante hallazgos que deban revisarse.

  ```bash
  tfsec . --format sarif --out reports/despues.sarif --soft-fail
  ls -lh reports/despues.sarif
  ```

  > **Salida esperada:** Existe el archivo `reports/despues.sarif`; contiene el análisis de la configuración ya corregida.
  {: .lab-note .output .compact}


- {% include step_label.html %} Compara los reportes **antes** y **después**. El primer SARIF se generó en la Tarea 3 antes de editar los archivos, de modo que no es necesario reconstruir ni reintroducir el código inseguro.

  ```bash
  ls -lh reports/antes.sarif reports/despues.sarif
  powershell.exe -NoProfile -Command '$a = Get-Content "reports/antes.sarif" -Raw | ConvertFrom-Json; $b = Get-Content "reports/despues.sarif" -Raw | ConvertFrom-Json; "Antes: $(@($a.runs[0].results).Count) hallazgos"; "Después: $(@($b.runs[0].results).Count) hallazgos"'
  ```

  > **Nota:** El conteo es un indicador, no la única prueba de seguridad: revisa también qué reglas se resolvieron. Los valores pueden variar con la versión de tfsec.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se imprimen las cantidades de resultados de ambos reportes sin modificar `main.tf` ni desplegar infraestructura.
  {: .lab-note .output .compact}


- {% include step_label.html %} Verifica que los reportes tengan el formato **SARIF 2.1.0** y comprueba que el proyecto no tenga recursos creados. Este archivo podrá utilizarse en una práctica posterior de integración CI/CD.

  ```bash
  powershell.exe -NoProfile -Command '$a = Get-Content "reports/antes.sarif" -Raw | ConvertFrom-Json; $b = Get-Content "reports/despues.sarif" -Raw | ConvertFrom-Json; "SARIF antes: $($a.version)"; "SARIF después: $($b.version)"'
  ls -la main.tf providers.tf variables.tf reports/*.sarif
  ```

  > **Nota:** No ejecutes `terraform apply` ni `terraform destroy`; el laboratorio no crea recursos. Si necesitas incorporar tfsec a GitHub Actions, conserva el reporte en `reports/despues.sarif` y evita subir archivos de estado o credenciales.
  {: .lab-note .info .compact}

  > **Salida esperada:** Los dos archivos existen y PowerShell informa su versión SARIF; la configuración sigue siendo exclusivamente de demostración.
  {: .lab-note .output .compact}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}
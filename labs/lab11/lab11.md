---
course_id: terrgcpint
lab_id: l011
tracking: true
position: 11
layout: lab
title: "Práctica 11: Generar doc automática con terraform-docs"
permalink: /lab11/lab11/
images_base: /labs/lab11/img
duration: "20 minutos"
objective:
  - "Generar y mantener documentación Markdown de un módulo Terraform para Google Cloud mediante terraform-docs y comprobar su actualización automática con pre-commit."
prerequisites:
  - "Prácticas 1 a 10 y carpeta terraform-gcp-intermediate en el Escritorio de Windows."
  - "VS Code con terminal Git Bash, Terraform y Git instalados."
  - "Acceso para instalar terraform-docs y Python con pip/pre-commit si aún no están disponibles."
introduction:
  - "Construirás un pequeño módulo Terraform para un bucket GCS, generarás tablas de variables y outputs en README.md, personalizarás su actualización mediante marcadores e integrarás un hook local de pre-commit. El laboratorio analiza archivos: no crea recursos ni requiere credenciales GCP."
slug: lab11
lab_number: 11
final_result: >
  Módulo gcs-bucket documentado con terraform-docs; README.md combina texto manual y secciones generadas, verifica actualizaciones y se regenera al ejecutar pre-commit.
notes:
  - "Todos los comandos locales se ejecutan desde Git Bash integrado en VS Code, dentro de terraform-gcp-intermediate/lab11, salvo cuando se indique VS Code o una terminal de Windows con permisos."
  - "El módulo gcs-bucket es demostrativo: no ejecutes terraform apply ni terraform destroy; la generación de documentación no requiere autenticación GCP ni genera costos."
  - "Ejecuta git init exclusivamente dentro de lab11; no inicialices ni modifiques el repositorio Jekyll del curso."
  - "La generación con terraform-docs conserva el texto manual situado fuera de BEGIN_TF_DOCS y END_TF_DOCS; revisa cualquier archivo actualizado por pre-commit antes de confirmar."
  - "El tiempo de 20 minutos supone que Terraform, Git, Python y terraform-docs ya están instalados; las instalaciones pueden extender la duración."
references:
  - text: "terraform-docs: sitio oficial"
    url: https://terraform-docs.io/
  - text: "terraform-docs: opciones de output"
    url: https://terraform-docs.io/user-guide/configuration/output/
  - text: "terraform-docs: releases oficiales"
    url: https://github.com/terraform-docs/terraform-docs/releases
  - text: "pre-commit: documentación oficial"
    url: https://pre-commit.com/
  - text: "Terraform: módulos"
    url: https://developer.hashicorp.com/terraform/language/modules
prev: /lab10/lab10/
next: /lab12/lab12/
---

---

## 🔎 Tarea 1. Preparar el módulo Terraform — 5 min

{% assign tracking_task_id = "t001" %}

Prepararás un directorio aislado para la práctica y crearás un módulo GCS sencillo cuyos metadatos serán la fuente de la documentación generada.

### Tarea 1.1. Crear lab11 y su estructura

Construir las rutas desde el directorio principal del curso y verificar que la terminal corresponda a Git Bash.

- {% include step_label.html id="s001" %} En **VS Code**, abre el proyecto principal `terraform-gcp-intermediate` en el Escritorio de Windows y selecciona **Git Bash** como terminal integrada.

  **Instrucciones:**

  - Abre **File > Open Folder...** y selecciona la carpeta del curso, no `lab10`.
  - Selecciona **Terminal > New Terminal**; en la lista de perfiles elige **Git Bash**.
  - Confirma que en **Explorer** se muestran las carpetas de las prácticas anteriores.

  > **Salida esperada:** El explorador presenta la raíz del curso y puedes introducir comandos Bash en la terminal.
  {: .lab-note .output .compact}


- {% include step_label.html id="s002" %} Desde **Git Bash**, crea explícitamente `lab11/modules/gcs-bucket` y entra en `lab11`. Comprueba la ubicación antes de continuar para evitar crear archivos en otra práctica.

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  cd "$DESKTOP/terraform-gcp-intermediate"
  mkdir -p lab11/modules/gcs-bucket
  cd lab11
  pwd
  ```

  > **Salida esperada:** La ruta de `pwd` termina en `/terraform-gcp-intermediate/lab11`, aunque el Escritorio esté redirigido a OneDrive.
  {: .lab-note .output .compact}


- {% include step_label.html id="s003" %} En **Git Bash**, crea los archivos del módulo y los de automatización de la raíz. El README y el archivo de configuración se crearán posteriormente, cuando veas cómo funciona la generación.

  ```bash
  touch modules/gcs-bucket/providers.tf modules/gcs-bucket/main.tf
  touch modules/gcs-bucket/variables.tf modules/gcs-bucket/outputs.tf
  touch .gitignore
  find modules -type f | sort
  ```

  > **Salida esperada:** Aparecen los cuatro archivos `.tf` en `modules/gcs-bucket/` y `.gitignore` en `lab11`.
  {: .lab-note .output .compact}


### Tarea 1.2. Definir el módulo GCS con entradas y salidas

Agregar código Terraform real pero destinado únicamente a lectura y documentación; no se desplegará infraestructura.

- {% include step_label.html id="s004" %} En **VS Code**, abre `lab11/modules/gcs-bucket/providers.tf` e indica el proveedor necesario. No incluyas un bloque `provider "google"` con credenciales: el módulo debe poder reutilizarse desde distintos proyectos.

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
  ```

  > **Salida esperada:** El archivo declara los requisitos de Terraform y del proveedor Google sin fijar credenciales.
  {: .lab-note .output .compact}


- {% include step_label.html id="s005" %} En `modules/gcs-bucket/variables.tf`, declara las cuatro entradas del módulo. Escribe las descripciones completas: `terraform-docs` utilizará exactamente estos textos para construir la tabla **Inputs**.

  ```hcl
  variable "bucket_name" {
    description = "Nombre único globalmente del bucket de Cloud Storage"
    type        = string
  }
  
  variable "location" {
    description = "Ubicación del bucket; por ejemplo US o us-central1"
    type        = string
    default     = "US"
  }
  
  variable "uniform_bucket_level_access" {
    description = "Gestionar accesos exclusivamente mediante IAM"
    type        = bool
    default     = true
  }
  
  variable "labels" {
    description = "Etiquetas para identificar los recursos del proyecto"
    type        = map(string)
    default     = {}
  }
  ```

  > **Salida esperada:** `bucket_name` es obligatorio; `location`, `uniform_bucket_level_access` y `labels` tienen valores predeterminados.
  {: .lab-note .output .compact}


- {% include step_label.html id="s006" %} En `modules/gcs-bucket/main.tf`, declara un bucket que consuma las variables anteriores. Aunque el recurso es válido para Terraform, en esta práctica **no ejecutes `apply`**; el objetivo es obtener documentación de su definición.

  ```hcl
  resource "google_storage_bucket" "this" {
    name                        = var.bucket_name
    location                    = var.location
    uniform_bucket_level_access = var.uniform_bucket_level_access
    labels                      = var.labels
  
    lifecycle_rule {
      condition { age = 30 }
      action { type = "Delete" }
    }
  }
  ```

  > **Nota:** La regla de ciclo de vida está declarada solo como ejemplo de documentación; ninguna política se aplica en Google Cloud.
  {: .lab-note .info .compact}

  > **Salida esperada:** El recurso `google_storage_bucket.this` usa las variables y aparece en el archivo `main.tf`.
  {: .lab-note .output .compact}


- {% include step_label.html id="s007" %} En `modules/gcs-bucket/outputs.tf`, crea dos salidas documentadas. Ambas leerán propiedades del recurso una vez que un consumidor lo despliegue, pero aquí solo se incluirán en la tabla **Outputs**.

  ```hcl
  output "bucket_name" {
    description = "Nombre del bucket administrado por el módulo"
    value       = google_storage_bucket.this.name
  }
  
  output "bucket_self_link" {
    description = "Identificador self_link del bucket administrado"
    value       = google_storage_bucket.this.self_link
  }
  ```

  > **Salida esperada:** El módulo dispone de cuatro variables y dos outputs con descripciones que podrán extraerse automáticamente.
  {: .lab-note .output .compact}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## 📄 Tarea 2. Instalar y generar documentación — 4 min

{% assign tracking_task_id = "t002" %}

Verificarás la disponibilidad de terraform-docs y crearás la primera documentación en Markdown a partir de los archivos HCL.

### Tarea 2.1. Comprobar instalación de terraform-docs

Confirmar la herramienta en Windows y ofrecer una alternativa cuando no esté previamente instalada.

- {% include step_label.html id="s008" %} En **Git Bash** desde `lab11`, comprueba la instalación de `terraform-docs` y de Terraform; evita reinstalar aplicaciones que ya funcionen.

  ```bash
  terraform version
  if command -v terraform-docs >/dev/null 2>&1; then
    terraform-docs --version
  else
    echo "terraform-docs no se encuentra en PATH"
  fi
  ```

  > **Salida esperada:** Terraform muestra su versión; `terraform-docs` informa su versión o aparece el aviso de que falta instalarlo.
  {: .lab-note .output .compact}


- {% include step_label.html id="s009" %} Si no tienes `terraform-docs`, utiliza el administrador de paquetes disponible o instala el ejecutable **Windows amd64** de la página oficial. Después abre una terminal Git Bash nueva para actualizar `PATH`.

  **Instrucciones:**

  - **Opción Chocolatey (administrador de Windows):** ejecuta `choco install terraform-docs -y` solo si ya tienes Chocolatey.
  - **Opción Scoop (terminal de Windows):** ejecuta `scoop install terraform-docs` solo si ya tienes Scoop configurado.
  - **Opción manual:** abre [GitHub Releases de terraform-docs](https://github.com/terraform-docs/terraform-docs/releases), descarga el archivo de Windows para tu arquitectura, extrae `terraform-docs.exe` y sitúalo en una carpeta incluida en `PATH`.
  - Comprueba al final en **Git Bash** el siguiente comando; si no funciona, no continúes hasta corregir `PATH`.

    ```bash
    terraform-docs --version
    ```

  > **Salida esperada:** `terraform-docs --version` muestra una versión y el ejecutable puede invocarse desde Git Bash.
  {: .lab-note .output .compact}


### Tarea 2.2. Generar y leer el README inicial

Mostrar primero la salida de la herramienta sin configuración adicional para identificar las secciones que produce.

- {% include step_label.html id="s010" %} Ubícate en el módulo y pide a `terraform-docs` que muestre una tabla Markdown en la terminal. Esta primera ejecución permite confirmar que detecta el HCL sin crear aún un archivo.

  ```bash
  cd modules/gcs-bucket
  terraform-docs markdown table .
  ```

  > **Salida esperada:** Se visualizan secciones como Requirements, Providers, Resources, Inputs y Outputs; aparecen las cuatro entradas y dos salidas.
  {: .lab-note .output .compact}


- {% include step_label.html id="s011" %} Genera el primer `README.md` redirigiendo la salida de `terraform-docs`. Este método sobrescribe el archivo, por lo que solo lo usamos **en esta primera generación**, cuando todavía no contiene texto manual.

  ```bash
  terraform-docs markdown table . > README.md
  ls -lh README.md
  ```

  > **Salida esperada:** Se crea `modules/gcs-bucket/README.md` con una tabla de variables y otra de outputs.
  {: .lab-note .output .compact}


- {% include step_label.html id="s012" %} Desde **VS Code**, abre `modules/gcs-bucket/README.md` y revisa la vista previa de Markdown para comprobar el resultado antes de personalizarlo.

  **Instrucciones:**

  - En **Explorer**, selecciona `modules > gcs-bucket > README.md`.
  - Abre **Open Preview to the Side** desde el menú contextual o pulsa **Ctrl+K, V**.
  - Busca `bucket_name` en **Inputs**, y `bucket_self_link` en **Outputs**.
  - Distingue las descripciones escritas manualmente en HCL de las tablas generadas por la herramienta.

  > **Salida esperada:** La vista previa muestra tablas Markdown legibles con requisitos, recursos, variables y salidas.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## 📝 Tarea 3. Personalizar y actualizar el README — 4 min

{% assign tracking_task_id = "t003" %}

Configurarás una región administrada por terraform-docs dentro de un README con texto manual y demostrarás que los cambios en el código actualizan la documentación.

### Tarea 3.1. Crear configuración para actualización controlada

Usar el modo inject y marcadores delimitados para preservar el encabezado y las instrucciones redactadas por el participante.

- {% include step_label.html id="s013" %} Regresa desde el módulo a la raíz `lab11` y crea `.terraform-docs.yml` dentro de `modules/gcs-bucket`. Este archivo definirá el formato y dónde se insertará el contenido generado.

  ```bash
  cd ../..
  touch modules/gcs-bucket/.terraform-docs.yml
  pwd
  ```

  > **Salida esperada:** La terminal vuelve a `lab11` y existe `modules/gcs-bucket/.terraform-docs.yml`.
  {: .lab-note .output .compact}


- {% include step_label.html id="s014" %} En `modules/gcs-bucket/.terraform-docs.yml`, configura la generación Markdown con modo `inject`. Introduce los marcadores **BEGIN_TF_DOCS** y **END_TF_DOCS** para delimitar el contenido automático.

  {%raw%}
  ```yaml
  formatter: markdown table
  
  output:
    file: README.md
    mode: inject
    template: |-
      <!-- BEGIN_TF_DOCS -->
      {{ .Content }}
      <!-- END_TF_DOCS -->
  ```
  {%endraw%}

  > **Nota:** Los marcadores se colocarán en `README.md`; el modo `inject` conserva el contenido fuera de la región marcada. El formato YAML debe mantenerse con espacios y con {%raw%}`{{ .Content }}`{%endraw%} dentro de `template`.
  {: .lab-note .info .compact}

  > **Salida esperada:** La configuración define `markdown table` y el archivo de salida `README.md` en modo `inject`.
  {: .lab-note .output .compact}


- {% include step_label.html id="s015" %} Reemplaza el contenido previo de `modules/gcs-bucket/README.md` con un encabezado manual, una explicación del uso y los marcadores. Solo el espacio entre esos marcadores podrá actualizarlo automáticamente terraform-docs.

  ```markdown
  # Módulo GCS Bucket
  
  Este módulo describe un bucket de Cloud Storage con acceso uniforme.
  
  ## Uso previsto
  
  Consúmelo desde un root module y define `bucket_name` antes de aplicar.
  Esta práctica solo genera documentación; no despliega recursos.
  
  <!-- BEGIN_TF_DOCS -->
  <!-- END_TF_DOCS -->
  
  ## Nota del equipo
  
  Revisar nombres únicos de buckets y políticas IAM antes de producción.
  ```

  > **Salida esperada:** El README incluye título, descripción, marcadores y una nota manual fuera de la zona generada.
  {: .lab-note .output .compact}


### Tarea 3.2. Verificar sincronización de variables y documentación

Generar la región automática y comprobar que una descripción HCL modificada se actualiza sin borrar el contenido del equipo.

- {% include step_label.html id="s016" %} Desde la raíz `lab11`, ejecuta `terraform-docs` con su archivo de configuración explícito. Este comando llenará la región marcada del README sin sustituir las secciones manuales.

  ```bash
  terraform-docs --config modules/gcs-bucket/.terraform-docs.yml modules/gcs-bucket
  sed -n '1,75p' modules/gcs-bucket/README.md
  ```

  > **Salida esperada:** Las tablas aparecen entre `BEGIN_TF_DOCS` y `END_TF_DOCS`; se preservan `Uso previsto` y `Nota del equipo`.
  {: .lab-note .output .compact}


- {% include step_label.html id="s017" %} En `modules/gcs-bucket/variables.tf`, cambia solo la descripción de `labels` para demostrar que el README se obtiene del código y no de una tabla editada manualmente.

  **Instrucciones:**

  - Localiza `variable "labels"` al final de `variables.tf`.
  - Reemplaza su línea `description` por `description = "Etiquetas de inventario para clasificar el bucket"`.
  - Guarda **Ctrl+S** y no edites a mano la tabla **Inputs** del README.

  > **Salida esperada:** El código HCL contiene la nueva descripción, mientras que el README conserva temporalmente el texto anterior.
  {: .lab-note .output .compact}


- {% include step_label.html id="s018" %} Vuelve a ejecutar la generación y busca la nueva descripción dentro del README. Confirma también que el encabezado, la explicación y la nota manual permanecen intactos.

  ```bash
  terraform-docs --config modules/gcs-bucket/.terraform-docs.yml modules/gcs-bucket
  grep -n "Etiquetas de inventario" modules/gcs-bucket/README.md
  grep -n "Nota del equipo" modules/gcs-bucket/README.md
  ```

  > **Salida esperada:** `grep` localiza la descripción actualizada y la nota del equipo: la documentación está sincronizada sin perder contenido manual.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## ⚙️ Tarea 4. Automatizar con pre-commit — 5 min

{% assign tracking_task_id = "t004" %}

Prepararás un repositorio Git independiente y configurarás un hook local que regenere la documentación del módulo antes de confirmar los cambios.

### Tarea 4.1. Verificar Python e inicializar Git

Preparar el ejecutor de pre-commit sin afectar al repositorio Jekyll ni a otros laboratorios.

- {% include step_label.html id="s019" %} En Git Bash, comprueba `git` y Python. **pre-commit** necesita Python; elige el comando disponible en tu VM antes de instalar dependencias.

  ```bash
  git --version
  if command -v py >/dev/null 2>&1; then
    py --version
  elif command -v python >/dev/null 2>&1; then
    python --version
  else
    echo "Instala Python para Windows antes de continuar"
  fi
  ```

  > **Salida esperada:** Se muestran las versiones de Git y Python; si falta Python, el paso indica claramente el requisito pendiente.
  {: .lab-note .output .compact}


- {% include step_label.html id="s020" %} Instala `pre-commit` en el entorno Python de Windows cuando todavía no esté disponible. La opción `--user` evita alterar paquetes del sistema; el comando de verificación permite saber si la instalación terminó.

  ```bash
  if command -v py >/dev/null 2>&1; then
    py -m pip install --user pre-commit
    py -m pre_commit --version
  else
    python -m pip install --user pre-commit
    python -m pre_commit --version
  fi
  ```

  > **Nota:** Si Python no dispone de `pip`, solicita su habilitación en la VM. No instales paquetes con privilegios de administrador sin autorización.
  {: .lab-note .info .compact}

  > **Salida esperada:** La salida muestra la versión de `pre-commit`; no es necesario que su ejecutable independiente esté en `PATH` porque lo invocaremos como módulo Python.
  {: .lab-note .output .compact}


- {% include step_label.html id="s021" %} Inicializa Git **dentro de `lab11`** y añade un `.gitignore` para evitar que se versionen estados Terraform o archivos de caché. No ejecutes `git init` en la carpeta que reúne todos los laboratorios.

  ```bash
  git init
  ```
  ```bash
  cat > .gitignore <<'EOF'
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  crash.log
  EOF
  ```
  ```bash
  git status --short
  ```

  > **Salida esperada:** `lab11/.git` se ha creado y `.gitignore` contiene exclusiones para estado y caché de Terraform.
  {: .lab-note .output .compact}


### Tarea 4.2. Configurar e instalar el hook documental

Hacer que Git invoque el mismo comando de generación usado manualmente y documentar cómo funciona cuando modifica archivos.

- {% include step_label.html id="s022" %} Crea `.pre-commit-config.yaml` en la raíz `lab11` y registra un hook local de sistema. El hook no requiere clonar repositorios externos ni ejecutar Terraform contra GCP.

  ```yaml
  repos:
    - repo: local
      hooks:
        - id: terraform-docs-gcs-bucket
          name: Actualizar README del modulo GCS
          entry: terraform-docs --config modules/gcs-bucket/.terraform-docs.yml modules/gcs-bucket
          language: system
          pass_filenames: false
          always_run: true
          stages: [pre-commit]
  ```

  > **Salida esperada:** El hook está declarado con `stages: [pre-commit]`, utiliza la herramienta local y no depende de archivos concretos de la selección Git.
  {: .lab-note .output .compact}


- {% include step_label.html id="s023" %} Instala el hook en `.git/hooks` mediante el Python disponible y comprueba que `pre-commit` detecta la configuración. La instalación se realiza dentro del repositorio `lab11`.

  ```bash
  if command -v py >/dev/null 2>&1; then
    py -m pre_commit install
  else
    python -m pre_commit install
  fi
  ls -l .git/hooks/pre-commit
  ```

  > **Salida esperada:** Git dispone de un archivo `.git/hooks/pre-commit` ejecutable que llamará a la automatización.
  {: .lab-note .output .compact}


- {% include step_label.html id="s024" %} Prepara los archivos rastreables e identifica qué se confirmará. Agrega solo el módulo, el README, `.gitignore` y la configuración del hook; revisa el listado antes de cualquier commit.

  ```bash
  git add .gitignore .pre-commit-config.yaml modules/gcs-bucket
  git diff --cached --name-only
  ```

  > **Salida esperada:** Se enumeran archivos `.tf`, `.terraform-docs.yml`, `README.md` y archivos de Git; no hay `tfstate` ni credenciales.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

## ✅ Tarea 5. Validar automatización y resultados — 2 min

{% assign tracking_task_id = "t005" %}

Demostrarás que el hook ejecuta terraform-docs y deja el README listo para versionarse, incluyendo el comportamiento esperado cuando un hook modifica archivos.

### Tarea 5.1. Ejecutar hook en archivos rastreados

Lanzar pre-commit de manera explícita antes de crear el commit y comprobar que los cambios son reproducibles.

- {% include step_label.html id="s025" %} Ejecuta todos los hooks desde Git Bash, usando el módulo Python que acabas de instalar. El hook puede modificar el README si detecta cambios en variables u outputs; si ocurre, prepara nuevamente el archivo.

  ```bash
  if command -v py >/dev/null 2>&1; then
    py -m pre_commit run --all-files
  else
    python -m pre_commit run --all-files
  fi
  ```

  > **Nota:** El primer intento puede señalar que `README.md` fue modificado por el hook. En ese caso revisa `git diff`, ejecuta `git add modules/gcs-bucket/README.md` y repite el comando.
  {: .lab-note .info .compact}

  > **Salida esperada:** El hook finaliza con `Passed`, o modifica README y permite repetir el análisis tras agregar el archivo actualizado.
  {: .lab-note .output .compact}


- {% include step_label.html id="s026" %} Confirma que el README conserva el contenido del equipo y que la descripción nueva de `labels` aparece en la región automática. Después, vuelve a agregar el README si el hook lo cambió.

  ```bash
  grep -n "Etiquetas de inventario" modules/gcs-bucket/README.md
  grep -n "Nota del equipo" modules/gcs-bucket/README.md
  ```
  ```bash
  git add modules/gcs-bucket/README.md
  git status --short
  ```

  > **Salida esperada:** Ambas frases aparecen; `git status` muestra los archivos preparados, sin archivos `.tfstate`.
  {: .lab-note .output .compact}


### Tarea 5.2. Crear commit y verificar salida documental

Comprobar la ejecución automática al confirmar cambios y dejar evidencia local de que la documentación está actualizada.

- {% include step_label.html id="s027" %} Configura el nombre y correo **solo para este repositorio** si Git aún no los tiene, y crea un commit de prueba. Si el hook genera cambios y bloquea el commit, inspecciona, agrega el README y repite el comando.

  **Instrucciones:**

  - Si `git config user.name` o `git config user.email` no muestran valor, establece los datos de tu cuenta únicamente dentro de `lab11` con `git config user.name "Tu Nombre"` y `git config user.email "tu-correo@example.com"`.
  - En Git Bash ejecuta los comandos siguientes y observa la salida del hook antes del mensaje final de Git.
  - Si aparece `files were modified by this hook`, utiliza `git add modules/gcs-bucket/README.md` y ejecuta el commit otra vez.

  ```bash
  git add .gitignore .pre-commit-config.yaml modules/gcs-bucket
  git commit -m "Documentar modulo GCS con terraform-docs"
  ```

  > **Salida esperada:** El hook se ejecuta y, cuando ya no hay modificaciones pendientes, Git crea el commit local de documentación.
  {: .lab-note .output .compact}


- {% include step_label.html id="s028" %} Comprueba el commit y la estructura definitiva. No se ha iniciado sesión en Google Cloud ni creado buckets: el resultado es un módulo **documentado y automatizado**, listo para utilizarse en un proyecto posterior.

  ```bash
  git log -1 --oneline
  git status --short
  find modules/gcs-bucket -maxdepth 1 -type f | sort
  ```

  > **Salida esperada:** Aparece un commit reciente, `git status --short` queda vacío y están presentes README, `.terraform-docs.yml` y los cuatro archivos HCL.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}
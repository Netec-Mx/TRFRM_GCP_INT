---
layout: lab
title: "Práctica 8: Usar secretos desde Google Cloud Secret Manager y validar que no se expongan"
permalink: /lab8/lab8/
images_base: /labs/lab8/img
duration: "45 minutos"
objective:
  - "Crear y consumir un secreto desde Google Cloud Secret Manager sin incluir su valor en archivos Terraform ni en el estado, y comprobar una conexión a Cloud SQL MySQL mediante Cloud SQL Auth Proxy desde Compute Engine."
prerequisites:
  - "Prácticas 1 a 7 completadas; carpeta terraform-gcp-intermediate en el Escritorio de Windows."
  - "VS Code, terminal Git Bash, Terraform >= 1.5, gcloud CLI y Python 3 instalados; proyecto GCP y ADC configurados."
  - "Facturación habilitada, cuota Cloud SQL y permisos para Compute Engine, Cloud SQL Admin, Secret Manager, cuentas de servicio e IAM; para SSH vía IAP se necesita acceso IAP."
  - "APIs compute.googleapis.com, sqladmin.googleapis.com, secretmanager.googleapis.com e iam.googleapis.com habilitables en el proyecto."
introduction:
  - "Crearás un secreto sin imprimir su valor, una cuenta de servicio con privilegios limitados, una VPC, una instancia MySQL y una VM. Un programa local asignará una misma contraseña a Cloud SQL y Secret Manager fuera del estado Terraform. En la VM, Cloud SQL Auth Proxy establecerá conectividad y un programa obtendrá el secreto mediante el token de la VM, ejecutará una consulta y borrará credenciales temporales. Finalmente, eliminarás únicamente los recursos lab8."
slug: lab8
lab_number: 8
final_result: >
  Habrás desplegado infraestructura Terraform y validado que una VM autorizada obtiene un secreto en tiempo de ejecución,
  se conecta a Cloud SQL mediante Auth Proxy, ejecuta SQL y no imprime ni almacena la contraseña en el estado Terraform.
  También comprobarás la destrucción y eliminación del secreto creado fuera de Terraform.
notes:
  - "Todos los comandos locales se ejecutan en Git Bash de VS Code dentro de terraform-gcp-intermediate/lab8, salvo indicación explícita de ejecutar dentro de la VM o en Google Cloud Console."
  - "La duración de 70 minutos supone herramientas, APIs y permisos listos. Cloud SQL tiene costo y sus tiempos de creación/destrucción pueden extender la sesión."
  - "Esta práctica usa estado local independiente y no reutiliza el bucket ni el backend GCS de los laboratorios anteriores. No publiques terraform.tfstate ni credenciales."
  - "El ID del secreto y los identificadores de recursos no son secretos; su valor sí lo es. Evita mostrar la contraseña, activar set -x o habilitar logs HTTP detallados."
  - "Los pasos de construcción dirigida requieren completar el código antes de ejecutar terraform validate. Si faltan permisos IAM, solicita al instructor habilitarlos, no sustituyas por permisos globales excesivos."
  - "El secreto se crea con gcloud y su versión con Python; no es administrado por Terraform. Se elimina de forma explícita al limpiar el laboratorio."
references:
  - text: "Google Cloud Secret Manager"
    url: https://cloud.google.com/secret-manager/docs
  - text: "Google Cloud: añadir una versión de secreto"
    url: https://cloud.google.com/secret-manager/docs/add-secret-version
  - text: "Cloud SQL Auth Proxy para MySQL"
    url: https://cloud.google.com/sql/docs/mysql/connect-auth-proxy
  - text: "SQL Admin API: usuarios"
    url: https://cloud.google.com/sql/docs/mysql/admin-api/rest/v1beta4/users
  - text: "Terraform: variables y estado"
    url: https://developer.hashicorp.com/terraform/language/values/variables
  - text: "Google Cloud: acceso a Secret Manager con IAM"
    url: https://cloud.google.com/secret-manager/docs/access-control
prev: /lab7/lab7/
next: /lab9/lab9/
---

---

## 🔎 Tarea 1. Preparar el directorio y verificar el entorno GCP — 7 min

Crearás `lab8` dentro del directorio central del curso y verificarás Terraform, Google Cloud CLI, Python, el proyecto activo y las APIs necesarias antes de crear recursos facturables.


### Tarea 1.1. Crear lab8 y los archivos de trabajo

Abrir el curso en VS Code y preparar tanto la configuración Terraform como los programas auxiliares de verificación, sin copiar archivos de otras prácticas.


- {% include step_label.html %} En **VS Code**, abre la carpeta principal del curso mediante **File > Open Folder...** y selecciona **Escritorio > terraform-gcp-intermediate**.

  1. Comprueba que el Explorador muestre `lab6` y `lab7` como carpetas vecinas.
  2. Selecciona **Terminal > New Terminal**.
  3. Cambia el perfil de terminal a **Git Bash** si VS Code abrió PowerShell.
  4. Mantén abierta esta carpeta principal; no trabajes dentro de `lab7`.

  > **Salida esperada:** El Explorador muestra la raíz del curso y la terminal integrada utiliza Git Bash.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **Git Bash**, crea la carpeta `lab8` dentro de `terraform-gcp-intermediate` y prepara sus archivos.

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab8/env"
  cd "$DESKTOP/terraform-gcp-intermediate/lab8"
  touch providers.tf variables.tf main.tf outputs.tf env/dev.tfvars bootstrap_secret.py .gitignore
  pwd
  ```

  > **Nota:** Si el Escritorio está redirigido a OneDrive, abre la carpeta correcta desde VS Code y ejecuta `mkdir -p lab8/env && cd lab8` desde esa ubicación.
  {: .lab-note .info .compact}

  > **Salida esperada:** La ruta termina en `/terraform-gcp-intermediate/lab8` y contiene `env/`, los cuatro archivos `.tf` y `bootstrap_secret.py`, aunque Windows redirija el Escritorio a OneDrive.
  {: .lab-note .output .compact}


- {% include step_label.html %} Crea un `.gitignore` básico para evitar publicar estado Terraform y artefactos de autenticación.

  ```bash
  cat > .gitignore <<'EOF'
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  *.tfvars.bak
  *.log
  *.json
  __pycache__/
  EOF
  cat .gitignore
  ```

  > **Nota:** Los `.tfvars` no incluirán contraseñas; nunca agregues una contraseña a ellos.
  {: .lab-note .info .compact}

  > **Salida esperada:** El archivo excluye estados, planes y archivos JSON que podrían contener información sensible.
  {: .lab-note .output .compact}


### Tarea 1.2. Validar herramientas, proyecto y permisos

Comprobar el contexto de gcloud y el estado de APIs antes de crear recursos que generan costos.


- {% include step_label.html %} Verifica en **Git Bash** las versiones instaladas de Terraform, Google Cloud CLI y Python 3.

  ```bash
  terraform version
  gcloud version | head -n 3
  python --version
  ```

  > **Nota:** Si `python` no existe, comprueba `py -3 --version` y usa `py -3` en los pasos de Python.
  {: .lab-note .info .compact}

  > **Salida esperada:** Los tres comandos informan una versión instalada; Python debe ser 3.x.
  {: .lab-note .output .compact}


- {% include step_label.html %} Consulta el usuario y el proyecto activos; guarda el identificador que se usará en los siguientes comandos.

  ```bash
  PROJECT_ID="$(gcloud config get-value project)"
  echo "Proyecto GCP: $PROJECT_ID"
  gcloud auth list --filter="status:ACTIVE" --format="value(account)"
  gcloud auth application-default print-access-token >/dev/null && echo "ADC disponible"
  ```

  > **Nota:** Si ADC falla, ejecuta `gcloud auth application-default login` con tu cuenta del curso y repite esta comprobación.
  {: .lab-note .info .compact}

  > **Salida esperada:** La cuenta y el proyecto corresponden al curso y la prueba de ADC no imprime el token.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba que las cuatro APIs requeridas estén habilitadas; habilita las ausentes solo si tienes autorización.

  ```bash
  gcloud services list --enabled --project="$PROJECT_ID" \
    --format="value(config.name)" | grep -E \
    "^(compute|sqladmin|secretmanager|iam)\.googleapis\.com$"
  # Si falta una API, habilita solo las necesarias con permiso del instructor:
  # gcloud services enable compute.googleapis.com sqladmin.googleapis.com \
  #   secretmanager.googleapis.com iam.googleapis.com --project="$PROJECT_ID"
  ```

  > **Nota:** Cloud SQL e IAM pueden estar restringidos por políticas del curso; si recibes PERMISSION_DENIED, solicita el permiso correspondiente.
  {: .lab-note .info .compact}

  > **Salida esperada:** La lista incluye `compute.googleapis.com`, `sqladmin.googleapis.com`, `secretmanager.googleapis.com` e `iam.googleapis.com`. La cuota y facturación deben confirmarse en GCP antes del despliegue.
  {: .lab-note .output .compact}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}


---

## ☁️ Tarea 2. Crear Secret Manager y planear el acceso mínimo — 13 min

Crearás el secreto `lab8-db-password` sin incluir contraseñas en archivos Terraform, revisarás su configuración y definirás los permisos que necesitará la Service Account de la VM.


### Tarea 2.1. Crear el secreto y comprobar sus metadatos

Crear un secreto del laboratorio fuera del estado Terraform. El contenido se cargará después de desplegar Cloud SQL.


- {% include step_label.html %} Define los identificadores del laboratorio en **Git Bash** y confirma el nombre del secreto.

  ```bash
  PROJECT_ID="$(gcloud config get-value project)"
  SECRET_ID="lab8-db-password"
  SQL_INSTANCE="lab8-mysql"
  REGION="us-central1"
  echo "Proyecto: $PROJECT_ID / Secreto: $SECRET_ID / SQL: $SQL_INSTANCE"
  ```

  > **Salida esperada:** Se muestran únicamente identificadores de recursos, no contraseñas.
  {: .lab-note .output .compact}


- {% include step_label.html %} Crea el secreto en Secret Manager con replicación administrada por Google Cloud.

  ```bash
  gcloud secrets create "$SECRET_ID" \
    --replication-policy="automatic" --project="$PROJECT_ID"
  ```

  > **Nota:** Si el recurso ya existe por una ejecución anterior, verifica con `gcloud secrets describe` y no lo sobrescribas a ciegas.
  {: .lab-note .info .compact}

  > **Salida esperada:** GCP confirma la creación del secreto `lab8-db-password`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba los metadatos del secreto sin recuperar ninguna versión de su contenido.

  ```bash
  gcloud secrets describe "$SECRET_ID" --project="$PROJECT_ID" \
    --format="yaml(name,replication)"
  ```

  > **Salida esperada:** Se muestran el nombre completo del recurso y la política de replicación; no aparece una contraseña.
  {: .lab-note .output .compact}


- {% include step_label.html %} Verifica que el secreto todavía no tiene versiones habilitadas: el contenido se generará después de crear la base de datos.

  ```bash
  gcloud secrets versions list "$SECRET_ID" --project="$PROJECT_ID" \
    --format="table(name,state,createTime)"
  ```

  > **Salida esperada:** La lista de versiones está vacía antes de ejecutar `bootstrap_secret.py`.
  {: .lab-note .output .compact}


### Tarea 2.2. Comprender la identidad de ejecución

Definir el principio de mínimo privilegio que se implementará en Terraform para la cuenta de servicio de la VM.


- {% include step_label.html %} Consulta el rol **Secret Manager Secret Accessor** en la documentación y distingue permiso de lectura de administración.

  1. Abre [Control de acceso de Secret Manager](https://cloud.google.com/secret-manager/docs/access-control).
  2. Localiza `roles/secretmanager.secretAccessor`.
  3. Identifica que la cuenta de servicio necesita acceso **al secreto específico**, no a todos los secretos del proyecto.
  4. No concedas `roles/editor` ni `roles/owner` a la VM.

  > **Salida esperada:** La identidad de ejecución tendrá lectura del secreto del laboratorio, no administración global.
  {: .lab-note .output .compact}


- {% include step_label.html %} Determina cómo Cloud SQL Auth Proxy autentica a la VM mediante la Service Account adjunta.

  1. Abre [Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/mysql/connect-auth-proxy).
  2. Identifica el rol `roles/cloudsql.client`.
  3. Confirma que la VM requerirá un token de su Service Account y el permiso para acceder a la instancia.
  4. No crees ni descargues una clave JSON de Service Account.

  > **Salida esperada:** Queda claro qué permisos necesita la VM para Secret Manager y Cloud SQL.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba que el usuario del curso puede crear cuentas de servicio y asignarles roles antes de ejecutar Terraform.

  ```bash
  gcloud iam service-accounts list --project="$PROJECT_ID" \
    --format="table(email,displayName)"
  ```

  > **Nota:** Poder listar cuentas no garantiza poder crearlas: si IAM está restringido, solicita al instructor permisos de creación, Service Account User y cambios IAM.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se lista la información de cuentas de servicio; crear y adjuntar una nueva requerirá permisos IAM adicionales.
  {: .lab-note .output .compact}


### Tarea 2.3. Preparar variables y verificar protección del secreto

Definir valores no sensibles en Terraform y comprender qué datos sí se podrán compartir.


- {% include step_label.html %} Crea `env/dev.tfvars` con el identificador de proyecto, la región y los nombres del laboratorio.

  ```hcl
  project_id    = "TU_PROJECT_ID"
  region        = "us-central1"
  zone          = "us-central1-a"
  secret_id     = "lab8-db-password"
  instance_name = "lab8-mysql"
  db_name       = "appdb"
  db_user       = "app_user"
  ```

  > **Salida esperada:** El archivo contiene solo identificadores no secretos. Sustituye `TU_PROJECT_ID` por el resultado de `gcloud config get-value project`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba con **Git Bash** que el archivo `.tfvars` no contiene credenciales y que el nombre del secreto coincide.

  ```bash
  grep -E '^(project_id|secret_id|instance_name|db_name|db_user)' env/dev.tfvars
  # Revisa el contenido completo desde VS Code y confirma que NO hay password ni token.
  ```

  > **Salida esperada:** El archivo incluye nombres de recursos y no incluye campos `password`, tokens ni credenciales.
  {: .lab-note .output .compact}


- {% include step_label.html %} Identifica explícitamente el límite de seguridad: marcar una variable como `sensitive` no elimina su valor del estado Terraform.

  - **Configuración:** no escribiremos contraseñas en `main.tf`, `variables.tf` o `.tfvars`.
  - **Estado:** Terraform solo administrará recursos sin valores de contraseña.
  - **Tiempo de ejecución:** el programa de la VM leerá el secreto desde Secret Manager usando identidad de la VM.
  - **Terminal:** nunca se ejecutará `echo` del valor ni se pasará la contraseña como argumento de `mysql`.

  > **Salida esperada:** Las cuatro reglas de no exposición quedan definidas antes de crear la base de datos.
  {: .lab-note .output .compact}


{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}


---

## 🧩 Tarea 3. Desplegar Cloud SQL, cuenta de servicio y VM con Terraform — 22 min

Construirás una VPC, una instancia Cloud SQL, una base MySQL y una VM con identidad limitada. Después generarás la contraseña fuera de Terraform y la cargarás en Secret Manager sin mostrarla.


### Tarea 3.1. Configurar Provider, variables y red

Crear la configuración HCL con referencias explícitas y recursos cuyo estado no contenga valores secretos.


- {% include step_label.html %} En `providers.tf`, declara el Google Provider y la versión requerida de Terraform.

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

  > **Salida esperada:** El bloque se encuentra en `lab8/providers.tf` y el Provider utilizará las variables del laboratorio.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `variables.tf`, declara los identificadores y propiedades de la infraestructura, sin ninguna variable `db_password`.

  ```hcl
  variable "project_id" {
    type = string
  }
  variable "region" {
    type    = string
    default = "us-central1"
  }
  variable "zone" {
    type    = string
    default = "us-central1-a"
  }
  variable "secret_id" {
    type = string
  }
  variable "instance_name" {
    type = string
  }
  variable "db_name" {
    type = string
  }
  variable "db_user" {
    type = string
  }
  ```

  > **Salida esperada:** `variables.tf` no contiene variables ni valores de contraseñas.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `main.tf`, crea una red VPC y una subred propias de `lab8`, diferentes de las prácticas anteriores.

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = "lab8-vpc"
    auto_create_subnetworks = false
  }

  resource "google_compute_subnetwork" "subnet" {
    name          = "lab8-subnet"
    ip_cidr_range = "10.80.1.0/24"
    region        = var.region
    network       = google_compute_network.vpc.id
  }

  resource "google_compute_firewall" "ssh_iap" {
    name          = "lab8-allow-ssh-iap"
    network       = google_compute_network.vpc.name
    direction     = "INGRESS"
    source_ranges = ["35.235.240.0/20"]
    target_tags   = ["lab8-ssh"]
    allow {
      protocol = "tcp"
      ports    = ["22"]
    }
  }
  ```

  > **Nota:** La regla de Firewall por sí sola no otorga acceso IAP: la identidad del usuario necesita los permisos correspondientes.
  {: .lab-note .info .compact}

  > **Salida esperada:** El archivo define una VPC personalizada, una subred y una regla SSH limitada al origen IAP.
  {: .lab-note .output .compact}


- {% include step_label.html %} En el mismo archivo `main.tf`, agrega el bloque `google_service_account.vm`.

  **Requisitos de construcción:**

  - Coloca el nuevo bloque **después** de `google_compute_firewall.ssh_iap`.
  - Usa el recurso `google_service_account` con nombre local `vm`.
  - Configura `account_id = "lab8-vm-sa"` y un `display_name` descriptivo.
  - Conserva una sola cuenta de servicio para la VM; no generes credenciales JSON.
  - Consulta la [documentación de Service Account](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/service_account).

  > **Salida esperada:** `main.tf` declara una Service Account con ID `lab8-vm-sa` utilizada después por los permisos y la VM.
  {: .lab-note .output .compact}


### Tarea 3.2. Crear Cloud SQL e IAM de mínimo privilegio

Aprovisionar MySQL y conceder a la VM únicamente acceso al secreto designado y a Cloud SQL.


- {% include step_label.html %} Debajo de la Service Account, configura el rol de lectura **sobre el secreto específico** y el rol de cliente Cloud SQL.

  ```hcl
  resource "google_secret_manager_secret_iam_member" "secret_reader" {
    project   = var.project_id
    secret_id = var.secret_id
    role      = "roles/secretmanager.secretAccessor"
    member    = "serviceAccount:${google_service_account.vm.email}"
  }

  resource "google_project_iam_member" "sql_client" {
    project = var.project_id
    role    = "roles/cloudsql.client"
    member  = "serviceAccount:${google_service_account.vm.email}"
  }
  ```

  > **Nota:** El recurso `google_secret_manager_secret_iam_member` referencia el secreto creado previamente con gcloud; no crea otra copia.
  {: .lab-note .info .compact}

  > **Salida esperada:** Secret Manager se autoriza por secreto; Cloud SQL Client se asigna a la cuenta de servicio de la VM.
  {: .lab-note .output .compact}


- {% include step_label.html %} Agrega una instancia Cloud SQL MySQL para laboratorio; habilita eliminación administrada por Terraform.

  ```hcl
  resource "google_sql_database_instance" "mysql" {
    name             = var.instance_name
    database_version = "MYSQL_8_0"
    region           = var.region

    settings {
      tier              = "db-f1-micro"
      availability_type = "ZONAL"
      disk_size         = 10
      edition           = "ENTERPRISE"
      ip_configuration {
        ipv4_enabled = true
      }
    }

    deletion_protection = false
  }
  ```

  > **Nota:** Cloud SQL es facturable y puede tardar varios minutos. Si la política exige otro tier, el instructor deberá ajustar cuotas o recursos antes de aplicar.
  {: .lab-note .info .compact}

  > **Salida esperada:** Cloud SQL se creará con una instancia pequeña de edición Enterprise, no una de alta disponibilidad.
  {: .lab-note .output .compact}


- {% include step_label.html %} Debajo de `google_sql_database_instance.mysql`, declara la base `appdb`; **no crees el usuario con contraseña en Terraform**.

  ```hcl
  resource "google_sql_database" "appdb" {
    name     = var.db_name
    instance = google_sql_database_instance.mysql.name
  }
  ```

  > **Salida esperada:** Terraform administra la instancia y la base, pero no conoce ninguna contraseña de MySQL.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `outputs.tf`, publica únicamente metadatos útiles y añade un output de la conexión como **construcción dirigida**.

  1. Copia primero el siguiente bloque en `outputs.tf`:

  ```hcl
  output "vm_name" {
    value = google_compute_instance.vm.name
  }
  output "secret_id" {
    value = var.secret_id
  }
  output "sql_instance" {
    value = google_sql_database_instance.mysql.name
  }
  ```

  2. Debajo, crea `output "sql_connection_name"` usando el atributo `connection_name` de `google_sql_database_instance.mysql`.
  3. No declares `output` para valores de secretos, credenciales ni tokens.
  4. Consulta la [documentación de outputs](https://developer.hashicorp.com/terraform/language/values/outputs).

  > **Salida esperada:** `outputs.tf` incluye identificadores y `sql_connection_name`, sin devolver información confidencial.
  {: .lab-note .output .compact}


### Tarea 3.3. Aprovisionar VM y configurar el usuario MySQL fuera de Terraform

Desplegar VM sin claves JSON y ejecutar un programa que genere una contraseña en memoria, la asigne al usuario SQL y la guarde en Secret Manager.


- {% include step_label.html %} Al final de `main.tf`, crea la VM Debian 12 y adjunta su Service Account; agrega los metadatos requeridos para el ejercicio.

  ```hcl
  resource "google_compute_instance" "vm" {
    name         = "lab8-vm"
    machine_type = "e2-micro"
    zone         = var.zone
    tags         = ["lab8-ssh"]

    boot_disk {
      initialize_params {
        image = "projects/debian-cloud/global/images/family/debian-12"
        size  = 10
        type  = "pd-standard"
      }
    }

    network_interface {
      subnetwork = google_compute_subnetwork.subnet.id
      access_config {}
    }

    service_account {
      email  = google_service_account.vm.email
      scopes = ["https://www.googleapis.com/auth/cloud-platform"]
    }

    metadata = {
      "lab8-secret-id"       = var.secret_id
      "lab8-sql-connection"  = google_sql_database_instance.mysql.connection_name
      "lab8-db-name"         = var.db_name
      "lab8-db-user"         = var.db_user
    }

    depends_on = [
      google_secret_manager_secret_iam_member.secret_reader,
      google_project_iam_member.sql_client
    ]
  }
  ```

  > **Nota:** Para adjuntar la cuenta de servicio, tu identidad necesita `iam.serviceAccounts.actAs`; el acceso SSH depende de las políticas OS Login e IAP vigentes.
  {: .lab-note .info .compact}

  > **Salida esperada:** La VM recibe únicamente IDs en metadata y usa la Service Account adjunta; no hay contraseña en metadata ni en el script de inicio.
  {: .lab-note .output .compact}


- {% include step_label.html %} Inicializa y valida Terraform desde **Git Bash**; corrige cualquier error HCL antes de desplegar.

  ```bash
  terraform init
  terraform fmt -recursive
  terraform validate
  terraform plan -var-file="env/dev.tfvars"
  ```

  > **Salida esperada:** `terraform validate` muestra una configuración válida y el plan incluye VPC, subred, Firewall, Service Account, IAM, Cloud SQL, base y VM; no muestra passwords.
  {: .lab-note .output .compact}


- {% include step_label.html %} Despliega los recursos y verifica que Cloud SQL haya finalizado su aprovisionamiento. **La creación tarda aproximadamente 13 minutos**

  ```bash
  terraform apply -var-file="env/dev.tfvars"
  gcloud sql instances describe lab8-mysql --project="$PROJECT_ID" \
    --format="value(state)"
  terraform output
  ```

  > **Nota:** No inicies el programa de creación de usuario hasta que el estado de Cloud SQL sea RUNNABLE.
  {: .lab-note .info .compact}

  > **Salida esperada:** El resultado de Cloud SQL es `RUNNABLE`, la VM existe y los outputs no contienen secretos.
  {: .lab-note .output .compact}


- {% include step_label.html %} Crea `bootstrap_secret.py` con el código siguiente, revísalo y ejecútalo **una sola vez después del apply**. Generará una contraseña en memoria, creará el usuario en Cloud SQL Admin API y agregará una versión a Secret Manager sin imprimirla.

  ```python
  import json
  import secrets
  import shutil
  import string
  import subprocess
  import urllib.request

  GCLOUD = shutil.which("gcloud") or "gcloud"
  PROJECT = subprocess.check_output(
      [GCLOUD, "config", "get-value", "project"], text=True
  ).strip()
  INSTANCE = "lab8-mysql"
  SECRET = "lab8-db-password"
  USER = "app_user"
  ALPHABET = string.ascii_letters + string.digits
  password = "".join(secrets.choice(ALPHABET) for _ in range(28))

  token = subprocess.check_output(
      [GCLOUD, "auth", "print-access-token"], text=True
  ).strip()
  url = ("https://sqladmin.googleapis.com/sql/v1beta4/projects/"
         f"{PROJECT}/instances/{INSTANCE}/users")
  body = json.dumps({
      "name": USER, "host": "%", "type": "BUILT_IN", "password": password
  }).encode()
  req = urllib.request.Request(
      url, data=body, method="POST",
      headers={"Authorization": f"Bearer {token}",
               "Content-Type": "application/json"}
  )
  with urllib.request.urlopen(req, timeout=120) as response:
      print("Creación del usuario solicitada a Cloud SQL (HTTP", response.status, ")")

  subprocess.run(
      [GCLOUD, "secrets", "versions", "add", SECRET,
       "--data-file=-", f"--project={PROJECT}"],
      input=password.encode(), check=True, stdout=subprocess.DEVNULL
  )
  print("Versión del secreto creada; valor no mostrado.")
  ```

  Después de guardar `bootstrap_secret.py` en **VS Code**, ejecútalo en **Git Bash local**:

  ```bash
  python bootstrap_secret.py
  gcloud secrets versions list lab8-db-password --project="$PROJECT_ID" \
    --format="table(name,state)"
  ```

  > **Nota:** Copia el código en `lab8/bootstrap_secret.py` (no en la terminal). Si falla después de crear el usuario, no vuelvas a ejecutarlo sin revisar la causa: la repetición podría responder que el usuario ya existe.
  {: .lab-note .info .compact}

  > **Salida esperada:** El programa confirma la solicitud de creación del usuario y la nueva versión del secreto sin mostrar su valor. Antes de probar MySQL, comprueba que la operación haya concluido.
  {: .lab-note .output .compact}


{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}


---

## ⚙️ Tarea 4. Preparar Cloud SQL Auth Proxy dentro de la VM — 12 min

Accederás a la VM mediante SSH autorizado, instalarás el cliente MySQL y Cloud SQL Auth Proxy, y comprobarás que el proxy escuche localmente antes de ejecutar consultas.


### Tarea 4.1. Acceder a la VM y preparar las herramientas

Validar la identidad de la sesión y dejar listo un cliente SQL sin crear archivos con claves permanentes.


- {% include step_label.html %} Desde **Git Bash local**, revisa el nombre, zona y Service Account adjunta a la VM.

  ```bash
  gcloud compute instances describe lab8-vm \
    --project="$PROJECT_ID" --zone="us-central1-a" \
    --format="yaml(name,status,serviceAccounts,networkInterfaces)"
  ```

  > **Salida esperada:** La VM está en `RUNNING` y su cuenta de servicio termina en `lab8-vm-sa@...`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Abre una sesión SSH con IAP desde Git Bash local; si la organización exige OS Login, usa la identidad autorizada.

  ```bash
  gcloud compute ssh lab8-vm \
    --project="$PROJECT_ID" --zone="us-central1-a" \
    --tunnel-through-iap
  ```

  > **Nota:** Si aparece PERMISSION_DENIED en IAP u OS Login, solicita los roles específicos al instructor. No abras SSH a `0.0.0.0/0` ni desactives políticas.
  {: .lab-note .info .compact}

  > **Salida esperada:** La terminal cambia al shell de Debian de `lab8-vm`.
  {: .lab-note .output .compact}


- {% include step_label.html %} **Dentro de la VM**, verifica el nombre del equipo, el usuario conectado y Python 3.

  ```bash
  hostname
  whoami
  python3 --version
  ```

  > **Salida esperada:** `hostname` identifica a la VM y `python3` está disponible para ejecutar consultas a Secret Manager.
  {: .lab-note .output .compact}


- {% include step_label.html %} **Dentro de la VM**, instala el cliente MySQL y herramientas HTTPS desde los repositorios Debian.

  ```bash
  sudo apt-get update
  sudo apt-get install -y default-mysql-client curl ca-certificates
  ```

  > **Salida esperada:** El cliente `mysql` queda instalado sin almacenar contraseñas ni archivos JSON.
  {: .lab-note .output .compact}


### Tarea 4.2. Instalar y arrancar Auth Proxy

Crear un túnel autenticado hacia Cloud SQL sin exponer directamente el puerto MySQL de la instancia.


- {% include step_label.html %} **Dentro de la VM**, descarga la versión fijada del Cloud SQL Auth Proxy y comprueba su versión.

  ```bash
  curl -fsSL \
    https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.11.4/cloud-sql-proxy.linux.amd64 \
    -o /tmp/cloud-sql-proxy
  sudo install -m 755 /tmp/cloud-sql-proxy /usr/local/bin/cloud-sql-proxy
  cloud-sql-proxy --version
  mysql --version
  ```

  > **Nota:** La descarga supone VM `x86_64` (e2-micro). Si la fuente no está disponible, consulta la versión soportada en la documentación oficial.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se muestran las versiones de Cloud SQL Auth Proxy y MySQL Client.
  {: .lab-note .output .compact}


- {% include step_label.html %} **Dentro de la VM**, obtén el identificador de conexión SQL desde metadata, sin copiarlo manualmente.

  ```bash
  META="http://metadata.google.internal/computeMetadata/v1"
  CONNECTION_NAME="$(curl -fsS -H 'Metadata-Flavor: Google' \
    "$META/instance/attributes/lab8-sql-connection")"
  printf 'Instancia destino: %s\n' "$CONNECTION_NAME"
  ```

  > **Salida esperada:** La salida tiene el formato `proyecto:us-central1:lab8-mysql`; solo es un identificador, no una contraseña.
  {: .lab-note .output .compact}


- {% include step_label.html %} **Dentro de la VM**, inicia Cloud SQL Auth Proxy en segundo plano y revisa su bitácora sin habilitar trazas de credenciales.

  ```bash
  cloud-sql-proxy --port 3306 "$CONNECTION_NAME" \
    >/tmp/lab8-proxy.log 2>&1 &
  PROXY_PID=$!
  sleep 5
  kill -0 "$PROXY_PID" && echo "Auth Proxy activo"
  tail -n 8 /tmp/lab8-proxy.log
  ```

  > **Nota:** Si el puerto 3306 ya está ocupado o Cloud SQL Client no se ha propagado, soluciona ese problema antes de la prueba.
  {: .lab-note .info .compact}

  > **Salida esperada:** El proceso queda activo y el log indica que escucha en `127.0.0.1:3306`.
  {: .lab-note .output .compact}


{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}


---

## 🔍 Tarea 5. Consumir el secreto y comprobar que no se expone — 11 min

Consumirás el secreto con el token temporal de la VM, comprobarás la conexión MySQL y ejecutarás una consulta real sin imprimir la contraseña. Después inspeccionarás el estado y los permisos.


### Tarea 5.1. Conectar a MySQL usando Secret Manager

Preparar un programa que obtenga el token de la VM, lea Secret Manager y cree un archivo temporal protegido en memoria para el cliente MySQL.


- {% include step_label.html %} **Dentro de la VM**, crea `secure_mysql_test.py` con un programa que consulte metadata y Secret Manager mediante HTTPS; no pegues ninguna contraseña en el archivo.

  ```bash
  cat > ~/secure_mysql_test.py <<'PY'
  import base64
  import json
  import os
  import subprocess
  import tempfile
  import urllib.request

  META = 'http://metadata.google.internal/computeMetadata/v1'
  def metadata(path):
      req = urllib.request.Request(
          META + '/' + path, headers={'Metadata-Flavor': 'Google'})
      return urllib.request.urlopen(req, timeout=15).read().decode()

  project = metadata('project/project-id')
  secret = metadata('instance/attributes/lab8-secret-id')
  user = metadata('instance/attributes/lab8-db-user')
  database = metadata('instance/attributes/lab8-db-name')
  token = json.loads(metadata('instance/service-accounts/default/token'))['access_token']
  url = (f'https://secretmanager.googleapis.com/v1/projects/{project}/'
         f'secrets/{secret}/versions/latest:access')
  req = urllib.request.Request(url, headers={'Authorization': f'Bearer {token}'})
  with urllib.request.urlopen(req, timeout=30) as response:
      password = base64.b64decode(json.load(response)['payload']['data']).decode()

  path = None
  try:
      with tempfile.NamedTemporaryFile('w', prefix='lab8_mysql_',
                                       dir='/dev/shm', delete=False) as config:
          path = config.name
          os.chmod(path, 0o600)
          config.write('[client]\n')
          config.write(f'user={user}\npassword={password}\n')
          config.write('host=127.0.0.1\nport=3306\n')
      password = None
      subprocess.run(['mysql', f'--defaults-extra-file={path}',
                      '--protocol=TCP', database,
                      '-e', 'SELECT 1 AS conexion_ok;'], check=True)
  finally:
      if path and os.path.exists(path):
          os.remove(path)
  print('Consulta completada sin imprimir el secreto.')
  PY
  chmod 600 ~/secure_mysql_test.py
  ```

  > **Nota:** Usa `/dev/shm` (tmpfs) y elimina el archivo protegido con `finally`; no protege frente a un administrador privilegiado.
  {: .lab-note .info .compact}

  > **Salida esperada:** El programa queda en el home de la VM; no contiene valores de contraseña escritos en el código.
  {: .lab-note .output .compact}


- {% include step_label.html %} **Dentro de la VM**, ejecuta la consulta de validación sin mostrar el secreto ni utilizar `mysql -pCONTRASEÑA`.

  ```bash
  python3 ~/secure_mysql_test.py
  ```

  > **Nota:** Si aparece Access denied, comprueba IAM de la VM, el usuario SQL y la versión del secreto; no imprimas el valor para depurar.
  {: .lab-note .info .compact}

  > **Salida esperada:** La consulta devuelve `conexion_ok` con valor `1` y después `Consulta completada sin imprimir el secreto`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Amplía el programa para ejecutar SQL de prueba usando el mismo archivo protegido.

  1. Abre `~/secure_mysql_test.py` dentro de la VM con `nano ~/secure_mysql_test.py`.
  2. Ubica la cadena `'SELECT 1 AS conexion_ok;'` dentro de `subprocess.run`.
  3. Sustituye **solo** esa cadena SQL por este valor de una sola línea:

  ```python
  'CREATE TABLE IF NOT EXISTS demo (id INT PRIMARY KEY AUTO_INCREMENT, msg VARCHAR(100)); INSERT INTO demo(msg) VALUES ("Conexion OK desde VM"); SELECT id, msg FROM demo ORDER BY id DESC LIMIT 3;'
  ```

  4. Conserva las comillas externas de Python y el resto de `subprocess.run` sin cambios.
  5. Guarda el archivo, sal del editor y vuelve a ejecutar `python3 ~/secure_mysql_test.py`.

  > **Salida esperada:** La tabla `demo` contiene una fila de ejemplo y la contraseña no aparece en la terminal.
  {: .lab-note .output .compact}


- {% include step_label.html %} Verifica que no permanecen archivos temporales de credenciales y comprueba el estado del proxy.

  ```bash
  find /dev/shm -maxdepth 1 -name 'lab8_mysql_*' -print
  pgrep -af cloud-sql-proxy | head -n 5
  ```

  > **Salida esperada:** La búsqueda en `/dev/shm` no devuelve archivos de la práctica; el Auth Proxy continúa activo.
  {: .lab-note .output .compact}


### Tarea 5.2. Validar estado, outputs y políticas de no exposición

Comprobar desde la computadora del participante que el estado Terraform y la configuración no incluyen contraseñas.


- {% include step_label.html %} Sal de la VM para regresar a **Git Bash local** y consulta únicamente metadatos del secreto.

  ```bash
  exit
  ```
  ```bash
  gcloud secrets versions list lab8-db-password \
    --project="$PROJECT_ID" --format="table(name,state,createTime)"
  ```

  > **Nota:** Ejecuta `exit` solo una vez para cerrar la sesión SSH; los siguientes pasos deben realizarse dentro de `lab8` en VS Code.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se ve una versión habilitada; el comando no recupera ni imprime su contenido.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **Git Bash local**, revisa que Terraform no administre versiones de secreto ni usuarios con contraseñas.

  ```bash
  terraform state list
  terraform output
  ```
  ```bash
  if grep -RniE 'random_password|secret_data[[:space:]]*=|password[[:space:]]*=' \
    --include='*.tf' --include='*.tfvars' .; then
    echo 'Revisar coincidencias antes de publicar'
  else
    echo 'Sin asignaciones de contraseñas en HCL o tfvars'
  fi
  ```

  > **Nota:** Esta verificación por patrones no demuestra ausencia absoluta de secretos en todos los registros y memorias del sistema; se limita a archivos HCL, tfvars, estado y outputs de esta práctica.
  {: .lab-note .info .compact}

  > **Salida esperada:** El estado incluye red, SQL, IAM y VM; no incluye `random_password`, Secret Manager Secret Version ni password SQL.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba el ámbito IAM específico del secreto y registra solo la cuenta autorizada.

  ```bash
  gcloud secrets get-iam-policy lab8-db-password \
    --project="$PROJECT_ID" --format="table(bindings.role,bindings.members)"
  ```

  > **Salida esperada:** El rol `roles/secretmanager.secretAccessor` corresponde a la Service Account `lab8-vm-sa`, sin asignarse a todo el proyecto.
  {: .lab-note .output .compact}


{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}


---

## 🧹 Tarea 6. Destruir recursos y comprobar limpieza — 5 min

Destruirás los recursos administrados por Terraform, eliminarás por separado el secreto creado con gcloud y confirmarás que ya no quedan recursos de `lab8` generando cargos.


### Tarea 6.1. Destruir infraestructura Terraform

Eliminar primero Cloud SQL y VM sin afectar redes, buckets o instancias de otros laboratorios.


- {% include step_label.html %} Antes de destruir, verifica desde **Git Bash local** que el plan apunta a los recursos `lab8`.

  ```bash
  pwd
  terraform state list
  terraform plan -destroy -var-file="env/dev.tfvars"
  ```

  > **Salida esperada:** El plan de destrucción incluye la VM, Cloud SQL, IAM, red y demás recursos de `lab8`, no recursos de otros laboratorios.
  {: .lab-note .output .compact}


- {% include step_label.html %} Ejecuta la destrucción y espera hasta que Terraform finalice; no cierres la terminal a mitad del proceso.

  ```bash
  terraform destroy -var-file="env/dev.tfvars"
  terraform state list
  ```

  > **Nota:** Cloud SQL puede tardar varios minutos en eliminarse. No ejecutes una segunda destrucción mientras la primera siga activa.
  {: .lab-note .info .compact}

  > **Salida esperada:** La destrucción finaliza y `terraform state list` no devuelve recursos administrados.
  {: .lab-note .output .compact}


### Tarea 6.2. Eliminar el secreto y verificar recursos residuales

Como el secreto se creó mediante gcloud y no Terraform, su eliminación debe realizarse de manera explícita.


- {% include step_label.html %} Elimina únicamente `lab8-db-password` de Secret Manager después de comprobar que Cloud SQL y la VM ya no existen.

  ```bash
  gcloud secrets delete lab8-db-password --project="$PROJECT_ID"
  ```

  > **Nota:** Este comando es destructivo; confirma que el nombre coincide exactamente con `lab8-db-password` antes de responder `Y`.
  {: .lab-note .info .compact}

  > **Salida esperada:** Secret Manager solicita confirmación y elimina el secreto de la práctica.
  {: .lab-note .output .compact}


- {% include step_label.html %} Verifica en GCP que los recursos concretos de `lab8` ya no aparecen; no borres elementos de otras prácticas.

  ```bash
  gcloud compute instances list --project="$PROJECT_ID" \
    --filter="name=lab8-vm" --format="value(name)"
  gcloud sql instances list --project="$PROJECT_ID" \
    --filter="name=lab8-mysql" --format="value(name)"
  gcloud secrets list --project="$PROJECT_ID" \
    --filter="name:lab8-db-password" --format="value(name)"
  ```

  > **Salida esperada:** Las consultas no muestran recursos `lab8`; la infraestructura y el secreto quedaron eliminados.
  {: .lab-note .output .compact}


{% capture r6 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r6 %}

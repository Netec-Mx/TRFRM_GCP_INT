---
course_id: terrgcpint
lab_id: l001
tracking: true
position: 1
layout: lab
title: "Práctica 1: Validar y desplegar una configuración básica de Terraform en Google Cloud"
permalink: /lab1/lab1/
images_base: /labs/lab1/img
duration: "25 minutos"
objective:
  - "Validar y desplegar una VPC, una subred y una máquina virtual en Google Cloud usando el flujo esencial de Terraform."
prerequisites:
  - "Máquina virtual Windows con VS Code, Git Bash, Terraform y Google Cloud CLI instalados."
  - "Cuenta Google Cloud con proyecto activo, facturación habilitada y permisos para Compute Engine y Service Usage."
  - "Conexión a Internet y posibilidad de iniciar sesión en Google Cloud desde el navegador de Windows."
  - "Conocimientos básicos de archivos HCL, VPC y rangos CIDR."
introduction:
  - "Desde la terminal Git Bash integrada en VS Code, prepararás el directorio del curso, autenticarás Google Cloud, definirás tres recursos con Terraform y completarás el ciclo init, validate, plan, apply y destroy."
slug: lab1
lab_number: 1
final_result: >
  Habrás creado y comprobado una VPC, una subred y una VM e2-micro, examinado el estado local de Terraform y eliminado los recursos en Google Cloud. Conservarás el código HCL en lab1 para consultas posteriores.
notes:
  - "El directorio terraform-gcp-intermediate se crea únicamente en esta práctica; las prácticas posteriores añaden lab2, lab3, etc."
  - "Todos los comandos bash se ejecutan en la terminal Git Bash integrada en VS Code, salvo los pasos que indican usar la interfaz gráfica."
  - "Los 25 minutos suponen que la CLI, la facturación y los permisos de GCP ya están preparados; autenticación, descargas o cuotas pueden incrementar el tiempo."
  - "El archivo terraform.tfstate puede contener información sensible: no compartirlo ni agregarlo al control de versiones."
references:
  - text: "Autenticación de Terraform en Google Cloud"
    url: https://cloud.google.com/docs/terraform/authentication
  - text: "Terraform CLI: ciclo de trabajo"
    url: https://developer.hashicorp.com/terraform/cli/commands
  - text: "Documentación del proveedor Google"
    url: https://registry.terraform.io/providers/hashicorp/google/latest/docs
  - text: "Creación de VM desde una imagen pública"
    url: https://cloud.google.com/compute/docs/instances/create-vm-from-public-image
prev: /
next: /lab2/lab2/
---

## 🔎 Tarea 1. Preparar el entorno y autenticar Google Cloud — 6 min

{% assign tracking_task_id = "t001" %}

Establecerás el directorio reutilizable del curso, abrirás la terminal correcta y vincularás la CLI y las credenciales ADC con un proyecto de Google Cloud.

### Tarea 1.1. Crear la estructura de carpetas

Crear el directorio central únicamente en la primera práctica y preparar lab1 para conservar el código sin mezclar estados entre laboratorios.

- {% include step_label.html id="s001" %} En **Windows**, abre **VS Code** desde el menú Inicio.

  > **Nota:** No es necesario ejecutar VS Code como administrador.
  {: .lab-note .info .compact}

  > **Salida esperada:** VS Code muestra su ventana principal.
  {: .lab-note .output .compact}

- {% include step_label.html id="s002" %} En **VS Code**, abre **Terminal > New Terminal** y selecciona **Git Bash** en el selector de perfiles.

  > **Importante:** Comprueba que la terminal sea Bash y no PowerShell ni Command Prompt.
  {: .lab-note .important .compact}

  > **Salida esperada:** La terminal presenta un prompt Git Bash, generalmente terminado en `$`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s003" %} En **Git Bash**, crea el directorio central y la carpeta `lab1` en el Escritorio de Windows.

  > **Nota:** Se utiliza la ruta de Windows al Escritorio; si está redirigido a OneDrive, ver la comprobación siguiente.
  {: .lab-note .info .compact}

  ```bash
  DESKTOP="$(powershell.exe -NoProfile -Command "[Environment]::GetFolderPath('Desktop')" | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab1"
  ```

  > **Salida esperada:** Se crea la carpeta `terraform-gcp-intermediate/lab1` en el Escritorio efectivo del usuario.
  {: .lab-note .output .compact}

- {% include step_label.html id="s004" %} En **Git Bash**, entra en `lab1` y comprueba la ruta de trabajo.

  > **Nota:** Todos los archivos de la práctica se guardarán dentro de esta carpeta.
  {: .lab-note .info .compact}

  ```bash
  cd "$DESKTOP/terraform-gcp-intermediate/lab1"
  pwd
  ```

  > **Salida esperada:** La ruta termina en `/terraform-gcp-intermediate/lab1`.
  {: .lab-note .output .compact}

### Tarea 1.2. Verificar VS Code y las herramientas instaladas

Confirmarás que Terraform y gcloud responden desde Git Bash antes de generar código y descargar proveedores.

- {% include step_label.html id="s005" %} En **VS Code**, selecciona **File > Open Folder** y abre la carpeta `lab1` recién creada.

  > **Nota:** Si VS Code pide confirmar la confianza en la carpeta que creaste, selecciona **Yes, I trust the authors**.
  {: .lab-note .info .compact}

  > **Salida esperada:** El explorador lateral muestra `lab1` como carpeta de trabajo.
  {: .lab-note .output .compact}

- {% include step_label.html id="s006" %} En la terminal **Git Bash de VS Code**, consulta la versión de Terraform.

  > **Importante:** Terraform debe estar accesible mediante PATH; si no, solicita corregir la instalación antes de avanzar.
  {: .lab-note .important .compact}

  ```bash
  terraform version
  ```

  > **Salida esperada:** Se muestra `Terraform v...` y la arquitectura de Windows.
  {: .lab-note .output .compact}

- {% include step_label.html id="s007" %} En la misma terminal, consulta la versión instalada de Google Cloud CLI.

  > **Nota:** Si `gcloud` no se reconoce, reinicia VS Code después de instalar la CLI o revisa PATH.
  {: .lab-note .info .compact}

  ```bash
  gcloud version
  ```

  > **Salida esperada:** Aparecen las versiones instaladas de Google Cloud CLI y componentes.
  {: .lab-note .output .compact}

### Tarea 1.3. Autenticar la CLI y seleccionar el proyecto

Iniciarás sesión en GCP, configurarás el proyecto de la práctica y generarás las credenciales ADC que usará Terraform.

- {% include step_label.html id="s008" %} En **Git Bash**, inicia sesión en Google Cloud y completa el flujo de navegador.

  > **Nota:** Utiliza la cuenta que tiene acceso al proyecto donde desplegarás los recursos.
  {: .lab-note .info .compact}

  ```bash
  gcloud auth login
  ```

  > **Salida esperada:** El navegador permite iniciar sesión y gcloud confirma la cuenta autenticada.
  {: .lab-note .output .compact}

- {% include step_label.html id="s009" %} En **Git Bash**, sustituye el marcador por el **Project ID**, no por el nombre visible del proyecto.

  > **Importante:** No incluyas espacios en el ID ni uses el número del proyecto.
  {: .lab-note .important .compact}

  ```bash
  export TF_VAR_project_id="TU_PROJECT_ID"
  gcloud config set project "$TF_VAR_project_id"
  ```

  > **Salida esperada:** gcloud confirma el proyecto seleccionado.
  {: .lab-note .output .compact}

- {% include step_label.html id="s010" %} En **Git Bash**, crea las **Application Default Credentials (ADC)** utilizadas por Terraform.

  > **Nota:** Este inicio de sesión es distinto de `gcloud auth login` y puede abrir nuevamente el navegador.
  {: .lab-note .info .compact}

  ```bash
  gcloud auth application-default login
  ```

  > **Salida esperada:** Se guardan credenciales ADC locales del usuario sin generar claves de service account.
  {: .lab-note .output .compact}

- {% include step_label.html id="s011" %} En **Git Bash**, asigna el proyecto de cuotas a las ADC y verifica el proyecto activo.

  > **Nota:** Si falta `serviceusage.services.use`, un administrador deberá conceder el permiso necesario.
  {: .lab-note .info .compact}

  ```bash
  gcloud auth application-default set-quota-project "$TF_VAR_project_id"
  gcloud config get-value project
  ```

  > **Salida esperada:** La última línea muestra el mismo Project ID de `TF_VAR_project_id`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s012" %} En **Git Bash**, comprueba si la API Compute Engine está activa para el proyecto.

  > **Advertencia:** Si aparece `DISABLED`, habilítala solo si tienes permiso; la activación puede tardar.
  {: .lab-note .warning .compact}

  ```bash
  gcloud services list --enabled --filter="config.name=compute.googleapis.com" --format="value(config.name)"
  # Solo si no figura habilitada:
  # gcloud services enable compute.googleapis.com --project="$TF_VAR_project_id"
  ```

  > **Salida esperada:** Aparece `compute.googleapis.com`. Si no aparece, ejecuta el comando comentado sin `#`.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## ☁️ Tarea 2. Definir los recursos de Terraform — 5 min

{% assign tracking_task_id = "t002" %}

Crearás un archivo main.tf con los bloques necesarios de proveedor, variable, VPC, subred, VM y output; después verificarás su contenido en VS Code.

### Tarea 2.1. Configurar Terraform y Google Provider

Definirás la versión mínima de Terraform, la dependencia del proveedor oficial y la identificación del proyecto en HCL.

- {% include step_label.html id="s013" %} En **VS Code > Explorer**, selecciona **New File**, escribe `main.tf` y pulsa **Enter**.

  > **Nota:** Crea `main.tf` directamente dentro de `lab1`; no crees subcarpetas adicionales.
  {: .lab-note .info .compact}

  > **Salida esperada:** El archivo vacío `main.tf` aparece en el explorador.
  {: .lab-note .output .compact}

- {% include step_label.html id="s014" %} En `main.tf`, agrega el bloque `terraform` para registrar el proveedor oficial de Google.

  > **Importante:** El rango `~> 6.0` mantiene la versión mayor 6; se instala la versión compatible disponible.
  {: .lab-note .important .compact}

  ```hcl
  terraform {
    required_version = ">= 1.8.0"
    required_providers {
      google = {
        source  = "hashicorp/google"
        version = "~> 6.0"
      }
    }
  }
  ```

  > **Salida esperada:** Se define el proveedor `hashicorp/google` sin fijar un parche obsoleto.
  {: .lab-note .output .compact}

- {% include step_label.html id="s015" %} Debajo del bloque `terraform`, agrega el proveedor con proyecto, región y zona.

  > **Nota:** La variable `project_id` se recibirá de `TF_VAR_project_id` en Git Bash.
  {: .lab-note .info .compact}

  ```hcl
  provider "google" {
    project = var.project_id
    region  = "us-central1"
    zone    = "us-central1-a"
  }
  ```

  > **Salida esperada:** El proveedor apunta a `us-central1` y `us-central1-a`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s016" %} Debajo del proveedor, declara la variable `project_id` y guarda `main.tf` con **Ctrl+S**.

  > **Importante:** No escribas credenciales, tokens ni claves JSON dentro del archivo HCL.
  {: .lab-note .important .compact}

  ```hcl
  variable "project_id" {
    type        = string
    description = "ID del proyecto de Google Cloud"
  }
  ```

  > **Salida esperada:** `main.tf` queda guardado con la variable requerida.
  {: .lab-note .output .compact}

### Tarea 2.2. Crear VPC, subred, VM y salida

Construirás tres recursos relacionados mediante referencias HCL y una salida que permitirá consultar la IP pública temporal.

- {% include step_label.html id="s017" %} Debajo de `variable "project_id"`, agrega una VPC sin subredes automáticas.

  > **Nota:** La red se crea con un nombre exclusivo de esta práctica para facilitar su limpieza.
  {: .lab-note .info .compact}

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = "lab01-vpc"
    auto_create_subnetworks = false
  }
  ```

  > **Salida esperada:** La VPC se llama `lab01-vpc` y usa subredes manuales.
  {: .lab-note .output .compact}

- {% include step_label.html id="s018" %} Debajo de la VPC, agrega la subred con CIDR `10.10.1.0/24` y referencia a la red creada.

  > **Nota:** La subred está en `us-central1`, donde se ubicará la VM.
  {: .lab-note .info .compact}

  ```hcl
  resource "google_compute_subnetwork" "subnet" {
    name          = "lab01-subnet"
    ip_cidr_range = "10.10.1.0/24"
    region        = "us-central1"
    network       = google_compute_network.vpc.id
  }
  ```

  > **Salida esperada:** La subred depende automáticamente de la VPC mediante su ID.
  {: .lab-note .output .compact}

- {% include step_label.html id="s019" %} Debajo de la subred, comienza el recurso VM con nombre, tipo de máquina y zona.

  > **Importante:** Este recurso quedará incompleto hasta terminar los dos pasos siguientes; aún no ejecutes `validate`.
  {: .lab-note .important .compact}

  ```hcl
  resource "google_compute_instance" "vm" {
    name         = "lab01-vm-instancia"
    machine_type = "e2-micro"
    zone         = "us-central1-a"
  ```

  > **Salida esperada:** Se inició el bloque de VM; continúa debajo sin cerrar todavía el recurso.
  {: .lab-note .output .compact}

- {% include step_label.html id="s020" %} Debajo de `zone`, agrega el disco de arranque con la familia de imágenes Debian 12.

  > **Nota:** Se utiliza una familia de imagen actualizable, en lugar de una versión fija de Debian 11.
  {: .lab-note .info .compact}

  ```hcl
    boot_disk {
      initialize_params {
        image = "debian-cloud/debian-12"
      }
    }
  ```

  > **Salida esperada:** El disco de arranque identifica la familia `debian-12`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s021" %} Debajo de `boot_disk`, agrega interfaz de red, IP pública y cierra el recurso VM.

  > **Advertencia:** Una IP pública y una VM pueden generar cargos; la infraestructura será eliminada al final.
  {: .lab-note .warning .compact}

  ```hcl
    network_interface {
      subnetwork = google_compute_subnetwork.subnet.id
      access_config {}
    }
  
    labels = {
      entorno = "laboratorio"
    }
  }
  ```

  > **Salida esperada:** La VM referencia la subred y solicita una IP externa efímera.
  {: .lab-note .output .compact}

- {% include step_label.html id="s022" %} Al final de `main.tf`, fuera del recurso VM, agrega el output y guarda con **Ctrl+S**.

  > **Nota:** La IP pública no se conocerá hasta que GCP cree la VM.
  {: .lab-note .info .compact}

  ```hcl
  output "vm_public_ip" {
    description = "IP externa efimera de la VM"
    value       = google_compute_instance.vm.network_interface[0].access_config[0].nat_ip
  }
  ```

  > **Salida esperada:** El archivo contiene `terraform`, `provider`, `variable`, tres `resource` y un `output`.
  {: .lab-note .output .compact}

{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## 🚀 Tarea 3. Inicializar, validar y desplegar — 9 min

{% assign tracking_task_id = "t003" %}

Inicializarás el proveedor, revisarás la validez de HCL, estudiarás el plan de tres recursos y confirmarás su creación mediante Terraform y Google Cloud CLI.

### Tarea 3.1. Inicializar y validar el directorio

Prepararás el directorio local y revisarás la estructura HCL antes de solicitar modificaciones a la infraestructura en GCP.

- {% include step_label.html id="s023" %} En **Git Bash**, verifica que el terminal se encuentre dentro de `lab1`.

  > **Nota:** Terraform opera sobre los archivos `.tf` del directorio actual.
  {: .lab-note .info .compact}

  ```bash
  pwd
  ```

  > **Salida esperada:** La ruta finaliza en `/terraform-gcp-intermediate/lab1`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s024" %} En **Git Bash**, inicializa el directorio y descarga el proveedor Google.

  > **Nota:** La primera descarga requiere conexión a Internet y puede tardar más que las siguientes.
  {: .lab-note .info .compact}

  ```bash
  terraform init
  ```

  > **Salida esperada:** Se muestra `Terraform has been successfully initialized!` y se crea `.terraform.lock.hcl`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s025" %} En **Git Bash**, comprueba el formato del archivo HCL.

  > **Nota:** `terraform fmt` normaliza automáticamente sangrías y alineación.
  {: .lab-note .info .compact}

  ```bash
  terraform fmt
  ```

  > **Salida esperada:** Se formatea `main.tf` si había diferencias; si ya estaba formateado, puede no imprimir nada.
  {: .lab-note .output .compact}

- {% include step_label.html id="s026" %} En **Git Bash**, valida la sintaxis y las referencias de la configuración.

  > **Importante:** `validate` no comprueba cuotas, facturación ni que tengas permisos para crear recursos.
  {: .lab-note .important .compact}

  ```bash
  terraform validate
  ```

  > **Salida esperada:** Se muestra `Success! The configuration is valid.`.
  {: .lab-note .output .compact}

### Tarea 3.2. Crear y revisar el plan

Generarás una propuesta de cambios sin crear recursos y comprobarás que Terraform planea exactamente una VPC, una subred y una VM.

- {% include step_label.html id="s027" %} En **Git Bash**, confirma que `TF_VAR_project_id` sigue disponible en la terminal.

  > **Nota:** Una nueva sesión Bash puede requerir volver a exportar la variable de entorno.
  {: .lab-note .info .compact}

  ```bash
  printf "Proyecto: %s\n" "$TF_VAR_project_id"
  ```

  > **Salida esperada:** Aparece el Project ID correcto y no una cadena vacía.
  {: .lab-note .output .compact}

- {% include step_label.html id="s028" %} En **Git Bash**, genera el plan y guárdalo como `lab1.tfplan`.

  > **Nota:** Un plan guardado permite aplicar exactamente el cambio revisado, si no ha cambiado el entorno.
  {: .lab-note .info .compact}

  ```bash
  terraform plan -out=lab1.tfplan
  ```

  > **Salida esperada:** Aparece `Plan: 3 to add, 0 to change, 0 to destroy.` si el proyecto no tiene esos recursos.
  {: .lab-note .output .compact}

- {% include step_label.html id="s029" %} En la salida del plan, identifica las acciones `+ create` de VPC, subred y VM.

  > **Importante:** Si el plan difiere de tres creaciones, no avances hasta identificar la causa.
  {: .lab-note .important .compact}

  > **Salida esperada:** La propuesta contiene los tres tipos `google_compute_network`, `google_compute_subnetwork` y `google_compute_instance`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s030" %} En **Git Bash**, muestra únicamente el resumen del plan guardado.

  > **Nota:** `terraform show` permite volver a consultar el plan sin volver a ejecutar `terraform plan`.
  {: .lab-note .info .compact}

  ```bash
  terraform show -no-color lab1.tfplan | tail -n 8
  ```

  > **Salida esperada:** El resumen indica tres altas y el output aparece como `(known after apply)`.
  {: .lab-note .output .compact}

### Tarea 3.3. Aplicar y comprobar el despliegue

Aprovisionarás los recursos en GCP y verificarás que la VM existe y que Terraform obtiene su dirección pública.

- {% include step_label.html id="s031" %} En **Git Bash**, ejecuta el plan guardado para desplegar los tres recursos.

  > **Advertencia:** Se crearán recursos reales con posibles costos en el proyecto configurado.
  {: .lab-note .warning .compact}

  ```bash
  terraform apply lab1.tfplan
  ```

  > **Salida esperada:** Al finalizar aparece `Apply complete! Resources: 3 added, 0 changed, 0 destroyed.`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s032" %} En **Git Bash**, muestra el valor de salida `vm_public_ip`.

  > **Nota:** Si no se muestra una IP, revisa la salida de `apply` y la interfaz de red configurada.
  {: .lab-note .info .compact}

  ```bash
  terraform output vm_public_ip
  ```

  > **Salida esperada:** Se muestra una dirección IPv4 pública asignada a la VM.
  {: .lab-note .output .compact}

- {% include step_label.html id="s033" %} En **Git Bash**, consulta la VM recién creada mediante Google Cloud CLI.

  > **Nota:** Usa la misma zona indicada en la configuración HCL.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute instances describe lab01-vm-instancia --zone=us-central1-a --format="value(name,status)"
  ```

  > **Salida esperada:** Se muestra `lab01-vm-instancia` y un estado, normalmente `RUNNING`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s034" %} En **Git Bash**, comprueba que la VPC existe en el proyecto seleccionado.

  > **Nota:** La red se creó sin subredes automáticas.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute networks describe lab01-vpc --format="value(name,autoCreateSubnetworks)"
  ```

  > **Salida esperada:** Se muestran el nombre `lab01-vpc` y el valor `False`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s035" %} En **Git Bash**, comprueba la subred y el rango CIDR.

  > **Nota:** El CIDR debe coincidir con el definido en `main.tf`.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute networks subnets describe lab01-subnet --region=us-central1 --format="value(name,ipCidrRange)"
  ```

  > **Salida esperada:** Se muestran `lab01-subnet` y `10.10.1.0/24`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s036" %} En el navegador de **Windows**, abre **Google Cloud Console > Compute Engine > VM instances**.

  > **Importante:** Selecciona el mismo proyecto de `TF_VAR_project_id` antes de buscar la VM.
  {: .lab-note .important .compact}

  > **Salida esperada:** La instancia `lab01-vm-instancia` aparece en `us-central1-a`.
  {: .lab-note .output .compact}

{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## 🧹 Tarea 4. Inspeccionar el estado y eliminar recursos — 5 min

{% assign tracking_task_id = "t004" %}

Revisarás la relación entre el estado local y GCP, comprobarás que no hay cambios pendientes y destruirás los recursos para detener costos.

### Tarea 4.1. Explorar el estado local de Terraform

Revisarás el archivo de estado y las direcciones de recursos sin editar el JSON manualmente ni exponer credenciales.

- {% include step_label.html id="s037" %} En **VS Code > Explorer**, identifica el archivo `terraform.tfstate` dentro de `lab1`.

  > **Advertencia:** No edites ni compartas este archivo; puede incluir atributos sensibles de infraestructura.
  {: .lab-note .warning .compact}

  > **Salida esperada:** El archivo `terraform.tfstate` existe tras ejecutar `terraform apply`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s038" %} En **Git Bash**, lista las direcciones de recursos administradas por Terraform.

  > **Nota:** Este comando es más seguro y claro que modificar manualmente el JSON del estado.
  {: .lab-note .info .compact}

  ```bash
  terraform state list
  ```

  > **Salida esperada:** Aparecen las direcciones `google_compute_network.vpc`, `google_compute_subnetwork.subnet` y `google_compute_instance.vm`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s039" %} En **Git Bash**, revisa los detalles registrados para la VM.

  > **Nota:** El estado contiene el vínculo entre el recurso HCL y el recurso real.
  {: .lab-note .info .compact}

  ```bash
  terraform state show google_compute_instance.vm
  ```

  > **Salida esperada:** Se muestra el ID de la instancia y sus atributos, incluida la zona.
  {: .lab-note .output .compact}

- {% include step_label.html id="s040" %} En **Git Bash**, comprueba la consistencia entre la configuración y lo desplegado.

  > **Importante:** El resultado sin cambios supone que nadie modificó los recursos manualmente en GCP.
  {: .lab-note .important .compact}

  ```bash
  terraform plan
  ```

  > **Salida esperada:** Se muestra `No changes. Your infrastructure matches the configuration.`.
  {: .lab-note .output .compact}

### Tarea 4.2. Destruir y verificar la limpieza

Eliminarás todos los recursos administrados por la práctica y comprobarás su desaparición desde Terraform y Google Cloud CLI.

- {% include step_label.html id="s041" %} En **Git Bash**, inicia la eliminación controlada de los recursos del laboratorio.

  > **Advertencia:** Este comando elimina de verdad la VM, la subred y la VPC; verifica que estás dentro de `lab1`.
  {: .lab-note .warning .compact}

  ```bash
  terraform destroy
  ```

  > **Salida esperada:** Terraform prepara un plan de destrucción y solicita confirmación.
  {: .lab-note .output .compact}

- {% include step_label.html id="s042" %} En el prompt de **Git Bash**, escribe `yes` y pulsa **Enter** para confirmar.

  > **Importante:** No cierres la terminal hasta recibir la confirmación de que terminó la destrucción.
  {: .lab-note .important .compact}

  > **Salida esperada:** Al terminar aparece `Destroy complete! Resources: 3 destroyed.`.
  {: .lab-note .output .compact}

- {% include step_label.html id="s043" %} En **Git Bash**, comprueba que Terraform ya no administra recursos del laboratorio.

  > **Nota:** Conserva `main.tf` y los archivos de trabajo para referencia en las siguientes prácticas.
  {: .lab-note .info .compact}

  ```bash
  terraform state list
  ```

  > **Salida esperada:** El comando finaliza sin direcciones de recursos.
  {: .lab-note .output .compact}

- {% include step_label.html id="s044" %} En **Git Bash**, confirma en GCP que la VM ya no existe.

  > **Nota:** Un resultado vacío es el esperado después de `destroy`.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute instances list --filter="name=lab01-vm-instancia" --format="value(name)"
  ```

  > **Salida esperada:** No aparece ninguna instancia con el nombre `lab01-vm-instancia`.
  {: .lab-note .output .compact}

{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}
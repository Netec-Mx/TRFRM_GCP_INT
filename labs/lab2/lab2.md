---
layout: lab
title: "Práctica 2: Crear y usar módulos reutilizables en Terraform"
permalink: /lab2/lab2/
images_base: /labs/lab2/img
duration: "55 minutos"
objective:
  - "Separar una infraestructura de Google Cloud en módulos reutilizables de red y cómputo, consumir sus variables y outputs desde el módulo raíz y comprobar el despliegue y acceso SSH."
prerequisites:
  - "Práctica 1 completada y carpeta terraform-gcp-intermediate creada en el Escritorio de Windows."
  - "VM Windows con VS Code, Git Bash, Terraform, gcloud CLI y cliente OpenSSH disponibles."
  - "Proyecto de GCP con facturación, Compute Engine API habilitada y permisos para administrar red, firewall e instancias."
  - "Autenticación ADC de Google Cloud previamente realizada; conexión a Internet y acceso al navegador."
introduction:
  - "Organizarás un proyecto Terraform en un módulo raíz y dos módulos locales: network, que crea VPC, subred y firewall SSH, y vm, que crea Compute Engine. Conectarás los módulos mediante variables y outputs, desplegarás los cuatro recursos, probarás SSH y limpiarás el entorno."
slug: lab2
lab_number: 2
final_result: >
  Habrás creado los módulos reutilizables network y vm, los habrás integrado desde main, desplegado y comprobado una VPC, subred, firewall y VM en Google Cloud, validado la conexión SSH y eliminado los recursos sin borrar el código fuente.
notes:
  - "Ejecuta todos los comandos de terminal desde Git Bash integrado en VS Code; el directorio central ya existe desde la Práctica 1."
  - "En esta práctica se usan claves SSH en metadatos de instancia; el proyecto debe permitirlas y OS Login no debe estar impuesto por una política de organización."
  - "La regla de firewall limita SSH a tu IP pública /32. Si cambia tu IP, ajusta terraform.tfvars, aplica cambios y vuelve a conectar."
  - "La estimación de 40 minutos supone API, permisos y facturación listos; conexiones lentas y restricciones organizacionales pueden ampliarla."
  - "Nunca compartas la clave SSH privada ni publiques terraform.tfstate, terraform.tfvars con datos sensibles o archivos de planes."
references:
  - text: "Módulos de Terraform"
    url: https://developer.hashicorp.com/terraform/language/modules
  - text: "Variables de entrada"
    url: https://developer.hashicorp.com/terraform/language/values/variables
  - text: "Outputs de Terraform"
    url: https://developer.hashicorp.com/terraform/language/values/outputs
  - text: "Proveedor Google en Terraform Registry"
    url: https://registry.terraform.io/providers/hashicorp/google/latest/docs
  - text: "Llaves SSH en metadatos de Compute Engine"
    url: https://cloud.google.com/compute/docs/connect/add-ssh-keys
prev: /lab1/lab1/
next: /lab3/lab3/
---

---

## 🔎 Tarea 1. Verificar el entorno y crear la estructura modular — 5 min

Comprobarás que GCP sigue accesible desde Git Bash y prepararás la carpeta `lab2` dentro del directorio central ya existente, sin modificar el código ni el estado de `lab1`.

### Tarea 1.1. Comprobar herramientas y proyecto activo

Verificar versiones, proyecto y credenciales disponibles antes de comenzar la configuración de módulos locales.

- {% include step_label.html %} En **VS Code**, abre **Terminal > New Terminal** y selecciona el perfil **Git Bash**.

  > **Salida esperada:** La terminal integrada muestra un prompt Bash terminado en `$`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **Git Bash**, confirma que Terraform, gcloud y SSH están disponibles.

  ```bash
  terraform version
  gcloud version | head -n 3
  ssh -V
  ```

  > **Salida esperada:** Se muestran versiones de Terraform, gcloud CLI y OpenSSH, sin errores de comando inexistente.
  {: .lab-note .output .compact}

- {% include step_label.html %} Verifica el proyecto activo y que las credenciales ADC ya existan.

  > **Importante:** Si no aparece el proyecto, ejecuta `gcloud config set project TU_PROJECT_ID`; si falta ADC, `gcloud auth application-default login`.
  {: .lab-note .important .compact}

  ```bash
  gcloud config get-value project
  gcloud auth application-default print-access-token >/dev/null && echo "ADC disponible"
  ```

  > **Salida esperada:** Aparece el ID del proyecto del curso y el mensaje `ADC disponible`.
  {: .lab-note .output .compact}

### Tarea 1.2. Preparar el directorio lab2 y los archivos

Crear las carpetas root module y child modules en el Escritorio efectivo de Windows y abrirlas en VS Code.

- {% include step_label.html %} En **Git Bash**, localiza el Escritorio real y crea **lab2**, sin volver a crear otro directorio central.

  ```bash
  DESKTOP="$(powershell.exe -NoProfile -Command "[Environment]::GetFolderPath('Desktop')" | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab2"
  cd "$DESKTOP/terraform-gcp-intermediate/lab2"
  ```

  > **Salida esperada:** La ruta actual termina en `terraform-gcp-intermediate/lab2`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Dentro de **lab2**, crea las carpetas `main`, `modules/network` y `modules/vm`.

  ```bash
  mkdir -p main modules/network modules/vm
  find . -maxdepth 2 -type d | sort
  ```

  > **Salida esperada:** Se listan `./main`, `./modules/network` y `./modules/vm`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Crea los archivos `.tf` vacíos y abre la carpeta **lab2** mediante **File > Open Folder** en VS Code.

  > **Nota:** El directorio `main` contendrá el state del laboratorio; ejecutarás Terraform exclusivamente desde ahí.
  {: .lab-note .info .compact}

  ```bash
  touch main/{providers.tf,main.tf,variables.tf,terraform.tfvars,outputs.tf}
  touch modules/network/{main.tf,variables.tf,outputs.tf}
  touch modules/vm/{main.tf,variables.tf,outputs.tf}
  find . -type f | sort
  ```

  > **Salida esperada:** El explorador de VS Code muestra 11 archivos en la estructura modular esperada.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}

{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## 🌐 Tarea 2. Construir el módulo reutilizable de red — 5 min

Crearás el child module `network` con la VPC, una subred personalizada y una regla SSH restringida; exportarás sus identificadores para que el módulo principal y la VM puedan reutilizarlos.

### Tarea 2.1. Definir recursos de VPC, subred y Firewall

Configurar los tres recursos de Google Cloud y vincularlos mediante referencias HCL en `modules/network/main.tf`.

- {% include step_label.html %} Abre **modules/network/main.tf** y define la VPC personalizada.

  ```hcl
  resource "google_compute_network" "vpc" {
    project                 = var.project_id
    name                    = var.network_name
    auto_create_subnetworks = false
  }
  ```

  > **Salida esperada:** El recurso `google_compute_network.vpc` está declarado sin subredes automáticas.
  {: .lab-note .output .compact}

- {% include step_label.html %} En el mismo archivo, **debajo del cierre de la VPC**, agrega la subred de la región indicada.

  ```hcl
  resource "google_compute_subnetwork" "subnet" {
    project       = var.project_id
    name          = var.subnetwork_name
    region        = var.region
    ip_cidr_range = var.ip_cidr_range
    network       = google_compute_network.vpc.id
  }
  ```

  > **Salida esperada:** La subred hace referencia a `google_compute_network.vpc.id`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de la subred, agrega el Firewall SSH para el tag `lab2-ssh`, limitado al CIDR de origen.

  > **Advertencia:** No utilices `0.0.0.0/0`: expone SSH a todo Internet. Solo la IP pública del participante debe aparecer en `allowed_ssh_cidr`.
  {: .lab-note .warning .compact}

  ```hcl
  resource "google_compute_firewall" "ssh" {
    project       = var.project_id
    name          = "${var.network_name}-allow-ssh"
    network       = google_compute_network.vpc.name
    direction     = "INGRESS"
    source_ranges = [var.allowed_ssh_cidr]
    target_tags   = ["lab2-ssh"]
    allow {
      protocol = "tcp"
      ports    = ["22"]
    }
  }
  ```

  > **Salida esperada:** El Firewall permite TCP/22 solamente al rango `allowed_ssh_cidr` y al tag `lab2-ssh`.
  {: .lab-note .output .compact}

### Tarea 2.2. Declarar las variables del módulo Network

Exponer las entradas del módulo que definirán proyecto, nombres, región y rango IP autorizado.

- {% include step_label.html %} En **modules/network/variables.tf**, define las variables para identificar el proyecto y la región.

  ```hcl
  variable "project_id" { type = string }
  variable "region"     { type = string }
  ```

  > **Salida esperada:** Se declaran `project_id` y `region`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de esas variables, declara los nombres que recibirá el módulo.

  ```hcl
  variable "network_name"    { type = string }
  variable "subnetwork_name" { type = string }
  ```

  > **Salida esperada:** El módulo acepta nombres parametrizados de VPC y subred.
  {: .lab-note .output .compact}

- {% include step_label.html %} Al final de **variables.tf**, agrega `ip_cidr_range` y `allowed_ssh_cidr` usando el ejemplo.

  > **Importante:** El CIDR de la subred y el CIDR autorizado para SSH son valores distintos.
  {: .lab-note .important .compact}

  ```hcl
  variable "ip_cidr_range"    { type = string }
  variable "allowed_ssh_cidr" { type = string }
  ```

  > **Salida esperada:** El módulo expone seis variables de entrada.
  {: .lab-note .output .compact}

### Tarea 2.3. Publicar salidas reutilizables del módulo

Configurar outputs, incluyendo una salida breve que el participante escribirá con apoyo de la documentación oficial.

- {% include step_label.html %} En **modules/network/outputs.tf**, agrega primero el output de la VPC.

  ```hcl
  output "network_self_link" {
    description = "Enlace de la VPC creada"
    value       = google_compute_network.vpc.self_link
  }
  ```

  > **Salida esperada:** El output `network_self_link` referencia la VPC.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Debajo de `network_self_link`**, crea un `output` llamado `subnetwork_self_link` con la propiedad `self_link` de `google_compute_subnetwork.subnet`.

  > **Nota:** Incluye una descripción. Consulta la [documentación oficial de outputs](https://developer.hashicorp.com/terraform/language/values/outputs); no copies el bloque anterior sin ajustar la referencia.
  {: .lab-note .info .compact}

  > **Salida esperada:** El archivo contiene `subnetwork_self_link` con `value = google_compute_subnetwork.subnet.self_link`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de la salida de la subred, agrega el output del nombre del Firewall.

  ```hcl
  output "ssh_firewall_name" {
    description = "Nombre de la regla SSH"
    value       = google_compute_firewall.ssh.name
  }
  ```

  > **Salida esperada:** El módulo Network expone tres outputs: VPC, subred y Firewall.
  {: .lab-note .output .compact}

{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## 🖥️ Tarea 3. Construir el módulo reutilizable de VM — 6 min

Separarás la creación de Compute Engine en `modules/vm`, configurando imagen, disco, red, metadatos SSH, etiquetas y outputs; evitarás mezclar OS Login con llaves en metadatos.

### Tarea 3.1. Declarar el recurso Compute Engine

Crear una instancia de bajo costo con parámetros externos y una subred recibida del módulo Network.

- {% include step_label.html %} En **modules/vm/main.tf**, inicia la VM y configura tipo, zona, tag y disco.

  ```hcl
  resource "google_compute_instance" "vm" {
    project      = var.project_id
    name         = var.vm_name
    machine_type = var.machine_type
    zone         = var.zone
    tags         = ["lab2-ssh"]
  
    boot_disk {
      initialize_params {
        image = var.image
        size  = var.boot_disk_size_gb
        type  = var.boot_disk_type
      }
    }
  }
  ```

  > **Salida esperada:** Existe un recurso `google_compute_instance.vm` con disco de arranque definido.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Dentro de `google_compute_instance.vm`**, agrega el bloque `network_interface` **después de `boot_disk` y antes de la llave final**.

  ```hcl
  network_interface {
    subnetwork = var.subnetwork_self_link
    access_config {}
  }
  ```

  > **Salida esperada:** La interfaz utiliza la subred recibida del módulo Network y solicita una IP pública efímera.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Debajo de `network_interface` y dentro de la VM**, configura la llave pública SSH en metadatos.

  > **Advertencia:** No agregues `enable-oslogin = "TRUE"`: OS Login ignora las llaves de metadatos. Este ejercicio supone que la organización permite esta modalidad.
  {: .lab-note .warning .compact}

  ```hcl
  metadata = {
    enable-oslogin = "FALSE"
    ssh-keys       = "${var.ssh_username}:${var.ssh_public_key}"
  }
  ```

  > **Salida esperada:** Solo se define la entrada `ssh-keys` en metadata, sin activar OS Login.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Antes de cerrar la VM**, crea un bloque `labels` con la clave `env` y valor `var.env`.

  > **Nota:** Apóyate en la [referencia oficial de `google_compute_instance`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance); el bloque va al mismo nivel que `metadata`.
  {: .lab-note .info .compact}

  > **Salida esperada:** La instancia contiene `labels = { env = var.env }` dentro de `google_compute_instance.vm`.
  {: .lab-note .output .compact}

### Tarea 3.2. Declarar variables y outputs de Compute Engine

Parametrizar la instancia y publicar su enlace y su dirección IPv4 pública.

- {% include step_label.html %} En **modules/vm/variables.tf**, declara identificadores de proyecto, zona, nombre y subred.

  ```hcl
  variable "project_id"           { type = string }
  variable "zone"                 { type = string }
  variable "vm_name"              { type = string }
  variable "subnetwork_self_link" { type = string }
  ```

  > **Salida esperada:** Se declaran cuatro entradas de identidad y conectividad.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo, agrega parámetros para tipo, imagen y características de disco.

  ```hcl
  variable "machine_type"       { type = string }
  variable "image"              { type = string }
  variable "boot_disk_size_gb"  { type = number }
  variable "boot_disk_type"     { type = string }
  ```

  > **Salida esperada:** La capacidad y el sistema operativo son configurables desde el root module.
  {: .lab-note .output .compact}

- {% include step_label.html %} Al final del archivo declara usuario y llave pública SSH, y etiqueta de entorno.

  ```hcl
  variable "ssh_username"   { type = string }
  variable "ssh_public_key" { type = string }
  variable "env" {
    type    = string
    default = "dev"
  }
  ```

  > **Salida esperada:** El módulo declara once variables en total, incluida la etiqueta opcional `env`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **modules/vm/outputs.tf**, publica la referencia de la instancia y su IP externa.

  ```hcl
  output "instance_self_link" {
    description = "Referencia a la VM"
    value       = google_compute_instance.vm.self_link
  }
  output "external_ip" {
    description = "IPv4 pública efímera"
    value       = google_compute_instance.vm.network_interface[0].access_config[0].nat_ip
  }
  ```

  > **Salida esperada:** El módulo VM expone `instance_self_link` y `external_ip`.
  {: .lab-note .output .compact}

{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## 🧩 Tarea 4. Integrar ambos módulos desde la configuración principal — 10 min

Definirás los parámetros del root module, pasarás al módulo VM un output del módulo Network, guardarás la IP permitida y configurarás la llave SSH sin pegar manualmente su contenido en los archivos.

### Tarea 4.1. Configurar el proveedor y las variables del root module

Definir la versión del proveedor de Google y las variables que consumirá el proyecto principal.

- {% include step_label.html %} En **main/providers.tf**, configura el proveedor oficial de Google Cloud.

  ```hcl
  terraform {
    required_version = ">= 1.6.0"
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
    zone    = var.zone
  }
  ```

  > **Salida esperada:** Terraform requiere un proveedor Google de la serie 6.x y un proyecto, región y zona configurables.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **main/variables.tf**, declara los parámetros generales del proyecto y ubicación.

  ```hcl
  variable "project_id" { type = string }
  variable "region" {
    type    = string
    default = "us-central1"
  }
  variable "zone" {
    type    = string
    default = "us-central1-a"
  }
  ```

  > **Salida esperada:** El proyecto es obligatorio y la región y zona tienen valores predeterminados.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo, agrega las variables de características de la VM.

  ```hcl
  variable "machine_type" {
    type    = string
    default = "e2-micro"
  }
  variable "image" {
    type    = string
    default = "debian-cloud/debian-12"
  }
  variable "boot_disk_size_gb" {
    type    = number
    default = 20
  }
  variable "boot_disk_type" {
    type    = string
    default = "pd-standard"
  }
  ```

  > **Salida esperada:** El root module define tamaño, imagen y disco de la VM.
  {: .lab-note .output .compact}

- {% include step_label.html %} Al final de **main/variables.tf**, declara usuario SSH y rango CIDR autorizado.

  ```hcl
  variable "ssh_username" {
    type    = string
    default = "gcpuser"
  }
  variable "allowed_ssh_cidr" {
    type        = string
    description = "IP pública autorizada en formato IPv4/32"
    validation {
      condition     = can(cidrhost(var.allowed_ssh_cidr, 0)) && endswith(var.allowed_ssh_cidr, "/32")
      error_message = "Indica una dirección IPv4 pública con máscara /32."
    }
  }
  ```

  > **Salida esperada:** Existe una validación para introducir el rango autorizado con máscara `/32`.
  {: .lab-note .output .compact}

### Tarea 4.2. Enlazar los módulos Network y VM

Consumir los módulos locales mediante sus rutas, variables y referencias de output.

- {% include step_label.html %} En **main/main.tf**, inserta el bloque de consumo del módulo `network`.

  ```hcl
  module "network" {
    source            = "../modules/network"
    project_id        = var.project_id
    region            = var.region
    network_name      = "lab2-vpc"
    subnetwork_name   = "lab2-subnet"
    ip_cidr_range     = "10.20.1.0/24"
    allowed_ssh_cidr  = var.allowed_ssh_cidr
  }
  ```

  > **Salida esperada:** El root module apunta a `../modules/network` y define los datos de red.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Debajo del módulo Network**, inicia el módulo VM con sus campos básicos.

  ```hcl
  module "vm" {
    source            = "../modules/vm"
    project_id        = var.project_id
    zone              = var.zone
    vm_name           = "lab2-vm"
    machine_type      = var.machine_type
    image             = var.image
    boot_disk_size_gb = var.boot_disk_size_gb
    boot_disk_type    = var.boot_disk_type
    ssh_username      = var.ssh_username
    ssh_public_key    = file(pathexpand("~/.ssh/gcp-demo.pub"))
    env               = "dev"
  }
  ```

  > **Salida esperada:** El módulo VM existe y lee el archivo local de llave pública sin copiarlo en `terraform.tfvars`.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Dentro de `module "vm"`, antes de la llave de cierre**, agrega la entrada `subnetwork_self_link` usando el output de `module.network`.

  > **Nota:** Lee la [documentación oficial sobre módulos](https://developer.hashicorp.com/terraform/language/modules/syntax) y utiliza la referencia `module.network.subnetwork_self_link`.
  {: .lab-note .info .compact}

  > **Salida esperada:** El módulo VM recibe `subnetwork_self_link = module.network.subnetwork_self_link`; Terraform infiere la dependencia.
  {: .lab-note .output .compact}

- {% include step_label.html %} Guarda **main/main.tf** y comprueba que aparecen exactamente dos bloques `module`.

  ```bash
  grep -nE '^module "(network|vm)"' main/main.tf
  ```

  > **Salida esperada:** Se muestran `module "network"` y `module "vm"` una vez cada uno.
  {: .lab-note .output .compact}

### Tarea 4.3. Configurar variables reales, llave y outputs principales

Recuperar el proyecto activo, generar la llave si falta y proporcionar únicamente datos específicos del participante.

- {% include step_label.html %} En **Git Bash**, ejecuta este comando para entrar al directorio raíz de Terraform.

  ```bash
  cd main
  pwd
  ```

  > **Salida esperada:** La ruta actual termina en `/lab2/main`; todos los comandos Terraform posteriores se ejecutan aquí.
  {: .lab-note .output .compact}

- {% include step_label.html %} Verifica si existe tu llave SSH; si falta, genera una llave Ed25519 personal.

  > **Importante:** No uses la contraseña compartida del material original. Si `ssh-keygen` solicita passphrase, crea una personal o acepta vacío solo para este laboratorio controlado.
  {: .lab-note .important .compact}

  ```bash
  mkdir -p ~/.ssh
  if [ ! -f ~/.ssh/gcp-demo.pub ]; then ssh-keygen -t ed25519 -C "terraform-lab2" -f ~/.ssh/gcp-demo; fi
  test -s ~/.ssh/gcp-demo.pub && echo "Llave pública disponible"
  ```

  > **Salida esperada:** La llave pública existe en `~/.ssh/gcp-demo.pub` y la privada en `~/.ssh/gcp-demo`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Obtén tu proyecto e IP pública; usa el navegador si la consulta de IP no responde.

  > **Importante:** Si tu red usa VPN o cambia de salida a Internet, usa la IP pública desde la que saldrá el cliente SSH.
  {: .lab-note .important .compact}

  ```bash
  gcloud config get-value project
  curl -4 -fsS https://api.ipify.org; echo
  ```

  > **Salida esperada:** Se obtienen un PROJECT_ID y una dirección IPv4 pública de tu conexión.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **main/terraform.tfvars**, establece los valores reales de `project_id` y `allowed_ssh_cidr`.

  > **Advertencia:** Reemplaza ambos marcadores. El `/32` debe corresponder a tu IP pública; no pegues aquí el contenido de llaves privadas.
  {: .lab-note .warning .compact}

  ```hcl
  project_id       = "TU_PROJECT_ID"
  allowed_ssh_cidr = "TU_IP_PUBLICA/32"
  ```

  > **Salida esperada:** El archivo contiene el ID real y tu dirección pública con `/32`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **main/outputs.tf**, publica la referencia de la VM que crea el módulo.

  ```hcl
  output "vm_self_link" {
    description = "Enlace de la instancia creada"
    value       = module.vm.instance_self_link
  }
  ```

  > **Salida esperada:** Se declara el output `vm_self_link`.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Debajo de `vm_self_link`**, crea un output llamado `vm_external_ip` que devuelva `module.vm.external_ip`.

  > **Nota:** Consulta [Outputs de Terraform](https://developer.hashicorp.com/terraform/language/values/outputs). Es un bloque independiente, no una línea dentro del output anterior.
  {: .lab-note .info .compact}

  > **Salida esperada:** `main/outputs.tf` expone `vm_external_ip` con la IP pública de `module.vm`.
  {: .lab-note .output .compact}

{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

## 🚀 Tarea 5. Validar, desplegar y acceder a la máquina virtual — 9 min

Inicializarás Terraform dentro del root module, corregirás formato y sintaxis, revisarás el plan y crearás los recursos para comprobar la conexión SSH desde Git Bash.

### Tarea 5.1. Inicializar, formatear, validar, planificar y aplicar

Ejecutar el ciclo básico de Terraform comprobando recursos y errores antes del despliegue real.

- {% include step_label.html %} En **Git Bash**, situado en **lab2/main**, inicializa los proveedores y módulos.

  > **Salida esperada:** Termina con `Terraform has been successfully initialized!`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Aplica formato HCL a los archivos del root module y a ambos módulos locales.

  ```bash
  terraform fmt -recursive ..
  ```

  > **Salida esperada:** Terraform informa de los archivos formateados o termina sin errores si ya tenían formato correcto.
  {: .lab-note .output .compact}

- {% include step_label.html %} Valida la estructura del proyecto.

  > **Salida esperada:** Se muestra `Success! The configuration is valid.`
  {: .lab-note .output .compact}

- {% include step_label.html %} Genera el plan y verifica el número de recursos antes de desplegar.

  > **Importante:** Se esperan cuatro recursos nuevos: VPC, subred, Firewall y VM. Si aparece una cantidad distinta o un error, revisa variables y archivos.
  {: .lab-note .important .compact}

  ```bash
  terraform plan -out=lab2.tfplan
  ```

  > **Salida esperada:** El resumen indica `Plan: 4 to add, 0 to change, 0 to destroy`, si el proyecto estaba limpio.
  {: .lab-note .output .compact}

- {% include step_label.html %} Aplica el plan guardado y espera la confirmación de creación.

  > **Advertencia:** Esta acción crea recursos facturables. `terraform apply lab2.tfplan` aplica el plan guardado sin pedir `yes` de nuevo.
  {: .lab-note .warning .compact}

  ```bash
  terraform apply lab2.tfplan
  ```

  > **Salida esperada:** Aparece `Apply complete! Resources: 4 added, 0 changed, 0 destroyed.`
  {: .lab-note .output .compact}

### Tarea 5.2. Comprobar IP y acceso SSH

Verificar desde Git Bash que la VM está disponible y acepta la llave privada correspondiente a la pública instalada.

- {% include step_label.html %} Consulta la IP externa desde el output del módulo raíz.

  ```bash
  terraform output -raw vm_external_ip
  ```

  > **Salida esperada:** Se imprime una IPv4 pública asignada a `lab2-vm`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Conéctate desde **Git Bash** mediante la llave privada; revisa la huella SSH antes de aceptarla.

  > **Importante:** Si tu organización obliga OS Login o bloquea metadatos SSH, esta variante del laboratorio debe adaptarse a su política.
  {: .lab-note .important .compact}

  ```bash
  VM_IP="$(terraform output -raw vm_external_ip)"
  ssh -i ~/.ssh/gcp-demo -o IdentitiesOnly=yes "gcpuser@$VM_IP"
  ```

  > **Salida esperada:** El prompt cambia al shell Linux de `lab2-vm`; confirma la huella solo si corresponde a la VM creada.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Dentro de la VM Linux**, consulta el sistema operativo y después cierra SSH.

  ```bash
  cat /etc/os-release | head -n 3
  uname -m
  exit
  ```

  > **Salida esperada:** Aparece información de Debian, arquitectura de la VM y regresas a Git Bash de Windows.
  {: .lab-note .output .compact}

{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

---

## 🧹 Tarea 6. Verificar el estado y eliminar los recursos — 5 min

Revisarás los outputs y la consistencia del estado local, destruirás los recursos para evitar costos y conservarás el código modular para futuras prácticas.

### Tarea 6.1. Inspeccionar los recursos administrados

Confirmar qué recursos administra el estado actual y que no existan cambios pendientes tras el despliegue.

- {% include step_label.html %} En **Git Bash**, desde **lab2/main**, enumera los recursos incluidos en el estado.

  ```bash
  terraform state list
  ```

  > **Salida esperada:** Se listan cuatro direcciones: una VM y tres recursos de red dentro de los módulos.
  {: .lab-note .output .compact}

- {% include step_label.html %} Consulta los outputs y comprueba que Terraform no tiene cambios pendientes.

  > **Nota:** El código de salida 0 significa sin cambios; 2 significa cambios previstos y 1 significa error.
  {: .lab-note .info .compact}

  ```bash
  terraform output
  terraform plan -detailed-exitcode
  ```

  > **Salida esperada:** Los outputs contienen la VM e IP pública y el plan termina con `No changes.`
  {: .lab-note .output .compact}

### Tarea 6.2. Destruir y verificar la limpieza

Eliminar los recursos creados sin borrar la estructura ni el código de los módulos locales.

- {% include step_label.html %} Ejecuta **terraform destroy** desde `lab2/main` y confirma escribiendo `yes` cuando se solicite.

  > **Advertencia:** Elimina únicamente los recursos administrados por el estado de este directorio. No ejecutes el comando en `lab1`.
  {: .lab-note .warning .compact}

  ```bash
  terraform destroy
  ```

  > **Salida esperada:** Terraform muestra `Destroy complete! Resources: 4 destroyed.` en un despliegue completo.
  {: .lab-note .output .compact}

- {% include step_label.html %} Comprueba que Terraform ya no gestiona recursos de esta práctica.

  ```bash
  terraform state list
  ```

  > **Salida esperada:** No aparece ningún recurso activo en la lista del estado.
  {: .lab-note .output .compact}

- {% include step_label.html %} Elimina el plan temporal guardado; conserva los archivos `.tf`, la llave y la estructura modular.

  ```bash
  rm -f lab2.tfplan
  ls -1 *.tf
  ```

  > **Salida esperada:** El plan guardado desaparece; el código fuente de `main` permanece disponible.
  {: .lab-note .output .compact}

{% capture r6 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r6 %}
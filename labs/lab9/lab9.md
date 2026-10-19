---
layout: lab
title: "Práctica 9: Refactorizar un proyecto real monolítico en modular y validado"
permalink: /lab9/lab9/
images_base: /labs/lab9/img
duration: "40 minutos"
objective:
  - "Refactorizar infraestructura desplegada desde un root monolítico hacia módulos reutilizables de Network y Compute, conservando el estado mediante moved blocks y comprobando que no haya reemplazos de recursos."
prerequisites:
  - "Prácticas 1 a 8 completadas; directorio terraform-gcp-intermediate creado en el Escritorio de Windows."
  - "VM Windows con VS Code, Git Bash, Terraform >= 1.5 y Google Cloud CLI instalados."
  - "Proyecto GCP con facturación, cuota para una VM, Compute Engine API habilitada y permisos para VPC, subredes, Firewall y Compute Engine."
  - "Cuenta gcloud y credenciales Application Default Credentials (ADC) configuradas para el proyecto del curso."
introduction:
  - "Crearás una VPC, una subred, una regla SSH limitada a IAP y una VM Debian desde un proyecto Terraform monolítico. Extraerás módulos Network y Compute sin cambiar los recursos físicos; registrarás moved blocks para trasladar sus direcciones de estado y comprobarás mediante terraform plan que no es necesario recrearlos. Finalmente documentarás y destruirás solo los recursos lab9."
slug: lab9
lab_number: 9
final_result: >
  Habrás refactorizado cuatro recursos de un proyecto Terraform monolítico a módulos Network y Compute,
  conservando el mismo estado local y verificando que el movimiento de direcciones no exige reemplazos.
  También habrás documentado el proyecto, comprobado sus recursos en GCP y limpiado la infraestructura mediante Terraform.
notes:
  - "Todos los comandos locales se ejecutan desde Git Bash integrado en VS Code, dentro de terraform-gcp-intermediate/lab9, salvo indicación contraria."
  - "Esta práctica usa un único root module y un estado local independiente. No reutiliza el backend GCS de la Práctica 7 ni despliega desde una copia del monolito."
  - "El Firewall permite SSH solo desde el rango IAP 35.235.240.0/20; las reglas de red no reemplazan los permisos IAM para conectarse."
  - "Durante la refactorización, detén el procedimiento si terraform plan muestra destrucciones o reemplazos físicos no esperados."
  - "La duración total de 65 minutos supone autenticación, APIs, cuotas y permisos preparados; el aprovisionamiento puede variar."
  - "Completa cada ejercicio de construcción dirigida antes de inicializar o validar la configuración modular."
references:
  - text: "Terraform: refactorización con moved blocks"
    url: https://developer.hashicorp.com/terraform/language/modules/develop/refactoring
  - text: "Terraform: módulos"
    url: https://developer.hashicorp.com/terraform/language/modules
  - text: "Terraform: moved block"
    url: https://developer.hashicorp.com/terraform/language/moved
  - text: "Terraform: validación de variables"
    url: https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules
  - text: "Google Cloud: reglas de Firewall"
    url: https://cloud.google.com/firewall/docs/firewalls
prev: /lab8/lab8/
next: /lab10/lab10/
---

---

## 🔎 Tarea 1. Preparar un proyecto monolítico — 9 min

Crear un root Terraform independiente, declarar cuatro recursos y reconocer por qué un archivo único puede dificultar el mantenimiento.


### Tarea 1.1. Crear lab9 y comprobar el entorno

Usar la carpeta principal del curso y verificar el proyecto antes de escribir infraestructura.


- {% include step_label.html %} En **VS Code**, selecciona **File > Open Folder...**, abre **Escritorio > terraform-gcp-intermediate** y después **Terminal > New Terminal**. Usa la carpeta existente, incluso si Windows tiene el Escritorio redirigido a OneDrive.

  **Acciones o requisitos:**

  1. En el menú superior, selecciona **File > Open Folder...** y elige la carpeta central del curso.
  2. Confirma en el Explorador que aparecen `lab1` a `lab8`.
  3. Selecciona **Terminal > New Terminal** y cambia el perfil a **Git Bash** si se abre otra terminal.
  4. No modifiques los archivos ni estados de ninguna práctica anterior.

  > **Salida esperada:** La raíz del curso aparece abierta y la terminal utiliza Git Bash.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **Git Bash**, crea `lab9` y sus archivos; el `pwd` confirma el directorio de trabajo.

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab9"
  cd "$DESKTOP/terraform-gcp-intermediate/lab9"
  touch providers.tf variables.tf main.tf outputs.tf terraform.tfvars .gitignore
  pwd
  ```

  > **Salida esperada:** La ruta termina en `/terraform-gcp-intermediate/lab9`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba Terraform, la cuenta activa y el proyecto de GCP. Corrige el proyecto antes de continuar si es diferente al asignado.

  ```bash
  terraform version
  gcloud auth list --filter="status:ACTIVE" --format="value(account)"
  PROJECT_ID="$(gcloud config get-value project)"
  echo "Proyecto actual: $PROJECT_ID"
  gcloud auth application-default print-access-token >/dev/null && echo "ADC disponible"
  gcloud services list --enabled --project="$PROJECT_ID" --filter="config.name=compute.googleapis.com" --format="value(config.name)"
  ```

  > **Nota:** Si ADC falla, ejecuta `gcloud auth application-default login` antes de `terraform plan`. Si la API no aparece, solicita al instructor habilitar `compute.googleapis.com`.
  {: .lab-note .info .compact}

  > **Salida esperada:** La cuenta y el proyecto coinciden con los del curso; se muestra `ADC disponible` y la API `compute.googleapis.com` está habilitada.
  {: .lab-note .output .compact}


### Tarea 1.2. Escribir configuración monolítica y analizarla

Declarar Provider, entradas, recursos y outputs directamente en el root module.


- {% include step_label.html %} En `lab9/providers.tf`, escribe el bloque de versión requerida y Google Provider. Comprueba que la autenticación use el proyecto de `terraform.tfvars`, no un ID fijo.

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

  > **Salida esperada:** Provider configurado con variables `project_id`, `region` y `zone`.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `lab9/variables.tf`, declara las tres variables. Después guarda `terraform.tfvars` con el ID real de tu proyecto.

  ```hcl
  variable "project_id" {
    description = "ID del proyecto de laboratorio"
    type        = string
  }
  
  variable "region" {
    description = "Región para la subred"
    type        = string
    default     = "us-central1"
  }
  
  variable "zone" {
    description = "Zona para la VM"
    type        = string
    default     = "us-central1-a"
  }
  ```

  > **Nota:** En `terraform.tfvars` escribe `project_id = "TU_PROJECT_ID"` y sustitúyelo por el valor mostrado por gcloud.
  {: .lab-note .info .compact}

  > **Salida esperada:** Los archivos contienen variables válidas y el Project ID correcto.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **VS Code**, abre `lab9/main.tf` y pega los cuatro recursos monolíticos. Observa las referencias a VPC y subred, pero **no los despliegues todavía**.

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = "lab9-vpc"
    auto_create_subnetworks = false
  }
  
  resource "google_compute_subnetwork" "subnet" {
    name          = "lab9-subnet"
    ip_cidr_range = "10.90.0.0/24"
    region        = var.region
    network       = google_compute_network.vpc.id
  }
  
  resource "google_compute_firewall" "ssh_iap" {
    name          = "lab9-allow-ssh-iap"
    network       = google_compute_network.vpc.name
    direction     = "INGRESS"
    source_ranges = ["35.235.240.0/20"]
    target_tags   = ["ssh"]
    allow {
      protocol = "tcp"
      ports    = ["22"]
    }
  }
  
  resource "google_compute_instance" "vm" {
    name         = "lab9-vm"
    machine_type = "e2-micro"
    zone         = var.zone
    tags         = ["ssh"]
    labels       = { env = "lab", app = "refactor" }
  
    boot_disk {
      initialize_params {
        image = "projects/debian-cloud/global/images/family/debian-12"
      }
    }
  
    network_interface {
      subnetwork = google_compute_subnetwork.subnet.self_link
      access_config {}
    }
    metadata = {
      enable-oslogin = "TRUE"
    }
  }
  ```

  > **Importante:** La regla SSH está limitada a IAP; no debes sustituir `35.235.240.0/20` por `0.0.0.0/0`.
  {: .lab-note .important .compact}

  > **Salida esperada:** Se declaran VPC, subred, firewall y VM en el mismo archivo.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **VS Code**, abre `lab9/outputs.tf` y agrega las salidas de red y cómputo. Después, crea las exclusiones en `.gitignore` para no publicar estado ni archivos temporales.

  ```hcl
  output "vpc_name" {
    value = google_compute_network.vpc.name
  }
  output "vm_name" {
    value = google_compute_instance.vm.name
  }
  output "vm_internal_ip" {
    value = google_compute_instance.vm.network_interface[0].network_ip
  }
  ```

  En el archivo `lab9/.gitignore`, escribe exactamente estas líneas:

  ```gitignore
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  *.tfvars
  !*.tfvars.example
  ```

  > **Nota:** Conserva `.terraform.lock.hcl` para fijar las versiones de proveedores. El archivo `terraform.tfvars` seguirá disponible localmente, pero Git no lo incluirá.
  {: .lab-note .info .compact}

  > **Salida esperada:** Outputs presentes y `.gitignore` evita publicar estado y variables locales.
  {: .lab-note .output .compact}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---


## 🔎 Tarea 2. Desplegar y registrar el estado inicial — 10 min

Aplicar el monolito una sola vez para crear recursos reales y capturar las direcciones actuales del estado.


### Tarea 2.1. Inicializar y validar el monolito

Realizar comprobaciones técnicas antes de crear recursos facturables.


- {% include step_label.html %} Completa `lab9/terraform.tfvars` con la línea indicada. Usa el valor real de tu proyecto, sin corchetes ni marcadores.

  ```hcl
  project_id = "TU_PROJECT_ID"
  ```

  > **Salida esperada:** El archivo `terraform.tfvars` contiene un Project ID válido.
  {: .lab-note .output .compact}


- {% include step_label.html %} Inicializa el directorio raíz y normaliza los archivos HCL. No cambies de carpeta.

  ```bash
  terraform init
  terraform fmt -recursive
  terraform validate
  ```

  > **Salida esperada:** `terraform validate` muestra `Success! The configuration is valid.`
  {: .lab-note .output .compact}


- {% include step_label.html %} Genera un plan previo al despliegue y examina el resumen. Deben aparecer exactamente cuatro recursos nuevos.

  ```bash
  terraform plan -out=monolith.tfplan
  terraform show monolith.tfplan
  ```

  > **Salida esperada:** El plan informa `Plan: 4 to add, 0 to change, 0 to destroy`.
  {: .lab-note .output .compact}


### Tarea 2.2. Crear y registrar direcciones de recursos

Guardar evidencia del estado original para identificar los movimientos durante el refactoring.


- {% include step_label.html %} Aplica el plan previamente revisado desde `lab9`; espera a que finalice la creación de Compute Engine.

  ```bash
  terraform apply monolith.tfplan
  ```

  > **Salida esperada:** `Apply complete! Resources: 4 added, 0 changed, 0 destroyed.`
  {: .lab-note .output .compact}


- {% include step_label.html %} Enumera las direcciones que Terraform administra y guárdalas como línea base de comparación.

  ```bash
  terraform state list
  terraform state list > state-before.txt
  terraform output
  ```

  > **Salida esperada:** `state-before.txt` registra cuatro recursos sin prefijos `module.`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Consulta la VPC y la VM desde gcloud; evita alterar recursos manualmente porque el estado debe conservar correspondencia.

  ```bash
  gcloud compute networks describe lab9-vpc --project="$PROJECT_ID" --format="value(name)"
  gcloud compute instances describe lab9-vm --project="$PROJECT_ID" --zone="us-central1-a" --format="value(name)"
  ```

  > **Salida esperada:** Ambos nombres aparecen sin errores y corresponden a recursos del estado local.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---


## 🔎 Tarea 3. Construir módulos reutilizables — 18 min

Extraer red y cómputo del monolito sin cambiar nombres ni atributos de los recursos desplegados.


### Tarea 3.1. Preparar estructura de módulos

Crear subdirectorios para Network y Compute y conservar respaldo del monolito fuera de los archivos `.tf` activos.


- {% include step_label.html %} Crea las carpetas `modules/network` y `modules/compute`, manteniendo `lab9` como único root Terraform.

  ```bash
  mkdir -p modules/network modules/compute
  pwd
  ```

  > **Salida esperada:** Existen dos carpetas bajo `lab9/modules/`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Crea los tres archivos HCL en cada módulo y revisa que VS Code los muestre en el explorador.

  ```bash
  touch modules/network/{main,variables,outputs}.tf
  touch modules/compute/{main,variables,outputs}.tf
  ```

  > **Salida esperada:** Cada módulo contiene `main.tf`, `variables.tf` y `outputs.tf`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Guarda una copia de la configuración monolítica como evidencia en un archivo **no .tf**. Terraform no debe interpretar esta copia como recursos duplicados.

  ```bash
  cp main.tf monolith-before.txt
  cp outputs.tf outputs-before.txt
  ```

  > **Salida esperada:** Hay archivos de respaldo `.txt`; el root conserva un único `main.tf` activo.
  {: .lab-note .output .compact}


### Tarea 3.2. Implementar el módulo Network

Extraer VPC, subred y firewall respetando los atributos originales, añadiendo entradas tipadas y outputs.


- {% include step_label.html %} Abre `modules/network/variables.tf` y declara las entradas `network_name`, `subnet_name`, `region`, `subnet_cidr` y `allow_ssh_iap`.

  ```hcl
  variable "network_name" {
    type = string
  }
  variable "subnet_name" {
    type = string
  }
  variable "region" {
    type = string
  }
  variable "subnet_cidr" {
    type = string
  }
  variable "allow_ssh_iap" {
    type    = bool
    default = true
  }
  ```

  > **Salida esperada:** Cinco variables están disponibles para el módulo Network.
  {: .lab-note .output .compact}


- {% include step_label.html %} En el mismo archivo, agrega **dentro de** `variable "subnet_cidr"`, antes de su `}`, una validación dirigida para un CIDR válido.

  **Acciones o requisitos:**

  - Usa `validation { condition = can(cidrhost(var.subnet_cidr, 0)) ... }`.
  - Incluye un `error_message` que explique el formato requerido.
  - Mantén el bloque `validation` dentro de la variable `subnet_cidr`.
  - Referencia: [Terraform variable validation](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules).

  > **Salida esperada:** La variable rechazará un CIDR inválido durante `terraform plan`.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `modules/network/main.tf`, declara la VPC y la subred con los nombres anteriores. Conserva el CIDR y la región para no forzar cambios.

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = var.network_name
    auto_create_subnetworks = false
  }
  
  resource "google_compute_subnetwork" "subnet" {
    name          = var.subnet_name
    ip_cidr_range = var.subnet_cidr
    region        = var.region
    network       = google_compute_network.vpc.id
  }
  ```

  > **Salida esperada:** Los recursos Network tienen los mismos nombres lógicos `vpc` y `subnet` dentro del módulo.
  {: .lab-note .output .compact}


- {% include step_label.html %} Debajo de la subred, agrega el firewall del módulo. Conservar los atributos originales evita diferencias de red durante la migración.

  ```hcl
  resource "google_compute_firewall" "ssh_iap" {
    count         = var.allow_ssh_iap ? 1 : 0
    name          = "lab9-allow-ssh-iap"
    network       = google_compute_network.vpc.name
    direction     = "INGRESS"
    source_ranges = ["35.235.240.0/20"]
    target_tags   = ["ssh"]
    allow {
      protocol = "tcp"
      ports    = ["22"]
    }
  }
  ```

  > **Nota:** Como el firewall ahora usa `count`, su dirección destino tendrá `[0]` dentro del módulo.
  {: .lab-note .info .compact}

  > **Salida esperada:** `google_compute_firewall.ssh_iap[0]` representa la única regla de firewall.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `modules/network/outputs.tf`, crea un output `network_name` y deja preparado uno llamado `subnet_self_link` mediante construcción dirigida.

  **Acciones o requisitos:**

  - Debajo del output mostrado, declara `output "subnet_self_link"`.
  - Su `value` debe ser `google_compute_subnetwork.subnet.self_link`.
  - No escribas un string fijo; referencia el recurso que administra el módulo.

  ```hcl
  output "network_name" {
    value = google_compute_network.vpc.name
  }
  ```

  > **Salida esperada:** Network expone el nombre de VPC y el self-link de subred.
  {: .lab-note .output .compact}


### Tarea 3.3. Implementar el módulo Compute

Encapsular la instancia Debian y obtener el enlace a la subred como entrada proveniente de otro módulo.


- {% include step_label.html %} En `modules/compute/variables.tf`, declara las entradas de VM y construye la variable `subnet_self_link` siguiendo sus requisitos.

  **Acciones o requisitos:**

  - Declara adicionalmente `variable "subnet_self_link" { type = string }`.
  - Usa la entrada como identificador de subred, no un nombre fijo.
  - Referencia: [Modules](https://developer.hashicorp.com/terraform/language/modules).

  ```hcl
  variable "instance_name" { type = string }
  variable "machine_type" { type = string }
  variable "zone" { type = string }
  variable "tags" { type = list(string) }
  variable "labels" { type = map(string) }
  ```

  > **Salida esperada:** Compute dispone de seis entradas, incluyendo el enlace de subred.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `modules/compute/main.tf`, escribe la VM Debian con la misma configuración que la original, incluida su IP externa y OS Login.

  ```hcl
  resource "google_compute_instance" "vm" {
    name         = var.instance_name
    machine_type = var.machine_type
    zone         = var.zone
    tags         = var.tags
    labels       = var.labels
  
    boot_disk {
      initialize_params {
        image = "projects/debian-cloud/global/images/family/debian-12"
      }
    }
  
    network_interface {
      subnetwork = var.subnet_self_link
      access_config {}
    }
    metadata = {
      enable-oslogin = "TRUE"
    }
  }
  ```

  > **Salida esperada:** La VM usa el mismo nombre, zona, imagen, tipo, etiquetas, tags e IP externa que antes.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `modules/compute/outputs.tf`, publica nombre, IP interna e IP externa mediante expresiones que no dependan de la posición de otros recursos.

  ```hcl
  output "instance_name" {
    value = google_compute_instance.vm.name
  }
  output "internal_ip" {
    value = google_compute_instance.vm.network_interface[0].network_ip
  }
  output "external_ip" {
    value = try(google_compute_instance.vm.network_interface[0].access_config[0].nat_ip, null)
  }
  ```

  > **Salida esperada:** Compute expone el nombre de VM y ambas direcciones IP.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba en el explorador que el módulo Network contiene tres recursos y Compute uno; no ejecutes `terraform apply` desde `modules/`.

  **Acciones o requisitos:**

  - `modules/network/main.tf`: VPC, subred y firewall.
  - `modules/compute/main.tf`: VM.
  - Los dos módulos usan entradas y outputs definidos en sus carpetas.

  > **Salida esperada:** Hay cuatro recursos distribuidos en dos módulos, aún sin modificar el estado.
  {: .lab-note .output .compact}


- {% include step_label.html %} Consulta el respaldo `monolith-before.txt` y compara los nombres de recursos; no modifiques `main.tf` del root hasta la siguiente tarea.

  ```bash
  grep -E '^resource ' monolith-before.txt
  find modules -name '*.tf' -print
  ```

  > **Salida esperada:** El respaldo conserva los cuatro recursos iniciales y ambos módulos están disponibles.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---


## 🔎 Tarea 4. Refactorizar el root y conservar el estado — 15 min

Sustituir recursos monolíticos por llamadas a módulos y declarar movimientos explícitos de las direcciones originales.


### Tarea 4.1. Sustituir recursos por llamadas a módulos

Dejar en el root solo la orquestación; no inicializar otro directorio ni cambiar el backend.


- {% include step_label.html %} En `lab9/main.tf`, **reemplaza todo su contenido anterior** por las llamadas a los módulos. Es imprescindible retirar los recursos monolíticos del root para no duplicarlos.

  ```hcl
  module "network" {
    source        = "./modules/network"
    network_name  = "lab9-vpc"
    subnet_name   = "lab9-subnet"
    subnet_cidr   = "10.90.0.0/24"
    region        = var.region
    allow_ssh_iap = true
  }
  
  module "compute" {
    source           = "./modules/compute"
    instance_name    = "lab9-vm"
    machine_type     = "e2-micro"
    zone             = var.zone
    tags             = ["ssh"]
    labels           = { env = "lab", app = "refactor" }
    subnet_self_link = module.network.subnet_self_link
  }
  ```

  > **Salida esperada:** El root declara solo `module.network` y `module.compute`; no hay bloques `resource` en su `main.tf`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba la relación entre los módulos como construcción dirigida: `Compute` debe recibir el output real de `Network`, no una ruta escrita a mano.

  **Acciones o requisitos:**

  - En `module "network"`, `source` apunta a `./modules/network`.
  - En `module "compute"`, `source` apunta a `./modules/compute`.
  - El argumento `subnet_self_link` debe usar `module.network.subnet_self_link`.
  - Referencia: [Module outputs](https://developer.hashicorp.com/terraform/language/modules).

  > **Salida esperada:** La VM depende automáticamente de la subred exportada por Network.
  {: .lab-note .output .compact}


- {% include step_label.html %} Reescribe `lab9/outputs.tf` para consumir los outputs de ambos módulos; no dejes las referencias antiguas a `google_compute_instance.vm` del root.

  ```hcl
  output "vpc_name" {
    value = module.network.network_name
  }
  output "vm_name" {
    value = module.compute.instance_name
  }
  output "vm_internal_ip" {
    value = module.compute.internal_ip
  }
  output "vm_external_ip" {
    value = module.compute.external_ip
  }
  ```

  > **Salida esperada:** Los outputs del root apuntan exclusivamente a `module.network` y `module.compute`.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `lab9`, confirma que no existe un segundo directorio raíz `monolitico` o `modularizado/main`; el trabajo se realiza en la misma carpeta y con el estado ya creado.

  ```bash
  pwd
  terraform workspace show
  ls -la terraform.tfstate main.tf modules/network/main.tf modules/compute/main.tf
  ```

  > **Salida esperada:** `terraform.tfstate` permanece en el root `lab9` y el workspace activo es `default`.
  {: .lab-note .output .compact}


### Tarea 4.2. Declarar movimientos del estado

Relacionar cada recurso monolítico con la nueva dirección exacta de su módulo.


- {% include step_label.html %} En **Git Bash**, ejecuta `touch moved.tf` dentro de `lab9`. Después, abre `lab9/moved.tf` en **VS Code** y agrega estos tres movimientos para red y VM. Cada bloque `from` apunta al recurso original y `to` al nuevo destino.

  ```hcl
  moved {
    from = google_compute_network.vpc
    to   = module.network.google_compute_network.vpc
  }
  
  moved {
    from = google_compute_subnetwork.subnet
    to   = module.network.google_compute_subnetwork.subnet
  }
  
  moved {
    from = google_compute_instance.vm
    to   = module.compute.google_compute_instance.vm
  }
  ```

  > **Salida esperada:** Tres movimientos relacionan VPC, subred y VM con sus nuevos módulos.
  {: .lab-note .output .compact}


- {% include step_label.html %} Debajo de esos bloques, crea mediante construcción dirigida el movimiento del firewall que ahora usa `count`.

  **Acciones o requisitos:**

  - Origen `google_compute_firewall.ssh_iap`.
  - Destino `module.network.google_compute_firewall.ssh_iap[0]`.
  - Utiliza un bloque `moved { from = ... to = ... }`.
  - No cambies la cantidad de recursos ni el `source_ranges`.
  - Referencia: [Refactoring](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring).

  > **Salida esperada:** Existen cuatro `moved` blocks, uno por recurso original.
  {: .lab-note .output .compact}


- {% include step_label.html %} Enumera las direcciones originales de `state-before.txt` y compáralas con `from` en `moved.tf`; todavía no alteres el estado mediante `terraform state mv`.

  ```bash
  cat state-before.txt
  cat moved.tf
  ```

  > **Importante:** No combines `terraform state mv` y `moved` blocks: esta práctica demostrará exclusivamente la migración declarativa.
  {: .lab-note .important .compact}

  > **Salida esperada:** Cada una de las cuatro direcciones originales aparece en un `from` único.
  {: .lab-note .output .compact}


### Tarea 4.3. Planificar y aplicar el refactoring

Verificar que los movimientos se representen como cambios de dirección, sin destruir recursos reales.


- {% include step_label.html %} Inicializa otra vez el root para instalar módulos locales y después comprueba el formato y la validez de toda la configuración.

  ```bash
  terraform init
  terraform fmt -recursive
  terraform validate
  ```

  > **Salida esperada:** Inicialización completada y `Success! The configuration is valid.`
  {: .lab-note .output .compact}


- {% include step_label.html %} Ejecuta un plan **sin auto-approve** y revísalo antes de cualquier escritura del estado.

  ```bash
  terraform plan -out=refactor.tfplan
  ```

  > **Importante:** Si el plan propone `destroy`, `-/+` o cambios inesperados, NO ejecutes apply. Compara nombres, propiedades y destinos `moved`.
  {: .lab-note .important .compact}

  > **Salida esperada:** El plan presenta `has moved to` para los recursos y `Plan: 0 to add, 0 to change, 0 to destroy`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Solo si el plan no propone modificaciones físicas, aplica el plan guardado y comprueba el nuevo listado de direcciones del estado.

  ```bash
  terraform apply refactor.tfplan
  terraform state list
  ```

  > **Salida esperada:** Cuatro recursos aparecen ahora bajo `module.network` y `module.compute`, con firewall `[0]`.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---


## 🔎 Tarea 5. Validar y documentar el proyecto — 9 min

Comparar evidencia previa y posterior, verificar que GCP conserve los recursos y documentar el uso de los módulos.


### Tarea 5.1. Comparar estado, recursos y outputs

Distinguir cambio de direcciones Terraform de cambios reales de infraestructura.


- {% include step_label.html %} Guarda el nuevo listado del estado y compáralo con el registro original, sin modificar manualmente los archivos `tfstate`.

  ```bash
  terraform state list > state-after.txt
  cat state-before.txt
  cat state-after.txt
  ```

  > **Salida esperada:** Antes: cuatro recursos raíz. Después: tres recursos de Network y uno de Compute.
  {: .lab-note .output .compact}


- {% include step_label.html %} Ejecuta un plan sin cambios para comprobar que la configuración modular coincide con la infraestructura existente.

  ```bash
  terraform plan
  ```

  > **Salida esperada:** Terraform indica `No changes. Your infrastructure matches the configuration.`
  {: .lab-note .output .compact}


- {% include step_label.html %} Valida los outputs y la presencia de VM y VPC en GCP; la refactorización no debió crear una segunda VM.

  ```bash
  terraform output
  gcloud compute networks describe lab9-vpc --project="$PROJECT_ID" --format="value(name)"
  gcloud compute instances list --project="$PROJECT_ID" --filter="name=lab9-vm" --format="table(name,zone,status)"
  ```

  > **Salida esperada:** Existe una sola VM `lab9-vm`, la VPC `lab9-vpc` y los outputs son consistentes.
  {: .lab-note .output .compact}


### Tarea 5.2. Documentar módulos y uso

Dejar instrucciones para que otro instructor o participante reproduzca la configuración sin adivinar directorios.


- {% include step_label.html %} En `lab9/README.md`, crea una descripción concreta del proyecto, la estructura de módulos y sus comandos básicos.

  ```markdown
  # Lab 9 — Refactorización Terraform
  
  Este proyecto administra una VPC, una subred, una regla IAP y una VM en Google Cloud.
  Los recursos se crearon inicialmente en el root y luego se trasladaron a módulos con moved blocks.
  
  ## Estructura
  - `main.tf`: orquestación de módulos Network y Compute.
  - `modules/network/`: red, subred y regla de firewall.
  - `modules/compute/`: instancia Compute Engine.
  - `moved.tf`: correspondencia de direcciones originales y modulares.
  
  ## Uso
  1. Editar `terraform.tfvars` con el ID del proyecto.
  2. Ejecutar `terraform init`, `terraform validate`, `terraform plan`.
  3. Revisar el plan antes de ejecutar `terraform apply`.
  4. Destruir solo desde la raíz `lab9` con `terraform destroy`.
  
  ## Seguridad
  SSH está restringido a IAP; el ejercicio no administra contraseñas.
  ```

  > **Salida esperada:** README explica organización, operación, refactorización y seguridad.
  {: .lab-note .output .compact}


- {% include step_label.html %} Completa la documentación de los inputs y outputs mediante una tabla sencilla debajo de `## Estructura` en el README.

  **Acciones o requisitos:**

  - Registra entradas `project_id`, `region` y `zone`.
  - Registra salidas `vpc_name`, `vm_name`, `vm_internal_ip` y `vm_external_ip`.
  - Aclara que el estado es local de este laboratorio y no debe publicarse.

  > **Salida esperada:** Otro participante puede reconocer variables, salidas y restricciones sin inspeccionar cada módulo.
  {: .lab-note .output .compact}


- {% include step_label.html %} Ejecuta la última revisión antes de la limpieza y comprueba que no haya recursos raíz duplicados.

  ```bash
  terraform fmt -check -recursive
  terraform validate
  terraform state list
  ```

  > **Salida esperada:** Formato y validación correctos; el listado contiene cuatro direcciones modulares.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

---


## 🔎 Tarea 6. Destruir infraestructura y comprobar limpieza — 4 min

Retirar los cuatro recursos administrados por el root modular y verificar que no existan recursos residuales del laboratorio.


### Tarea 6.1. Destruir desde el root refactorizado

Eliminar desde el mismo estado que administró tanto el monolito como los módulos.


- {% include step_label.html %} Comprueba que ejecutas los comandos desde `lab9` y que el estado enumera los recursos del ejercicio, no los de laboratorios anteriores.

  ```bash
  pwd
  terraform state list
  ```

  > **Salida esperada:** Solo aparecen cuatro recursos bajo los módulos `network` y `compute`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Ejecuta `terraform destroy`, examina el plan propuesto y escribe `yes` únicamente si elimina estos cuatro recursos.

  ```bash
  terraform destroy
  ```

  > **Importante:** No ejecutes `destroy` desde `modules/network` ni `modules/compute` y no borres archivos de estado manualmente.
  {: .lab-note .important .compact}

  > **Salida esperada:** Terraform indica `Destroy complete! Resources: 4 destroyed.`
  {: .lab-note .output .compact}


### Tarea 6.2. Verificar recursos eliminados

Confirmar la limpieza tanto en Terraform como en Google Cloud.


- {% include step_label.html %} Consulta el estado de Terraform después de la destrucción. No debe contener recursos administrados.

  ```bash
  terraform state list
  ```

  > **Salida esperada:** La salida es vacía.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba específicamente VPC, instancia y firewall del laboratorio. Si alguna consulta devuelve recursos, revisa el estado antes de borrarlos manualmente.

  ```bash
  gcloud compute networks list --project="$PROJECT_ID" --filter="name=lab9-vpc" --format="value(name)"
  gcloud compute instances list --project="$PROJECT_ID" --filter="name=lab9-vm" --format="value(name)"
  gcloud compute firewall-rules list --project="$PROJECT_ID" --filter="name=lab9-allow-ssh-iap" --format="value(name)"
  ```

  > **Salida esperada:** Las tres consultas devuelven salida vacía y el laboratorio queda sin recursos activos.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r6 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r6 %}
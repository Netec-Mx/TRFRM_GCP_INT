---
layout: lab
title: "Práctica 5: Crear archivos .tfvars por entorno y outputs condicionales"
permalink: /lab5/lab5/
images_base: /labs/lab5/img
duration: "35 minutos"
objective:
  - "Definir variables complejas, filtrar instancias con for_each y configurar IP pública condicional mediante dynamic, administrando dev y prod con archivos .tfvars y estados separados."
prerequisites:
  - "Prácticas 1 a 4 completadas; carpeta terraform-gcp-intermediate creada en el Escritorio de Windows."
  - "VS Code con Git Bash integrado, Terraform y Google Cloud CLI instalados."
  - "Proyecto GCP con facturación habilitada, Compute Engine API activa y cuotas suficientes para tres VM e2-micro."
  - "Credenciales gcloud y ADC configuradas; permisos para crear redes, reglas de firewall y máquinas virtuales."
introduction:
  - "Crearás una configuración HCL reutilizable para dos entornos simulados: DEV tendrá una VM habilitada y otra deshabilitada; PROD tendrá dos VMs, una con IP externa y otra únicamente privada. Utilizarás variables list(object), map y sensitive, archivos .tfvars distintos, Workspaces para separar estados, y outputs condicionales para verificar los resultados."
slug: lab5
lab_number: 5
final_result: >
  Habrás desplegado simultáneamente entornos DEV y PROD con estados Terraform independientes, verificado que for_each omite VMs deshabilitadas y que dynamic permite IP externa solo cuando corresponde. También comprobarás outputs no sensibles y el tratamiento de variables sensibles, y destruirás todos los recursos conservando el código.
notes:
  - "Ejecuta los comandos en Git Bash integrado en VS Code desde terraform-gcp-intermediate/lab5; los comandos no requieren PowerShell."
  - "Los entornos DEV y PROD son simulados dentro del mismo proyecto GCP: Workspaces separa estados, no permisos IAM."
  - "La regla de firewall permite SSH solo desde el rango IAP, pero esta práctica no requiere conectarse por SSH ni configurar llaves."
  - "La variable sensitive oculta valores en salidas normales, pero no impide que Terraform los conserve en el estado; usa exclusivamente datos ficticios."
  - "El tiempo de 30 minutos supone herramientas instaladas, autenticación vigente y cuotas listas; la creación/destrucción de VMs puede variar."
references:
  - text: "Terraform: variables y tipos complejos"
    url: https://developer.hashicorp.com/terraform/language/values/variables
  - text: "Terraform: for_each"
    url: https://developer.hashicorp.com/terraform/language/meta-arguments/for_each
  - text: "Terraform: expresiones dynamic"
    url: https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks
  - text: "Terraform: outputs"
    url: https://developer.hashicorp.com/terraform/language/values/outputs
  - text: "Terraform: Workspaces"
    url: https://developer.hashicorp.com/terraform/cli/workspaces
  - text: "Google Cloud: IAP TCP forwarding"
    url: https://cloud.google.com/iap/docs/using-tcp-forwarding
prev: /lab4/lab4/
next: /lab6/lab6/
---

---

## 🔎 Tarea 1. Preparar el proyecto y comprobar el entorno — 4 min

Crearás `lab5` dentro del directorio central, prepararás archivos por entorno y confirmarás herramientas, proyecto de GCP y autenticación antes de definir recursos.

### Tarea 1.1. Crear la carpeta lab5 y sus archivos

Organizar la configuración compartida y los dos archivos de parámetros en un proyecto aislado de los demás laboratorios.

- {% include step_label.html %} En **VS Code**, abre **File > Open Folder** y selecciona `terraform-gcp-intermediate` en el Escritorio.

  > **Nota:** No reutilices `lab4`; Workspaces y estados de esta práctica deben permanecer independientes.
  {: .lab-note .info .compact}

  > **Salida esperada:** El explorador muestra las carpetas de prácticas anteriores.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **VS Code**, selecciona **Terminal > New Terminal** y comprueba que el perfil activo sea **Git Bash**.

  > **Salida esperada:** La terminal muestra un prompt terminado en `$`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **Git Bash**, crea la estructura y entra al directorio `lab5`.

  ```bash
  cd "$(cygpath -u "$USERPROFILE")/Desktop/terraform-gcp-intermediate"
  mkdir -p lab5/env
  cd lab5
  touch providers.tf variables.tf main.tf outputs.tf env/dev.tfvars env/prod.tfvars .gitignore
  ```

  > **Salida esperada:** Los seis archivos Terraform y `.gitignore` aparecen en `lab5` y `env/`.
  {: .lab-note .output .compact}

### Tarea 1.2. Verificar herramientas, proyecto y API

Comprobar que Terraform y gcloud responden y que el proyecto de capacitación está listo para desplegar Compute Engine.

- {% include step_label.html %} En **Git Bash**, comprueba que Terraform y Google Cloud CLI estén disponibles.

  ```bash
  terraform version
  gcloud version
  ```

  > **Salida esperada:** Ambos comandos muestran su versión sin errores de comando desconocido.
  {: .lab-note .output .compact}

- {% include step_label.html %} Consulta la cuenta y el proyecto activos antes de crear infraestructura.

  > **Nota:** Si la cuenta o el proyecto no son los asignados, corrígelos antes de continuar; no repitas ADC innecesariamente.
  {: .lab-note .info .compact}

  ```bash
  gcloud auth list --filter="status:ACTIVE" --format="value(account)"
  gcloud config get-value project
  ```

  > **Salida esperada:** Aparecen tu cuenta de curso y el Project ID esperado.
  {: .lab-note .output .compact}

- {% include step_label.html %} Verifica desde **Git Bash** que Compute Engine API esté habilitada.

  ```bash
  gcloud services list --enabled --filter="config.name=compute.googleapis.com" --format="value(config.name)"
  ```

  > **Salida esperada:** Se muestra `compute.googleapis.com`.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}

{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## 🧩 Tarea 2. Definir variables complejas y recursos condicionales — 10 min

Declararás variables `list(object)` y `map`, una variable sensible demostrativa y recursos de red/cómputo. Las VMs se filtrarán por `enabled` y recibirán IP externa solamente cuando se solicite.

### Tarea 2.1. Declarar proveedor y variables tipadas

Configurar Google Provider y parámetros comunes, incluyendo objetos de VM y valores demostrativos sensibles.

- {% include step_label.html %} En **VS Code**, abre `lab5/providers.tf` y agrega la definición de Terraform y Google Provider.

  > **Nota:** Terraform generará `.terraform.lock.hcl` durante `init` para registrar las versiones concretas usadas.
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
    zone    = var.zone
  }
  ```

  > **Salida esperada:** El archivo `providers.tf` contiene `required_providers` y `provider "google"`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `lab5/variables.tf`, declara los parámetros del proyecto y las características de red.

  ```hcl
  variable "project_id" { type = string }
  variable "region" {
    type = string
    default = "us-central1"
  }
  variable "zone" {
    type = string
    default = "us-central1-a"
  }
  variable "network_name" { type = string }
  variable "subnet_cidr" { type = string }
  ```

  > **Salida esperada:** Cinco variables quedan disponibles para la configuración del proveedor y la red.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de `subnet_cidr`, crea en `variables.tf` la variable compleja `vm_config` siguiendo estos requisitos.

  > **Importante:** Inserta el bloque al nivel raíz del archivo; no lo coloques dentro de otra variable.
  {: .lab-note .important .compact}

  **Requisitos del bloque:**
    - Nombre: `vm_config`; tipo: `list(object({...}))`.
    - Atributos: `name` y `machine_type` (`string`); `enabled` y `create_eip` (`bool`).
    - Atributos restantes: `labels` (`map(string)`) y `tags` (`list(string)`).
    - Agrega una descripción breve y guarda el archivo.
    - Referencia: [Variables de Terraform](https://developer.hashicorp.com/terraform/language/values/variables).

  > **Salida esperada:** `vm_config` acepta listas de objetos con seis atributos tipados.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de `vm_config`, agrega una variable de ejemplo marcada como sensible; no almacenes contraseñas reales.

  > **Advertencia:** `sensitive` oculta el valor en salidas habituales, pero no lo cifra ni lo elimina del estado.
  {: .lab-note .warning .compact}

  ```hcl
  variable "app_secrets" {
    description = "Valores ficticios para demostrar sensitive"
    type = object({
      db_user     = string
      db_password = string
    })
    sensitive = true
  }
  ```

  > **Salida esperada:** La variable `app_secrets` está declarada con `sensitive = true`.
  {: .lab-note .output .compact}

### Tarea 2.2. Crear red y regla de firewall para IAP

Desplegar una VPC y subred por entorno, más una regla de firewall limitada al rango de IAP para SSH.

- {% include step_label.html %} En `lab5/main.tf`, declara una VPC personalizada bajo el nombre configurado en `network_name`.

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = var.network_name
    auto_create_subnetworks = false
  }
  ```

  > **Salida esperada:** El recurso `google_compute_network.vpc` define una VPC sin subredes automáticas.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo del cierre de la VPC, agrega la subred asociada con su CIDR y región.

  ```hcl
  resource "google_compute_subnetwork" "subnet" {
    name          = "${var.network_name}-subnet"
    region        = var.region
    ip_cidr_range = var.subnet_cidr
    network       = google_compute_network.vpc.id
  }
  ```

  > **Salida esperada:** La subred referencia la VPC y usa el CIDR del entorno seleccionado.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de la subred, crea una regla de firewall SSH restringida a las direcciones de IAP.

  > **Nota:** La regla no abre SSH a todo Internet; acceder mediante IAP exigiría permisos IAM adicionales.
  {: .lab-note .info .compact}

  ```hcl
  resource "google_compute_firewall" "iap_ssh" {
    name          = "${var.network_name}-iap-ssh"
    network       = google_compute_network.vpc.name
    direction     = "INGRESS"
    source_ranges = ["35.235.240.0/20"]
    target_tags   = ["lab5-ssh"]
    allow {
      protocol = "tcp"
      ports    = ["22"]
    }
  }
  ```

  > **Salida esperada:** El puerto 22 solo admite tráfico desde `35.235.240.0/20` hacia VMs con tag `lab5-ssh`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **VS Code**, confirma que los tres recursos de red cierren sus bloques correctamente.

  > **Nota:** La subred y el firewall deben quedar al nivel raíz de `main.tf`, no anidados dentro de `google_compute_network`.
  {: .lab-note .info .compact}

  > **Salida esperada:** `main.tf` contiene VPC, subred y firewall como recursos independientes.
  {: .lab-note .output .compact}

### Tarea 2.3. Implementar instancias condicionales

Definir instancias Compute Engine que omiten elementos deshabilitados y asignan IP pública solo cuando `create_eip` es verdadero.

- {% include step_label.html %} Debajo del firewall, inicia el recurso de Compute Engine y prepara el mapa de instancias habilitadas.

  > **Importante:** Esta es la estructura inicial: los siguientes pasos agregarán bloques **dentro** del recurso, antes de su última llave `}`.
  {: .lab-note .important .compact}

  ```hcl
  resource "google_compute_instance" "vm" {
    for_each = {
      for vm in var.vm_config : vm.name => vm
      if vm.enabled
    }
    name         = each.value.name
    machine_type = each.value.machine_type
    zone         = var.zone
    labels       = each.value.labels
    tags         = concat(each.value.tags, ["lab5-ssh"])
  }
  ```

  > **Salida esperada:** `for_each` solo incluye los objetos cuyo atributo `enabled` vale `true`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `main.tf`, identifica el filtro de `for_each` y verifica que excluya las VMs deshabilitadas.

  **Comprueba estos elementos:**
    - El mapa usa `vm.name` como clave.
    - La selección contiene `if vm.enabled`.
    - `each.value` lee la configuración de cada VM.
    - Referencia: [for_each en Terraform](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each).

  > **Salida esperada:** La VM marcada como `enabled = false` en DEV no aparecerá en el plan.
  {: .lab-note .output .compact}

- {% include step_label.html %} Dentro de `google_compute_instance.vm`, debajo de `tags`, agrega el disco de arranque.

  ```hcl
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 10
      type  = "pd-standard"
    }
  }
  ```

  > **Salida esperada:** Cada VM utiliza Debian 12 con un disco de arranque estándar de 10 GB.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo del disco y dentro del mismo recurso, agrega la interfaz de red y la IP externa condicional.

  > **Nota:** `dynamic` crea `access_config` solo para las VMs con `create_eip = true`.
  {: .lab-note .info .compact}

  ```hcl
  network_interface {
    subnetwork = google_compute_subnetwork.subnet.id
    dynamic "access_config" {
      for_each = each.value.create_eip ? [1] : []
      content {}
    }
  }
  ```

  > **Salida esperada:** Una VM con `create_eip = false` tendrá interfaz interna, pero ninguna IP externa.
  {: .lab-note .output .compact}

- {% include step_label.html %} Guarda `main.tf` y comprueba que `boot_disk` y `network_interface` estén dentro del recurso VM.

  > **Importante:** La llave final de `google_compute_instance.vm` debe aparecer **después** de los dos bloques; no crees un segundo recurso VM.
  {: .lab-note .important .compact}

  > **Salida esperada:** El recurso VM contiene `for_each`, propiedades, disco y red sin bloques duplicados.
  {: .lab-note .output .compact}

{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## 🗂️ Tarea 3. Configurar .tfvars por entorno y outputs — 6 min

Crearás archivos de parámetros para DEV y PROD, y salidas que muestran exclusivamente las VMs desplegadas y sus IP públicas. El ejemplo sensible quedará oculto en la salida convencional.

### Tarea 3.1. Crear dev.tfvars y prod.tfvars

Definir los valores por entorno y una lista de VMs que demuestre filtrado de recursos y creación condicional de IP pública.

- {% include step_label.html %} En **Git Bash**, captura el Project ID para utilizarlo al completar los archivos `.tfvars`.

  ```bash
  PROJECT_ID="$(gcloud config get-value project)"
  echo "$PROJECT_ID"
  ```

  > **Salida esperada:** Se muestra el identificador del proyecto del curso.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `lab5/env/dev.tfvars`, escribe el archivo DEV y reemplaza `TU_PROJECT_ID` por el valor anterior.

  > **Advertencia:** El valor de demostración no debe coincidir con ninguna contraseña real.
  {: .lab-note .warning .compact}

  ```hcl
  project_id   = "TU_PROJECT_ID"
  region       = "us-central1"
  zone         = "us-central1-a"
  network_name = "lab5-vpc-dev"
  subnet_cidr  = "10.10.0.0/24"
  vm_config = [
    {
      name = "lab5-vm-dev", machine_type = "e2-micro", enabled = true
      labels = { env = "dev", app = "demo" }
      tags = ["demo"], create_eip = true
    },
    {
      name = "lab5-vm-disabled", machine_type = "e2-micro", enabled = false
      labels = { env = "dev", app = "disabled" }
      tags = [], create_eip = false
    }
  ]
  app_secrets = {
    db_user = "demo_dev"
    db_password = "SOLO_DEMO_NO_REAL"
  }
  ```

  > **Salida esperada:** DEV define dos objetos, pero solo `lab5-vm-dev` está habilitada.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `lab5/env/prod.tfvars`, define el entorno PROD con la VM principal y los parámetros de red.

  ```hcl
  project_id   = "TU_PROJECT_ID"
  region       = "us-central1"
  zone         = "us-central1-a"
  network_name = "lab5-vpc-prod"
  subnet_cidr  = "10.20.0.0/24"
  vm_config = [
    {
      name = "lab5-vm-web", machine_type = "e2-micro", enabled = true
      labels = { env = "prod", app = "web" }
      tags = ["web"], create_eip = true
    }
  ]
  app_secrets = {
    db_user = "demo_prod"
    db_password = "SOLO_DEMO_NO_REAL"
  }
  ```

  > **Salida esperada:** PROD define la VM `lab5-vm-web` y sus parámetros de red.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `env/prod.tfvars`, agrega un segundo objeto a `vm_config`, **después** del objeto web y antes de `]`.

  > **Importante:** No sustituyas `lab5-vm-web`; el resultado debe contener **dos** objetos en `vm_config`.
  {: .lab-note .important .compact}

  **Requisitos del segundo objeto:**
    - `name = "lab5-vm-api"`, `machine_type = "e2-micro"` y `enabled = true`.
    - `labels = { env = "prod", app = "api" }` y `tags = ["api"]`.
    - `create_eip = false` para evitar IP pública.
    - Inserta una coma después del objeto web si hace falta.
    - Referencia: [Colecciones y objetos](https://developer.hashicorp.com/terraform/language/expressions/type-constraints).

  > **Salida esperada:** PROD contiene `lab5-vm-web` con IP pública y `lab5-vm-api` sin ella.
  {: .lab-note .output .compact}

### Tarea 3.2. Definir outputs condicionales

Exponer el inventario efectivo de VMs y sus IPs, evitando mostrar por accidente contraseñas de demostración.

- {% include step_label.html %} En `lab5/outputs.tf`, crea un output con los nombres de instancias realmente desplegadas.

  ```hcl
  output "enabled_vms" {
    description = "Nombres de las VMs creadas"
    value       = keys(google_compute_instance.vm)
  }
  ```

  > **Salida esperada:** `enabled_vms` deriva los nombres de las claves reales de `for_each`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de `enabled_vms`, construye el output `public_ips` a partir de los requisitos indicados.

  > **Nota:** Inserta el bloque fuera de `enabled_vms`; usa `[0]` solo después de comprobar que existe `access_config`.
  {: .lab-note .info .compact}

  **Requisitos del bloque:**
    - Nombre: `public_ips`, con una descripción breve.
    - Usa una expresión `for name, vm in google_compute_instance.vm` para generar un mapa.
    - Cuando `length(vm.network_interface[0].access_config) > 0`, devuelve `nat_ip`.
    - En caso contrario devuelve `null`.
    - Referencia: [Outputs de Terraform](https://developer.hashicorp.com/terraform/language/values/outputs).

  > **Salida esperada:** El mapa tendrá IP para la VM web y `null` para la VM API.
  {: .lab-note .output .compact}

- {% include step_label.html %} Debajo de `public_ips`, agrega el output sensible para demostrar la ocultación.

  > **Advertencia:** No ejecutes `terraform output -raw demo_secret`: imprimiría el valor deliberadamente oculto.
  {: .lab-note .warning .compact}

  ```hcl
  output "demo_secret" {
    description = "Secreto ficticio para la demostración"
    value       = var.app_secrets.db_password
    sensitive   = true
  }
  ```

  > **Salida esperada:** El valor `demo_secret` se presentará como sensible en la salida habitual.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `lab5/.gitignore`, protege estados, archivos de variables con datos y otros artefactos locales.

  > **Nota:** Este `.gitignore` protege el repositorio solo si la carpeta se utiliza en Git; no sustituye las políticas de seguridad.
  {: .lab-note .info .compact}

  ```bash
  cat > .gitignore <<'EOF'
  .terraform/
  *.tfstate
  *.tfstate.*
  *.tfplan
  *.tfvars
  *.tfvars.json
  !*.tfvars.example
  EOF
  ```

  > **Salida esperada:** Los estados y los `.tfvars` están excluidos de versiones Git.
  {: .lab-note .output .compact}

{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## 🚀 Tarea 4. Desplegar y validar DEV y PROD — 7 min

Inicializarás Terraform y desplegarás un estado distinto por Workspace, confirmando que los archivos `.tfvars` controlan parámetros sin reemplazar los recursos del otro entorno.

### Tarea 4.1. Inicializar y preparar Workspaces

Descargar el proveedor y crear los entornos lógicos sin reutilizar los workspaces de la Práctica 4.

- {% include step_label.html %} En Git Bash dentro de `lab5`, inicializa la configuración Terraform.

  ```bash
  terraform init
  ```

  > **Salida esperada:** Terraform instala el proveedor y muestra `Terraform has been successfully initialized!`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Normaliza formato y valida la configuración antes de desplegar.

  > **Nota:** Si falla `validate`, revisa primero la definición de `vm_config`, la segunda VM PROD y el output `public_ips`.
  {: .lab-note .info .compact}

  ```bash
  terraform fmt -recursive
  terraform validate
  ```

  > **Salida esperada:** Aparece `Success! The configuration is valid.`
  {: .lab-note .output .compact}

- {% include step_label.html %} Crea Workspaces nuevos para `dev` y `prod`, sin ejecutar `apply` en `default`.

  ```bash
  terraform workspace new dev
  terraform workspace new prod
  terraform workspace list
  ```

  > **Salida esperada:** Aparecen `default`, `dev` y `prod` con `prod` activo.
  {: .lab-note .output .compact}

### Tarea 4.2. Desplegar DEV y verificar la selección

Comprobar que únicamente se crea la VM habilitada y que el estado DEV usa los parámetros definidos en su archivo.

- {% include step_label.html %} Selecciona DEV y revisa su plan antes de aplicar.

  > **Nota:** Revisa que sean **4 recursos**: VPC, subred, firewall y una VM. No debe aparecer `lab5-vm-disabled`.
  {: .lab-note .info .compact}

  ```bash
  terraform workspace select dev
  terraform plan -var-file="env/dev.tfvars"
  ```

  > **Salida esperada:** El plan muestra `4 to add`, si el estado DEV estaba vacío.
  {: .lab-note .output .compact}

- {% include step_label.html %} Despliega DEV mediante Terraform y espera a que termine.

  > **Nota:** Escribe `yes` cuando Terraform solicite confirmar; si el despliegue falla por cuota, no avances a PROD.
  {: .lab-note .info .compact}

  ```bash
  terraform apply -var-file="env/dev.tfvars"
  ```

  > **Salida esperada:** `Apply complete! Resources: 4 added` en un despliegue nuevo.
  {: .lab-note .output .compact}

- {% include step_label.html %} Consulta el inventario y las salidas de DEV.

  ```bash
  terraform output enabled_vms
  terraform output public_ips
  terraform output demo_secret
  ```

  > **Salida esperada:** Aparece `lab5-vm-dev`, su IP externa y `demo_secret = <sensitive>`.
  {: .lab-note .output .compact}

### Tarea 4.3. Desplegar PROD y comparar resultados

Cambiar a otro estado y confirmar dos VMs creadas, una con IP externa y otra solo con IP privada.

- {% include step_label.html %} Selecciona PROD y genera el plan con su archivo de parámetros.

  > **Nota:** El plan de PROD no debe mostrar eliminaciones de DEV; usan estados distintos.
  {: .lab-note .info .compact}

  ```bash
  terraform workspace select prod
  terraform plan -var-file="env/prod.tfvars"
  ```

  > **Salida esperada:** Se proponen **5 recursos**: VPC, subred, firewall y dos VMs.
  {: .lab-note .output .compact}

- {% include step_label.html %} Aplica el plan de PROD de manera interactiva.

  ```bash
  terraform apply -var-file="env/prod.tfvars"
  ```

  > **Salida esperada:** `Apply complete! Resources: 5 added` en el primer despliegue de PROD.
  {: .lab-note .output .compact}

- {% include step_label.html %} Consulta las salidas condicionales de PROD.

  ```bash
  terraform output enabled_vms
  terraform output public_ips
  terraform output demo_secret
  ```

  > **Salida esperada:** Se muestran `lab5-vm-web` y `lab5-vm-api`; solo la primera tiene IP pública.
  {: .lab-note .output .compact}

- {% include step_label.html %} Confirma desde GCP que existen las VMs de ambos entornos simultáneamente.

  ```bash
  gcloud compute instances list --filter="name~^lab5-vm-" --format="table(name,zone,status,networkInterfaces[0].accessConfigs[0].natIP)"
  ```

  > **Salida esperada:** Aparecen tres VM activas: DEV, WEB y API; API no tiene IP pública.
  {: .lab-note .output .compact}

{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

## 🧹 Tarea 5. Comprobar estados y limpiar los entornos — 3 min

Verificarás la independencia de los estados antes de ejecutar `destroy` por entorno y eliminar los Workspaces vacíos, conservando los archivos fuente para prácticas posteriores.

### Tarea 5.1. Revisar el estado de DEV y PROD

Comparar el inventario de Terraform en cada Workspace y detectar recursos antes de iniciar la limpieza.

- {% include step_label.html %} Selecciona DEV y consulta sus recursos administrados.

  ```bash
  terraform workspace select dev
  terraform state list
  ```

  > **Salida esperada:** Aparecen la VPC, la subred, el firewall y `google_compute_instance.vm["lab5-vm-dev"]`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Selecciona PROD y verifica que maneja un inventario diferente.

  > **Nota:** Cada Workspace mantiene su propio estado local en `terraform.tfstate.d/<workspace>/terraform.tfstate`.
  {: .lab-note .info .compact}

  ```bash
  terraform workspace select prod
  terraform state list
  ```

  > **Salida esperada:** Aparecen dos instancias de PROD y sus tres recursos de red.
  {: .lab-note .output .compact}

### Tarea 5.2. Destruir recursos y eliminar Workspaces

Eliminar recursos del proyecto sin afectar otras prácticas, luego borrar Workspaces vacíos y verificar el estado final.

- {% include step_label.html %} Con PROD seleccionado, destruye sus recursos usando los valores de PROD.

  > **Importante:** Revisa `Plan: ... to destroy` y confirma con `yes`; no uses los parámetros de DEV.
  {: .lab-note .important .compact}

  ```bash
  terraform destroy -var-file="env/prod.tfvars"
  ```

  > **Salida esperada:** `Destroy complete! Resources: 5 destroyed` si ambos recursos VM estaban presentes.
  {: .lab-note .output .compact}

- {% include step_label.html %} Selecciona DEV y destruye sus recursos mediante el archivo de valores correcto.

  ```bash
  terraform workspace select dev
  terraform destroy -var-file="env/dev.tfvars"
  ```

  > **Salida esperada:** `Destroy complete! Resources: 4 destroyed` si DEV estaba completo.
  {: .lab-note .output .compact}

- {% include step_label.html %} Regresa a `default`, elimina los dos Workspaces vacíos y verifica que las VM del laboratorio ya no existan.

  > **Advertencia:** Si `workspace delete` señala recursos pendientes, no uses `-force`: revisa y destruye primero el estado correspondiente.
  {: .lab-note .warning .compact}

  ```bash
  terraform workspace select default
  terraform workspace delete dev
  terraform workspace delete prod
  terraform workspace list
  gcloud compute instances list --filter="name~^lab5-vm-"
  ```

  > **Salida esperada:** Solo queda `default` y la consulta a Compute Engine no muestra VMs `lab5-vm-*`.
  {: .lab-note .output .compact}

{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}


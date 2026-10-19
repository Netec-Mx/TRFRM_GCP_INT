---
layout: lab
title: "Práctica 6: Crear múltiples subredes y validar valores de entrada"
permalink: /lab6/lab6/
images_base: /labs/lab6/img
duration: "55 minutos"
objective:
  - "Implementar una VPC con subredes dinámicas mediante for_each, validar entradas mediante validation y crear reglas de firewall condicionales limitadas a rangos de laboratorio."
prerequisites:
  - "Prácticas 1 a 5 completadas y directorio terraform-gcp-intermediate en el Escritorio de Windows."
  - "VS Code con terminal Git Bash, Terraform >= 1.5 y Google Cloud CLI instalados."
  - "Proyecto GCP con Compute Engine API habilitada, facturación y permisos para redes, subredes y reglas de firewall."
  - "gcloud y Application Default Credentials configurados para el proyecto del curso."
introduction:
  - "Construirás la red lab6-vpc con dos subredes y una regla de firewall que se crea solo para elementos clasificados lógicamente como public. Probarás validaciones negativas controladas y verificarás los recursos con Terraform y gcloud antes de eliminarlos. En Google Cloud, public/private no es una propiedad nativa de la subred y las reglas de firewall se aplican a la VPC, no directamente a una subred."
slug: lab6
lab_number: 6
final_result: >
  Habrás creado una VPC y dos subredes mediante for_each, validado parámetros inválidos antes del despliegue,
  generado una regla de firewall condicional con destino limitado y comprobado la eliminación de todos los recursos.
notes:
  - "Ejecuta los comandos en Git Bash integrado en VS Code desde terraform-gcp-intermediate/lab6."
  - "Esta práctica utiliza estado local independiente; no reutilices backend ni Workspaces de laboratorios anteriores."
  - "La clasificación public/private es una etiqueta de configuración Terraform; ninguna subred es pública por sí misma en Google Cloud."
  - "La regla de firewall del laboratorio limita origen y destino; no se crean máquinas virtuales ni se prueba tráfico HTTP real."
  - "Los bloques de construcción dirigida son ejercicios: completa el HCL solicitado antes de ejecutar validate o plan."
references:
  - text: "Terraform: Validación de variables"
    url: https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules
  - text: "Terraform: for_each"
    url: https://developer.hashicorp.com/terraform/language/meta-arguments/for_each
  - text: "Terraform: cidrhost"
    url: https://developer.hashicorp.com/terraform/language/functions/cidrhost
  - text: "Google Cloud: VPC firewall rules"
    url: https://cloud.google.com/firewall/docs/firewalls
  - text: "Terraform Google: google_compute_firewall"
    url: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_firewall
prev: /lab5/lab5/
next: /lab7/lab7/
---

{% assign results = site.data.task-results[page.slug].results %}

---

## 🔎 Tarea 1. Preparar directorio y verificar el entorno GCP — 5 min

Organizarás la carpeta `lab6` dentro del directorio principal del curso y confirmarás que Terraform y Google Cloud CLI apuntan al proyecto correcto. Esta preparación evita modificar accidentalmente la infraestructura de otras prácticas.


### Tarea 1.1. Crear directorio lab6 y archivos Terraform

Preparar un directorio independiente para no modificar los recursos de prácticas anteriores.


- {% include step_label.html %} En **VS Code**, abre el directorio principal del curso con las siguientes opciones. Comprueba la carpeta elegida antes de crear nuevos archivos.

  > **Nota:** No abras `lab5`; el laboratorio tendrá su propio estado local.
  {: .lab-note .info .compact}

  | Campo o acción | Valor esperado |
  |---|---|
  | Menú | **File > Open Folder...** |
  | Ubicación | **Desktop (Escritorio)** |
  | Carpeta | `terraform-gcp-intermediate` |
  | Confirmación | **Select Folder** |

  > **Salida esperada:** El archivo o la configuración indicada queda guardado correctamente para la siguiente validación con Terraform.
  {: .lab-note .output .compact}


- {% include step_label.html %} Abre una terminal integrada y comprueba que sea **Git Bash** antes de pegar comandos de Linux.

  > **Nota:** Todos los comandos bash se ejecutarán dentro de la terminal integrada.
  {: .lab-note .info .compact}

  - En VS Code, selecciona **Terminal > New Terminal**.
  - Si aparece PowerShell, abre la flecha junto al botón **+** y elige **Git Bash**.
  - Comprueba que el prompt corresponda al directorio `terraform-gcp-intermediate` o navega a él en el siguiente paso.

  > **Salida esperada:** El archivo o la configuración indicada queda guardado correctamente para la siguiente validación con Terraform.
  {: .lab-note .output .compact}


- {% include step_label.html %} En **Git Bash**, crea `lab6` y los archivos del ejercicio.


  ```bash
  cd "$(cygpath -u "$USERPROFILE")/Desktop/terraform-gcp-intermediate"
  mkdir -p lab6 && cd lab6
  touch providers.tf variables.tf main.tf outputs.tf terraform.tfvars .gitignore
  pwd
  ```

  > **Importante:** Antes de subir este directorio a Git, agrega a `.gitignore` las entradas `.terraform/`, `*.tfstate`, `*.tfstate.*`, `*.tfplan` y `*.tfvars` si contienen información sensible.
  {: .lab-note .important .compact}

  > **Salida esperada:** La terminal muestra una ruta que termina en `/terraform-gcp-intermediate/lab6` y los seis archivos están disponibles en VS Code.
  {: .lab-note .output .compact}


### Tarea 1.2. Comprobar herramientas, autenticación y proyecto

Validar el contexto para evitar crear recursos en un proyecto incorrecto.


- {% include step_label.html %} Comprueba las versiones instaladas de Terraform y gcloud.


  ```bash
  terraform version
  gcloud version
  ```

  > **Salida esperada:** Se muestran las versiones instaladas de Terraform y Google Cloud CLI; Terraform cumple el requisito `>= 1.5`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Consulta la cuenta y el proyecto configurados; confirma que pertenecen al curso.


  ```bash
  gcloud auth list --filter="status:ACTIVE" --format="value(account)"
  gcloud config get-value project
  ```

  > **Salida esperada:** Se muestra la cuenta activa del curso y el ID del proyecto seleccionado, sin errores de autenticación.
  {: .lab-note .output .compact}


- {% include step_label.html %} Guarda y verifica el Project ID activo; comprueba que la API de Compute esté habilitada.


  ```bash
  PROJECT_ID="$(gcloud config get-value project)"
  echo "Proyecto: $PROJECT_ID"
  gcloud services list --enabled --project="$PROJECT_ID" --filter="config.name=compute.googleapis.com" --format="value(config.name)"
  ```

  > **Salida esperada:** Aparece `compute.googleapis.com`; si no aparece, solicita habilitarla antes de continuar.
  {: .lab-note .output .compact}


{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}


---

## 🔎 Tarea 2. Definir variables y reglas de validación — 12 min

Declararás variables tipadas y reglas `validation` para controlar nombres, regiones, tipos y rangos CIDR. Probarás estas restricciones posteriormente, antes de crear infraestructura y generar costos.


### Tarea 2.1. Definir proyecto, región y nombre de VPC

Crear variables sencillas y una validación de región.


- {% include step_label.html %} En `lab6/providers.tf`, configura Terraform y Google Provider.


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


  > **Salida esperada:** `providers.tf` contiene `required_providers` y el bloque `provider "google"` que utiliza `var.project_id` y `var.region`.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `lab6/variables.tf`, crea la variable obligatoria `project_id` y `network_name`.


  ```hcl
  variable "project_id" {
    description = "Identificador del proyecto del curso"
    type        = string
    validation {
      condition     = length(trimspace(var.project_id)) > 0
      error_message = "project_id no puede estar vacío."
    }
  }
  
  variable "network_name" {
    description = "Nombre único de la VPC de este laboratorio"
    type        = string
    default     = "lab6-vpc"
    validation {
      condition     = can(regex("^[a-z]([-a-z0-9]*[a-z0-9])?$", var.network_name)) && length(var.network_name) <= 63
      error_message = "network_name debe ser un nombre GCP válido de hasta 63 caracteres."
    }
  }
  ```


  > **Salida esperada:** `variables.tf` declara `project_id` y `network_name`, incluida la validación del nombre de la VPC.
  {: .lab-note .output .compact}


- {% include step_label.html %} Debajo de `network_name`, crea la variable `region` mediante una **construcción dirigida**.

  > **Importante:** Declara `region` al nivel raíz del archivo, debajo del cierre de `network_name`.
  {: .lab-note .important .compact}

  **Requisitos:**
  
  - Nombre `region`, tipo `string`, valor predeterminado `us-central1`.
  - Agrega un bloque `validation` que acepte **solo** `us-central1` o `us-east1`.
  - Utiliza `contains([...], var.region)` y un `error_message` claro.
  - Referencia: [Validación de variables](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules).


  > **Salida esperada:** La variable `region` rechaza valores que no pertenezcan a las dos regiones permitidas.
  {: .lab-note .output .compact}


- {% include step_label.html %} Guarda `providers.tf` y `variables.tf` con **Ctrl+S**. Revisa el panel **Problems** de VS Code y verifica que no haya llaves sin cerrar ni variables duplicadas; no confundas las advertencias del editor con una validación definitiva de Terraform.


  > **Nota:** Terraform realizará la comprobación efectiva con `terraform validate` más adelante.
  {: .lab-note .info .compact}

  > **Salida esperada:** El archivo o la configuración indicada queda guardado correctamente para la siguiente validación con Terraform.
  {: .lab-note .output .compact}


### Tarea 2.2. Definir mapa de subredes y restricciones

Usar `map(object)` y controles de formato, tipo y duplicados.


- {% include step_label.html %} Al final de `variables.tf`, agrega una variable `subnets` que acepte un mapa de objetos.

  > **Importante:** Deja abierto este bloque `variable "subnets"` mientras añades las validaciones siguientes.
  {: .lab-note .important .compact}

  ```hcl
  variable "subnets" {
    description = "Subredes: nombre lógico => CIDR y tipo"
    type = map(object({
      cidr = string
      tipo = string
    }))
  ```

  > **Salida esperada:** Se inició el bloque `variable "subnets"` con tipo `map(object({ cidr = string, tipo = string }))`; todavía falta agregar validaciones y el cierre final.
  {: .lab-note .output .compact}


- {% include step_label.html %} Dentro de `subnets`, agrega una validación de nombres y cantidad de elementos.


  ```hcl
  validation {
      condition = length(var.subnets) >= 1 && alltrue([
        for name, subnet in var.subnets :
        can(regex("^[a-z]([-a-z0-9]*[a-z0-9])?$", name)) && length(name) <= 63
      ])
      error_message = "Declara al menos una subred y utiliza nombres GCP válidos."
    }
  ```


  > **Salida esperada:** La primera validación exige al menos una subred y claves compatibles con nombres de subred de GCP.
  {: .lab-note .output .compact}


- {% include step_label.html %} Dentro de `subnets`, construye una validación para `tipo` **sin copiar un bloque completo**.


  **Requisitos:**
  
  - Agrega otro `validation` después del control de nombres.
  - Recorre `var.subnets` mediante una expresión `for` dentro de `alltrue`.
  - Usa `contains` para aceptar únicamente `public` y `private`.
  - Incluye el mensaje `Cada tipo debe ser public o private.`
  - Referencia: [Terraform variable validation](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules).


  > **Salida esperada:** La validación para `tipo` permite solo `public` o `private`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Agrega una validación del formato IPv4/CIDR utilizando `can(cidrhost(...))`.

  > **Nota:** `cidrhost` comprueba que el prefijo sea interpretable; la expresión regular limita la entrada a notación IPv4.
  {: .lab-note .info .compact}

  ```hcl
  validation {
      condition = alltrue([
        for name, subnet in var.subnets :
        can(regex("^[0-9.]+/[0-9]+$", subnet.cidr)) &&
        can(cidrhost(subnet.cidr, 0))
      ])
      error_message = "Cada cidr debe ser un rango IPv4/CIDR válido."
    }
  ```

  > **Salida esperada:** La condición comprueba la notación IPv4/CIDR y utiliza `cidrhost` para rechazar prefijos no válidos.
  {: .lab-note .output .compact}


- {% include step_label.html %} Agrega una validación de CIDR repetidos y cierra `variable "subnets"`.

  > **Advertencia:** Esta validación detecta duplicados exactos; no identifica todos los rangos CIDR parcialmente superpuestos.
  {: .lab-note .warning .compact}

  ```hcl
  validation {
      condition = length(distinct([
        for name, subnet in var.subnets : subnet.cidr
      ])) == length(var.subnets)
      error_message = "No se permiten CIDR duplicados."
    }
  }
  ```

  > **Salida esperada:** El bloque `subnets` queda cerrado y Terraform rechazará dos cadenas CIDR idénticas.
  {: .lab-note .output .compact}


### Tarea 2.3. Configurar parámetros de las dos subredes

Asignar CIDR diferentes y clasificación lógica public/private.


- {% include step_label.html %} En `lab6/terraform.tfvars`, define el proyecto del curso y la región; sustituye `TU_PROJECT_ID`.


  ```hcl
  project_id   = "TU_PROJECT_ID"
  region       = "us-central1"
  network_name = "lab6-vpc"
  ```


  > **Salida esperada:** `terraform.tfvars` contiene el Project ID real, `us-central1` y `lab6-vpc`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Debajo de `network_name`, define `subnets` con frontend y backend.


  ```hcl
  subnets = {
    frontend = {
      cidr = "10.60.1.0/24"
      tipo = "public"
    }
    backend = {
      cidr = "10.60.2.0/24"
      tipo = "private"
    }
  }
  ```


  > **Salida esperada:** `terraform.tfvars` incluye `frontend` y `backend` con sus CIDR y tipo; el mapa está cerrado correctamente.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba en `terraform.tfvars` que ambos registros tengan valores diferentes antes de guardar con **Ctrl+S**.

  > **Nota:** La clasificación `public` y `private` es una etiqueta lógica para filtrar recursos; GCP no asigna esa propiedad a las subredes.
  {: .lab-note .info .compact}

  | Entrada | CIDR | Clasificación lógica |
  |---|---|---|
  | `frontend` | `10.60.1.0/24` | `public` |
  | `backend` | `10.60.2.0/24` | `private` |

  > **Salida esperada:** El archivo o la configuración indicada queda guardado correctamente para la siguiente validación con Terraform.
  {: .lab-note .output .compact}


{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}


---

## 🔎 Tarea 3. Implementar infraestructura dinámica de red — 10 min

Construirás una VPC personalizada y generarás dos subredes a partir de un solo bloque `for_each`. Agregarás una regla Firewall condicional y outputs que permitan inspeccionar los recursos sin duplicar configuraciones.


### Tarea 3.1. Configurar VPC y su relación con el Provider

Usar una red personalizada sin crear automáticamente subredes.


- {% include step_label.html %} En `lab6/main.tf`, crea una VPC personalizada `google_compute_network.vpc`.


  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = var.network_name
    auto_create_subnetworks = false
    description             = "VPC de laboratorio para for_each y validation"
  }
  ```


  > **Salida esperada:** `main.tf` declara `google_compute_network.vpc` con nombre `lab6-vpc` y creación automática de subredes desactivada.
  {: .lab-note .output .compact}


- {% include step_label.html %} Confirma que la VPC utiliza `var.network_name` y tiene `auto_create_subnetworks = false`.


  > **Salida esperada:** La VPC tiene `auto_create_subnetworks = false`, por lo que únicamente existirán las subredes declaradas explícitamente.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `main.tf`, deja una línea en blanco después del cierre de la VPC para insertar el bloque de subredes.

  > **Nota:** La siguiente configuración va al nivel raíz, nunca dentro de la VPC.
  {: .lab-note .info .compact}

  > **Salida esperada:** El siguiente bloque `resource` podrá agregarse fuera de `google_compute_network.vpc`, en el nivel raíz de `main.tf`.
  {: .lab-note .output .compact}


### Tarea 3.2. Crear subredes dinámicas con for_each

Crear dos recursos a partir de un mapa sin duplicar bloques.


- {% include step_label.html %} Debajo de la VPC, inicia un recurso `google_compute_subnetwork.subnets`.

  > **Importante:** El bloque está incompleto deliberadamente: `each.key` todavía requiere `for_each`.
  {: .lab-note .important .compact}

  ```hcl
  resource "google_compute_subnetwork" "subnets" {
    name          = "lab6-${each.key}"
    ip_cidr_range = each.value.cidr
    region        = var.region
    network       = google_compute_network.vpc.id
  }
  ```

  > **Salida esperada:** El recurso de subred tiene `name`, `ip_cidr_range`, `region` y `network`; el siguiente paso agregará `for_each`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Dentro del bloque de subredes, agrega `for_each` como **construcción dirigida**.


  **Requisitos:**
  
  - Debe ubicarse inmediatamente después de la apertura `resource "google_compute_subnetwork" "subnets" {`.
  - Itera **todos** los elementos de `var.subnets`.
  - Mantén `each.key` en `name` y `each.value.cidr` en `ip_cidr_range`.
  - Referencia: [for_each](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each).


  > **Salida esperada:** El bloque de subredes itera el mapa de entrada mediante `for_each`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Verifica que `network` se asocie con `google_compute_network.vpc.id` y no con otra VPC.


  > **Nota:** El recurso crea automáticamente la dependencia entre subredes y VPC.
  {: .lab-note .info .compact}

  > **Salida esperada:** La expresión `google_compute_network.vpc.id` asocia ambas subredes con la VPC del laboratorio.
  {: .lab-note .output .compact}


- {% include step_label.html %} Identifica cómo Terraform nombrará cada subred: `lab6-frontend` y `lab6-backend`.


  > **Salida esperada:** Durante `terraform plan` aparecerán `google_compute_subnetwork.subnets["frontend"]` y `google_compute_subnetwork.subnets["backend"]`.
  {: .lab-note .output .compact}


### Tarea 3.3. Filtrar subredes public, Firewall y outputs

Configurar una regla condicional con origen y destino limitados.


- {% include step_label.html %} Debajo del recurso subredes, declara `locals` y construye `public_subnets` mediante un filtro **dirigido**.


  **Requisitos:**
  
  - Declara `locals { ... }` al nivel raíz de `main.tf`.
  - Dentro crea `public_subnets` como mapa derivado de `var.subnets`.
  - Utiliza una expresión `for key, subnet in var.subnets : key => subnet if ...`.
  - Conserva solo los objetos cuyo `tipo` sea `public`.
  - Referencia: [for expressions](https://developer.hashicorp.com/terraform/language/expressions/for).


  > **Salida esperada:** `local.public_subnets` contiene `frontend` y excluye `backend`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Al final de `main.tf`, crea un Firewall condicional para cada entrada de `local.public_subnets`.


  ```hcl
  resource "google_compute_firewall" "http_public" {
    for_each = local.public_subnets
  
    name      = "lab6-${each.key}-http"
    network   = google_compute_network.vpc.name
    direction = "INGRESS"
  
    allow {
      protocol = "tcp"
      ports    = ["80"]
    }
  
    source_ranges      = ["10.60.0.0/16"]
    destination_ranges = [each.value.cidr]
    description        = "Demostracion restringida: HTTP hacia ${each.key}"
  }
  ```


  > **Salida esperada:** La regla `lab6-frontend-http` se declarará solo para la entrada `frontend` del mapa filtrado.
  {: .lab-note .output .compact}


- {% include step_label.html %} Verifica la diferencia entre origen y destino de la regla Firewall.


  > **Importante:** `source_ranges` limita IPs de origen y `destination_ranges` limita IPs de destino. Las reglas VPC se aplican a la red, no se adjuntan directamente a subredes.
  {: .lab-note .important .compact}
  
  > **Nota:** No se abre HTTP a Internet (`0.0.0.0/0`) ni se crean instancias que acepten tráfico.
  {: .lab-note .info .compact}

  > **Salida esperada:** La regla solo permite TCP/80 con origen `10.60.0.0/16` y destino `10.60.1.0/24`; no se abre HTTP hacia Internet.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `outputs.tf`, agrega nombres de VPC y Firewall condicional para inspección.


  ```hcl
  output "vpc_name" {
    description = "VPC creada"
    value       = google_compute_network.vpc.name
  }
  
  output "firewall_names" {
    description = "Reglas creadas para subredes public"
    value       = [for name, fw in google_compute_firewall.http_public : fw.name]
  }
  ```


  > **Salida esperada:** La regla `lab6-frontend-http` se declarará solo para la entrada `frontend` del mapa filtrado.
  {: .lab-note .output .compact}


- {% include step_label.html %} En `outputs.tf`, agrega el output `subnets` como **construcción dirigida**.


  **Requisitos:**
  
  - Declara `output "subnets"` después de `firewall_names`.
  - Devuelve un mapa con una entrada por subred creada en `google_compute_subnetwork.subnets`.
  - Incluye para cada entrada `cidr` y `region` obtenidos del recurso real.
  - Usa una expresión `for key, subnet in ... : key => {...}`.
  - Referencia: [Outputs](https://developer.hashicorp.com/terraform/language/values/outputs).


  > **Salida esperada:** `terraform output subnets` entregará frontend y backend con CIDR y región.
  {: .lab-note .output .compact}


{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}


---

## 🔎 Tarea 4. Validar configuraciones y desplegar recursos — 9 min

Ejecutarás pruebas negativas de `validation` con valores inválidos y después restaurarás la configuración. Cuando Terraform confirme que el plan es correcto, desplegarás cuatro recursos y verificarás sus atributos con `gcloud`.


### Tarea 4.1. Inicializar y probar validaciones negativas

Verificar la sintaxis y rechazar valores inválidos antes del despliegue.


- {% include step_label.html %} En `lab6`, inicializa Terraform y da formato a todos los archivos HCL.


  ```bash
  terraform init
  terraform fmt -recursive
  ```


  > **Salida esperada:** Se descarga/inicializa el proveedor y `terraform fmt` deja los archivos HCL con formato consistente.
  {: .lab-note .output .compact}


- {% include step_label.html %} Comprueba estructura y sintaxis ejecutando `terraform validate`.


  ```bash
  terraform validate
  ```


  > **Salida esperada:** Terraform muestra `Success! The configuration is valid.`
  {: .lab-note .output .compact}


- {% include step_label.html %} Prueba una región no permitida sin editar archivos; espera un error de `validation`.


  ```bash
  terraform plan -var="region=us-west1"
  ```


  > **Salida esperada:** El plan rechaza `us-west1` mediante la validación de la variable `region`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Prueba un valor `tipo` incorrecto editando temporalmente `terraform.tfvars`, luego restaura el valor.


  En `lab6/terraform.tfvars`, cambia únicamente `frontend.tipo` de `"public"` a `"externa"`. Ejecuta:
  
  ```bash
  terraform plan
  ```
  
  **Después del error**, devuelve `frontend.tipo` a `"public"` y guarda el archivo.
  
  > **Advertencia:** No continúes con una configuración inválida.
  {: .lab-note .important .compact}

  > **Salida esperada:** El plan rechaza `externa` y el archivo queda restaurado a `public`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Prueba un CIDR inválido y vuelve al valor original.


  En `terraform.tfvars`, cambia temporalmente `frontend.cidr` a `"10.60.999.0/24"` y ejecuta:
  
  ```bash
  terraform plan
  ```
  
  Restaura inmediatamente `frontend.cidr = "10.60.1.0/24"` y guarda.
  
  > **Salida esperada:** El plan rechaza el CIDR inválido. Después de restaurar el archivo, las subredes vuelven a tener `10.60.1.0/24` y `10.60.2.0/24`.
  {: .lab-note .output .compact}


### Tarea 4.2. Planificar, aplicar y verificar en Google Cloud

Crear cuatro recursos y comprobar las subredes y la regla Firewall.


- {% include step_label.html %} Con todos los valores restaurados, ejecuta `terraform plan` y revisa los recursos esperados.


  ```bash
  terraform validate
  terraform plan
  ```


  > **Salida esperada:** El plan indica 1 VPC, 2 subredes y 1 regla Firewall: 4 recursos por agregar.
  {: .lab-note .output .compact}


- {% include step_label.html %} Despliega la infraestructura desde **Git Bash** y confirma el resultado.


  ```bash
  terraform apply
  ```


  > **Salida esperada:** Terraform presenta `Apply complete! Resources: 4 added, 0 changed, 0 destroyed.` si no existían recursos previos.
  {: .lab-note .output .compact}


- {% include step_label.html %} Consulta outputs y estado Terraform para comprobar los cuatro recursos.


  ```bash
  terraform output
  terraform state list
  ```


  > **Salida esperada:** La salida contiene VPC, dos subredes y un firewall `lab6-frontend-http`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Valida los recursos con gcloud en el proyecto asignado.


  ```bash
  gcloud compute networks describe lab6-vpc --project="$PROJECT_ID" --format="value(name)"
  gcloud compute networks subnets list --project="$PROJECT_ID" --filter="network:lab6-vpc" --format="table(name,region,ipCidrRange)"
  gcloud compute firewall-rules describe lab6-frontend-http --project="$PROJECT_ID" --format="yaml(name,network,sourceRanges,destinationRanges,allowed)"
  ```


  > **Salida esperada:** Existen `lab6-frontend` y `lab6-backend`, y la regla `lab6-frontend-http` apunta a `10.60.1.0/24`.
  {: .lab-note .output .compact}


{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}


---

## 🔎 Tarea 5. Eliminar recursos y verificar limpieza — 4 min

Eliminarás únicamente los recursos creados en `lab6` y verificarás tanto el estado local como el proyecto GCP. La inspección del plan de destrucción es obligatoria para evitar afectar otras prácticas.


### Tarea 5.1. Destruir infraestructura controladamente

Confirmar que la configuración es válida y evitar eliminar infraestructura de otras prácticas.


- {% include step_label.html %} Confirma desde Git Bash que sigues en el directorio `lab6` y revisa recursos administrados.


  ```bash
  pwd
  terraform state list
  ```


  > **Salida esperada:** Solo aparecen recursos de `lab6`.
  {: .lab-note .output .compact}


- {% include step_label.html %} Ejecuta `terraform destroy` y confirma `yes` después de revisar el plan.


  ```bash
  terraform destroy
  ```


  > **Salida esperada:** Terraform indica la destrucción de los cuatro recursos del laboratorio.
  {: .lab-note .output .compact}


### Tarea 5.2. Verificar que no queden recursos

Comprobar el estado Terraform y consultar el proyecto GCP.


- {% include step_label.html %} Comprueba que el estado Terraform ya no tenga recursos administrados.


  ```bash
  terraform state list
  ```


  > **Salida esperada:** No se listan recursos en el estado actual.
  {: .lab-note .output .compact}


- {% include step_label.html %} Verifica en GCP que la VPC y la regla firewall de lab6 ya no existan.


  ```bash
  gcloud compute networks list --project="$PROJECT_ID" --filter="name=lab6-vpc" --format="value(name)"
  gcloud compute firewall-rules list --project="$PROJECT_ID" --filter="name=lab6-frontend-http" --format="value(name)"
  ```


  > **Salida esperada:** Ambos comandos devuelven resultados vacíos; los recursos de `lab6` han sido eliminados.
  {: .lab-note .output .compact}


{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

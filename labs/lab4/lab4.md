---
layout: lab
title: "Práctica 4: Crear y gestionar entornos dev, test, prod con Workspaces."
permalink: /lab4/lab4/
images_base: /labs/lab4/img
duration: "35 minutos"
objective:
  - "Administrar entornos dev, test y prod mediante Terraform Workspaces, desplegando VPC y subred con nombres y CIDR específicos y comprobando el aislamiento del estado local."
prerequisites:
  - "Prácticas 1, 2 y 3 realizadas; directorio terraform-gcp-intermediate ya creado en el Escritorio de Windows."
  - "VM Windows con Visual Studio Code, Git Bash integrado, Terraform y Google Cloud CLI instalados."
  - "Proyecto de Google Cloud con facturación, Compute Engine API habilitada y permisos para administrar VPC y subredes."
  - "Cuenta gcloud y credenciales Application Default Credentials (ADC) previamente configuradas."
introduction:
  - "Trabajarás con una única configuración Terraform que creará redes independientes para los workspaces dev, test y prod. Identificarás el entorno activo mediante terraform.workspace, asignarás rangos CIDR distintos, verificarás los recursos con gcloud y compararás los archivos de estado antes de destruirlos."
slug: lab4
lab_number: 4
final_result: >
  Habrás creado y utilizado los workspaces dev, test y prod con una única configuración HCL; comprobado tres VPC y tres subredes con nombres y CIDR distintos; identificado estados locales separados; y eliminado los recursos y workspaces, conservando los archivos fuente del laboratorio.
notes:
  - "Todos los comandos se ejecutan desde Git Bash integrado en VS Code, dentro de terraform-gcp-intermediate/lab4."
  - "A diferencia de la Práctica 3, esta práctica usa un backend local independiente; no reutiliza el bucket GCS. Los workspaces no aíslan identidades, permisos ni proyectos."
  - "El workspace default no debe desplegar recursos; se agrega una precondición para evitar su uso accidental."
  - "Los entornos dev, test y prod son simulados y se despliegan dentro del mismo proyecto de capacitación."
  - "La estimación de 30 minutos supone autenticación, API y permisos preparados; los tiempos de aprovisionamiento pueden variar."
references:
  - text: "Terraform CLI Workspaces"
    url: https://developer.hashicorp.com/terraform/cli/workspaces
  - text: "Referencia de terraform.workspace"
    url: https://developer.hashicorp.com/terraform/language/expressions/references
  - text: "Variables de entrada en Terraform"
    url: https://developer.hashicorp.com/terraform/language/values/variables
  - text: "Outputs de Terraform"
    url: https://developer.hashicorp.com/terraform/language/values/outputs
  - text: "Recurso google_compute_network"
    url: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_network
  - text: "Recurso google_compute_subnetwork"
    url: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_subnetwork
prev: /lab3/lab3/
next: /lab5/lab5/
---

---

## 🔎 Tarea 1. Preparar lab4 y verificar el acceso a GCP — 4 min

Crearás el directorio `lab4` dentro del árbol del curso y comprobarás herramientas, proyecto y autenticación. Conservarás intactas las carpetas de las prácticas anteriores y trabajarás únicamente con la terminal Git Bash integrada.

### Tarea 1.1. Crear el directorio y los archivos Terraform

Preparar cinco archivos HCL en una carpeta propia, sin reutilizar archivos `.tfstate` de prácticas anteriores.

- {% include step_label.html %} En **VS Code**, abre la carpeta **terraform-gcp-intermediate** mediante **File > Open Folder**.

  > **Nota:** Utiliza la carpeta que creaste en la Práctica 1; no generes un segundo directorio central.
  {: .lab-note .info .compact}

  > **Salida esperada:** El explorador de VS Code muestra `lab1`, `lab2` y `lab3` dentro de la carpeta del curso.
  {: .lab-note .output .compact}

- {% include step_label.html %} Abre **Terminal > New Terminal** y selecciona **Git Bash** como perfil de la terminal integrada.

  > **Salida esperada:** La terminal muestra un prompt Bash con el símbolo `$`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **Git Bash**, crea y abre `lab4` desde la carpeta del curso y genera los cinco archivos iniciales.

  ```bash
  DESKTOP_WIN="$(powershell.exe -NoProfile -Command '[Environment]::GetFolderPath("Desktop")' | tr -d '\r')"
  DESKTOP="$(cygpath -u "$DESKTOP_WIN")"
  mkdir -p "$DESKTOP/terraform-gcp-intermediate/lab4"
  cd "$DESKTOP/terraform-gcp-intermediate/lab4"
  touch providers.tf main.tf variables.tf outputs.tf terraform.tfvars
  ```

  > **Salida esperada:** Existe `lab4` con `providers.tf`, `main.tf`, `variables.tf`, `outputs.tf` y `terraform.tfvars`.
  {: .lab-note .output .compact}

### Tarea 1.2. Comprobar Terraform y el proyecto activo

Verificar el entorno de ejecución ya instalado y que la cuenta puede acceder al proyecto asignado para los tres entornos.

- {% include step_label.html %} En **Git Bash**, comprueba la ubicación actual y la versión instalada de Terraform.

  ```bash
  pwd
  terraform version
  ```

  > **Salida esperada:** La ruta termina en `/terraform-gcp-intermediate/lab4` y Terraform muestra su versión.
  {: .lab-note .output .compact}

- {% include step_label.html %} Confirma en la misma terminal la cuenta activa de **gcloud** y el proyecto seleccionado.

  ```bash
  gcloud auth list --filter="status:ACTIVE" --format="value(account)"
  gcloud config get-value project
  ```

  > **Importante:** Si falta el proyecto, ejecuta `gcloud config set project TU_PROJECT_ID`. Utiliza tu ID real.
  {: .lab-note .important .compact}

  > **Salida esperada:** Aparecen la cuenta del curso y el identificador correcto del proyecto GCP.
  {: .lab-note .output .compact}

- {% include step_label.html %} Comprueba que las credenciales **ADC** y la API de Compute Engine están disponibles.

  ```bash
  gcloud auth application-default print-access-token >/dev/null && echo "ADC disponible"
  gcloud services list --enabled --filter="NAME:compute.googleapis.com" --format="value(NAME)"
  ```

  > **Nota:** Si ADC no está disponible, ejecuta `gcloud auth application-default login`. Si la API no aparece, solicita o ejecuta `gcloud services enable compute.googleapis.com` con permisos adecuados.
  {: .lab-note .info .compact}

  > **Salida esperada:** Aparece `ADC disponible` y el servicio `compute.googleapis.com`.
  {: .lab-note .output .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## ☁️ Tarea 2. Definir infraestructura dinámica por entorno — 8 min

Configurarás un único conjunto de archivos HCL que define una VPC y una subred por workspace, utilizando nombres derivados del entorno y una tabla de rangos CIDR sin superposición. Completarás algunos bloques a partir de requisitos concretos.

### Tarea 2.1. Configurar el proveedor y las variables

Establecer el proveedor de Google Cloud y los valores de entrada necesarios para el proyecto, región y rangos CIDR.

- {% include step_label.html %} En **VS Code**, abre `lab4/providers.tf` y pega la configuración de Terraform y del proveedor Google.

  ```hcl
  terraform {
    required_version = ">= 1.6.0"

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

  > **Nota:** `.terraform.lock.hcl`, generado por `terraform init`, fijará la versión concreta instalada del proveedor.
  {: .lab-note .info .compact}

  > **Salida esperada:** `providers.tf` define el proveedor Google y referencia `project_id` y `region`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **VS Code**, abre `lab4/variables.tf` y declara la variable `project_id`.

  ```hcl
  variable "project_id" {
    description = "Identificador del proyecto Google Cloud del curso"
    type        = string
  }
  ```

  > **Salida esperada:** La variable `project_id` está declarada sin un proyecto fijo dentro del código.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `lab4/variables.tf`, agrega **debajo de `variable "project_id"`** un bloque para la región.

  > **Importante:** Escribe tú el bloque `variable`; debe llamarse `region`, usar `type = string` y tener `default = "us-central1"`. No lo insertes dentro de las llaves de `project_id`.
  {: .lab-note .important .compact}

  > **Nota:** Consulta [Variables de entrada de Terraform](https://developer.hashicorp.com/terraform/language/values/variables) para verificar la sintaxis de `type` y `default`.
  {: .lab-note .info .compact}

  > **Salida esperada:** `variables.tf` contiene dos bloques `variable` independientes: `project_id` y `region`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En el mismo archivo, agrega **debajo de `variable "region"`** el mapa de CIDR para los tres entornos.

  ```hcl
  variable "subnet_cidrs" {
    description = "Rango CIDR de subred para cada workspace"
    type        = map(string)
    default = {
      dev  = "10.10.1.0/24"
      test = "10.20.1.0/24"
      prod = "10.30.1.0/24"
    }
  }
  ```

  > **Salida esperada:** `subnet_cidrs` contiene tres claves (`dev`, `test`, `prod`) con rangos diferentes.
  {: .lab-note .output .compact}

### Tarea 2.2. Configurar nombres dinámicos y recursos de red

Utilizar `terraform.workspace` para nombrar recursos y bloquear el despliegue involuntario desde el workspace default.

- {% include step_label.html %} En **VS Code**, abre `lab4/main.tf` y agrega un bloque `locals` al comienzo del archivo.

  > **Importante:** Construye una variable local llamada `env` cuyo valor sea `terraform.workspace`. El bloque debe quedar **antes de cualquier `resource`**, fuera de otras llaves.
  {: .lab-note .important .compact}

  > **Nota:** Consulta [Named Values: terraform.workspace](https://developer.hashicorp.com/terraform/language/expressions/references) para obtener la referencia exacta al workspace activo.
  {: .lab-note .info .compact}

  > **Salida esperada:** Los recursos podrán utilizar `local.env` para construir nombres según `dev`, `test` o `prod`.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Debajo del bloque `locals`**, agrega en `main.tf` una VPC y una precondición que impida desplegar desde `default`.

  ```hcl
  resource "google_compute_network" "vpc" {
    name                    = "lab4-${local.env}-vpc"
    auto_create_subnetworks = false
    description             = "VPC de laboratorio para ${local.env}"

    lifecycle {
      precondition {
        condition     = terraform.workspace != "default" && contains(keys(var.subnet_cidrs), terraform.workspace)
        error_message = "Selecciona un workspace válido: dev, test o prod; no utilices default."
      }
    }
  }
  ```

  > **Importante:** La precondición evita un `apply` accidental en `default` y workspaces no definidos en el mapa.
  {: .lab-note .important .compact}

  > **Salida esperada:** La VPC utilizará nombres como `lab4-dev-vpc` y bloqueará workspaces no autorizados.
  {: .lab-note .output .compact}

- {% include step_label.html %} **Debajo del recurso VPC**, agrega una subred que dependa de la VPC y obtenga su CIDR desde el mapa.

  ```hcl
  resource "google_compute_subnetwork" "subnet" {
    name          = "lab4-${local.env}-subnet"
    region        = var.region
    network       = google_compute_network.vpc.id
    ip_cidr_range = var.subnet_cidrs[local.env]
    description   = "Subred de laboratorio para ${local.env}"
  }
  ```

  > **Nota:** La referencia `google_compute_network.vpc.id` permite que Terraform detecte automáticamente la dependencia.
  {: .lab-note .info .compact}

  > **Salida esperada:** El recurso `subnet` usa una VPC asociada y el rango del workspace activo.
  {: .lab-note .output .compact}

- {% include step_label.html %} Guarda `main.tf` y comprueba visualmente que `locals`, VPC y subred sean bloques independientes.

  > **Salida esperada:** Las referencias `local.env`, `var.region` y `var.subnet_cidrs` corresponden a definiciones existentes.
  {: .lab-note .output .compact}

### Tarea 2.3. Configurar parámetros y salidas de infraestructura

Definir el proyecto en un solo archivo de valores y preparar outputs para comparar lo desplegado por workspace.

- {% include step_label.html %} En **VS Code**, abre `lab4/terraform.tfvars` y escribe tu ID real de GCP.

  ```hcl
  project_id = "REEMPLAZA_CON_TU_PROJECT_ID"
  ```

  > **Importante:** Obtén el ID con `gcloud config get-value project` en Git Bash y sustitúyelo entre comillas; no pegues claves privadas.
  {: .lab-note .important .compact}

  > **Salida esperada:** La variable `project_id` contiene tu identificador de proyecto GCP.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **VS Code**, abre `lab4/outputs.tf` y agrega los valores del entorno, nombre de VPC y nombre de subred.

  ```hcl
  output "env" {
    description = "Workspace activo"
    value       = terraform.workspace
  }

  output "vpc_name" {
    description = "Nombre de la VPC"
    value       = google_compute_network.vpc.name
  }

  output "subnet_name" {
    description = "Nombre de la subred"
    value       = google_compute_subnetwork.subnet.name
  }
  ```

  > **Salida esperada:** Hay tres outputs independientes que permiten identificar cada despliegue.
  {: .lab-note .output .compact}

- {% include step_label.html %} En `lab4/outputs.tf`, agrega **debajo de `output "subnet_name"`** un cuarto output.

  > **Importante:** Debe llamarse `subnet_cidr`, incluir `description` y devolver el atributo `ip_cidr_range` del recurso `google_compute_subnetwork.subnet`.
  {: .lab-note .important .compact}

  > **Nota:** Consulta [Outputs de Terraform](https://developer.hashicorp.com/terraform/language/values/outputs) para estructurar el bloque y referenciar un atributo del recurso.
  {: .lab-note .info .compact}

  > **Salida esperada:** `outputs.tf` contiene cuatro outputs y el último devuelve el CIDR asignado a la subred.
  {: .lab-note .output .compact}

{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

## 🚀 Tarea 3. Crear Workspaces y desplegar tres entornos — 10 min

Inicializarás Terraform, crearás los workspaces `dev`, `test` y `prod` y desplegarás dos recursos por entorno. El mismo conjunto de archivos funcionará en los tres casos, cambiando el nombre y CIDR según el workspace activo.

### Tarea 3.1. Inicializar y comprobar los Workspaces

Validar HCL, instalar el proveedor y crear workspaces nombrados antes de cualquier despliegue.

- {% include step_label.html %} En **Git Bash**, ubicado en `lab4`, inicializa Terraform y descarga el proveedor.

  ```bash
  terraform init
  ```

  > **Salida esperada:** Se muestra `Terraform has been successfully initialized!` y aparece `.terraform.lock.hcl`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Ejecuta el formato y la validación antes de crear recursos.

  ```bash
  terraform fmt
  terraform validate
  ```

  > **Salida esperada:** El comando `validate` indica `Success! The configuration is valid.`
  {: .lab-note .output .compact}

- {% include step_label.html %} Crea los workspaces **dev**, **test** y **prod** de forma consecutiva.

  ```bash
  terraform workspace new dev
  terraform workspace new test
  terraform workspace new prod
  ```

  > **Nota:** Cada `workspace new` activa el workspace recién creado; después de estos comandos quedará activo `prod`.
  {: .lab-note .info .compact}

  > **Salida esperada:** Se crean los tres workspaces sin errores de nombres duplicados.
  {: .lab-note .output .compact}

- {% include step_label.html %} Lista los workspaces y verifica cuál está seleccionado.

  ```bash
  terraform workspace list
  terraform workspace show
  ```

  > **Salida esperada:** Aparecen `default`, `dev`, `test` y `prod`; `prod` está marcado con `*`.
  {: .lab-note .output .compact}

### Tarea 3.2. Planificar y aplicar en dev, test y prod

Ejecutar el mismo código para crear VPC y subred por entorno, inspeccionando los cambios propuestos antes de aplicarlos.

- {% include step_label.html %} Selecciona **dev** y confirma que es el workspace activo.

  ```bash
  terraform workspace select dev
  terraform workspace show
  ```

  > **Salida esperada:** El workspace activo se muestra como `dev`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **dev**, revisa el plan y crea los recursos después de comprobar sus nombres.

  ```bash
  terraform plan
  terraform apply
  ```

  > **Importante:** El plan debe indicar **2 to add, 0 to change, 0 to destroy**; comprueba los nombres `lab4-dev-*` antes de escribir `yes`.
  {: .lab-note .important .compact}

  > **Salida esperada:** `Apply complete! Resources: 2 added, 0 changed, 0 destroyed.`
  {: .lab-note .output .compact}

- {% include step_label.html %} Cambia a **test**, revisa el plan y despliega la VPC y la subred de pruebas.

  ```bash
  terraform workspace select test
  terraform plan
  terraform apply
  ```

  > **Importante:** Antes de confirmar `yes`, valida que los nombres comiencen con `lab4-test-` y el CIDR sea `10.20.1.0/24`.
  {: .lab-note .important .compact}

  > **Salida esperada:** Se crean `lab4-test-vpc` y `lab4-test-subnet`, sin cambios en dev.
  {: .lab-note .output .compact}

- {% include step_label.html %} Cambia a **prod**, revisa el plan y despliega el entorno simulado de producción.

  ```bash
  terraform workspace select prod
  terraform plan
  terraform apply
  ```

  > **Advertencia:** `prod` es solo un nombre lógico de laboratorio; comparte el mismo proyecto y los mismos permisos que los demás workspaces.
  {: .lab-note .warning .compact}

  > **Salida esperada:** Se crean `lab4-prod-vpc` y `lab4-prod-subnet`, con CIDR `10.30.1.0/24`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Consulta los outputs del workspace activo **prod** para confirmar sus valores.

  ```bash
  terraform workspace show
  terraform output
  ```

  > **Salida esperada:** Aparecen `env = "prod"`, `vpc_name`, `subnet_name` y `subnet_cidr = "10.30.1.0/24"`.
  {: .lab-note .output .compact}

{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

## 🔍 Tarea 4. Verificar recursos y separación del estado — 5 min

Contrastarás la infraestructura real de Google Cloud con los outputs de Terraform y explorarás el estado de cada workspace. Comprobarás que cambiar de workspace modifica el estado consultado, pero no altera los otros entornos.

### Tarea 4.1. Validar VPC, subred y CIDR en Google Cloud

Comprobar por separado la existencia de las seis piezas de red desplegadas en el proyecto del curso.

- {% include step_label.html %} En **Git Bash**, lista únicamente las VPC de la Práctica 4 en el proyecto activo.

  ```bash
  gcloud compute networks list --filter="name~'^lab4-(dev|test|prod)-vpc$'" --format="table(name,subnet_mode)"
  ```

  > **Salida esperada:** Aparecen `lab4-dev-vpc`, `lab4-test-vpc` y `lab4-prod-vpc`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Lista las subredes y compara los rangos CIDR asignados a cada entorno.

  ```bash
  gcloud compute networks subnets list --filter="name~'^lab4-(dev|test|prod)-subnet$'" --format="table(name,region,ipCidrRange)"
  ```

  > **Salida esperada:** Las subredes `dev`, `test` y `prod` muestran `10.10.1.0/24`, `10.20.1.0/24` y `10.30.1.0/24`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Selecciona **dev** y comprueba con Terraform que no quedan cambios pendientes.

  ```bash
  terraform workspace select dev
  terraform plan
  ```

  > **Salida esperada:** Terraform informa `No changes. Your infrastructure matches the configuration.`
  {: .lab-note .output .compact}

### Tarea 4.2. Examinar estados independientes de Workspaces

Identificar la organización del estado local y verificar que los recursos de cada entorno pertenecen únicamente a su workspace.

- {% include step_label.html %} En **Git Bash**, localiza los archivos locales de estado bajo `terraform.tfstate.d`.

  ```bash
  find terraform.tfstate.d -type f -name 'terraform.tfstate' | sort
  ```

  > **Salida esperada:** Se muestran los archivos `dev/terraform.tfstate`, `test/terraform.tfstate` y `prod/terraform.tfstate`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Con **dev** activo, consulta el workspace y los recursos almacenados en su estado.

  ```bash
  terraform workspace show
  terraform state list
  terraform output
  ```

  > **Salida esperada:** Dev presenta dos recursos Terraform y los outputs correspondientes al prefijo `lab4-dev-`.
  {: .lab-note .output .compact}

- {% include step_label.html %} Compara el estado de **test** con el de **dev**, sin modificar ni crear infraestructura.

  > **Importante:** Selecciona `test`, ejecuta `terraform state list` y `terraform output`, y compara `vpc_name` y `subnet_cidr` con los resultados anteriores de dev.
  {: .lab-note .important .compact}

  > **Nota:** Consulta [CLI Workspaces](https://developer.hashicorp.com/terraform/cli/workspaces) para comprender por qué cada workspace utiliza una instancia distinta del estado.
  {: .lab-note .info .compact}

  > **Salida esperada:** Test conserva los mismos tipos de recursos, pero reporta `lab4-test-vpc` y `10.20.1.0/24`.
  {: .lab-note .output .compact}

- {% include step_label.html %} En **Git Bash**, vuelve a **prod** y comprueba que conserva sus outputs originales.

  ```bash
  terraform workspace select prod
  terraform output vpc_name
  terraform output subnet_cidr
  ```

  > **Salida esperada:** Prod sigue mostrando `lab4-prod-vpc` y `10.30.1.0/24`, sin haberse afectado por las consultas anteriores.
  {: .lab-note .output .compact}

{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

## 🧹 Tarea 5. Destruir recursos y retirar Workspaces — 3 min

Liberarás los recursos de los tres entornos para evitar costos y después eliminarás los workspaces vacíos. Conservarás los archivos `.tf` como material de estudio y no borrarás directorios de prácticas anteriores.

### Tarea 5.1. Destruir las VPC y subredes por entorno

Eliminar primero los recursos administrados en cada estado, verificando que cada ejecución apunte al workspace correcto.

- {% include step_label.html %} En **Git Bash**, selecciona **dev** y destruye sus recursos tras revisar el plan.

  ```bash
  terraform workspace select dev
  terraform destroy
  ```

  > **Advertencia:** Comprueba que el plan solo incluya la VPC y subred con prefijo `lab4-dev-` antes de confirmar con `yes`.
  {: .lab-note .warning .compact}

  > **Salida esperada:** `Destroy complete! Resources: 2 destroyed.`
  {: .lab-note .output .compact}

- {% include step_label.html %} Selecciona **test** y destruye únicamente los recursos de ese workspace.

  ```bash
  terraform workspace select test
  terraform destroy
  ```

  > **Importante:** Comprueba que el plan solo incluye `lab4-test-vpc` y `lab4-test-subnet` antes de confirmar.
  {: .lab-note .important .compact}

  > **Salida esperada:** `Destroy complete! Resources: 2 destroyed.`
  {: .lab-note .output .compact}

- {% include step_label.html %} Selecciona **prod** y destruye los dos recursos del entorno simulado.

  ```bash
  terraform workspace select prod
  terraform destroy
  ```

  > **Importante:** El nombre `prod` no lo convierte en producción real; aun así, valida siempre el plan antes de destruir.
  {: .lab-note .important .compact}

  > **Salida esperada:** `Destroy complete! Resources: 2 destroyed.`
  {: .lab-note .output .compact}

### Tarea 5.2. Eliminar los Workspaces y verificar la limpieza

Volver a default para eliminar los workspaces vacíos sin intentar borrar el workspace activo.

- {% include step_label.html %} Selecciona **default** y elimina los workspaces `dev`, `test` y `prod`.

  ```bash
  terraform workspace select default
  terraform workspace delete dev
  terraform workspace delete test
  terraform workspace delete prod
  ```

  > **Advertencia:** Solo elimines un workspace si su `terraform destroy` terminó correctamente. No uses `-force`.
  {: .lab-note .warning .compact}

  > **Salida esperada:** Terraform elimina los tres workspaces sin advertencias de estado con recursos.
  {: .lab-note .output .compact}

- {% include step_label.html %} Verifica los workspaces restantes y confirma en GCP que no persisten redes `lab4-*`.

  ```bash
  terraform workspace list
  gcloud compute networks list --filter="name~'^lab4-(dev|test|prod)-vpc$'" --format="value(name)"
  gcloud compute networks subnets list --filter="name~'^lab4-(dev|test|prod)-subnet$'" --format="value(name)"
  ```

  > **Salida esperada:** Solo permanece `default` y las consultas de redes y subredes no devuelven recursos de la Práctica 4.
  {: .lab-note .output .compact}

{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

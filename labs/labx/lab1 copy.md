---
layout: lab
title: "Práctica 1: CAMBIAR_AQUI_NOMBRE_DE_LA_PRACTICA" # CAMBIAR POR CADA PRACTICA
permalink: /lab1/lab1/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab1/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - OBJECTIVO_DE_LA_PRACTICA
prerequisites: # CAMBIAR POR CADA PRACTICA
  - PREREQUISITO_1
  - PREREQUISITO_2
  - PREREQUISITO_3
  - PREREQUISITO_4
  - PREREQUISITO_X
introduction: # CAMBIAR POR CADA PRACTICA
  - INTRODUCCIÓN_DE_LA_PRACTICA
slug: lab1 # CAMBIAR POR CADA PRACTICA
lab_number: 1 # CAMBIAR POR CADA PRACTICA
final_result: > # CAMBIAR POR CADA PRACTICA
  RESULTADO_FINAL_ESPERADO_DE_LA_PRACTICA
notes: # CAMBIAR POR CADA PRACTICA
  - NOTAS_CONSIDERACIONES_ADICIONALES
  - NOTAS_CONSIDERACIONES_ADICIONALES
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION Y SU DESCRIPCION
  - text: Documentación oficial de Terraform
    url: https://developer.hashicorp.com/terraform
  - text: Documentación de Azure CLI
    url: https://learn.microsoft.com/es-es/cli/azure/
prev: / # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab3/lab2/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---

---
<!-- EJEMPLO DE PREPARACIÓN DE AMBIENTE SE DEBE ADAPTAR A LA PRACTICA INTERAFAZ GRAFICA O SI SON DIRECTORIOS-->
## 📁 Preparación del ambiente

- {% include step_label.html %} Abre **Visual Studio Code**, selecciona `C:\LABS\couchbase-nosql` y abre una terminal integrada **Git Bash**.

- {% include step_label.html %} Crea una estructura separada para scripts, escenarios, resultados, planes, manifiestos y reportes.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab12/{scripts,manifests,scenarios,results,plans,reports}
  cd /c/LABS/couchbase-nosql/lab12

  pwd
  find . -maxdepth 1 -type d | sort
  ```

  <!-- NOTA PARA DESCRIPCION DE SALIDA ESPERADA MAXIMO 200 CARACTERES -->
  > **salida esperada:** La terminal debe confirmar que el directorio activo es `/c/LABS/couchbase-nosql/lab12` y listar las seis carpetas de trabajo, verificando que la estructura base quedó preparada antes de crear archivos o resultados.
  {: .lab-note .output .compact}

---

<!-- EJEMPLO DE UNA TAREA -->
## 🔎 Tarea 1. NOMBRE DE LA TAREA — 5 min
<!-- DESCRIPCION DE LA TAREA DE 250 CARACTERES -->
En esta sección se descargará Terraform desde la fuente oficial, se instalará y se configurará la variable de entorno `PATH` para poder utilizarlo desde cualquier terminal.
<!-- EJEMPLO DE UNA SUBTAREA -->
### Tarea 1.1. NOMBRE DE LA SUBTAREA
<!-- EJEMPLO DE UN PASO QUE EXPLIQUE LO QUE SE HARA, MAXIMO 150 CARACTERES -->
- {% include step_label.html %} Una vez descargado el archivo, haz clic derecho…
<!-- EJEMPLO DE UNA DEFINICION DE IMAGEN -->
  {% include step_image.html %}

- {% include step_label.html %} TEXTO
{% comment %}
- {% include step_label.html start_at=1 %} Una vez descargado…
{% endcomment %}


  {% include step_image.html %}

- {% include step_label.html %} TEXTO

  {% include step_image.html %}

- {% include step_label.html %} TEXTO
{% comment %}
- {% include step_label.html step=3 %} Extrae el contenido…
{% endcomment %}


  {% include step_image.html %}

- {% include step_label.html %} Abre tu navegador web y visita: [**Descarga Terraform Aquí**](https://developer.hashicorp.com/terraform/install#windows)

  {% include step_image.html %}

- {% include step_label.html %} Selecciona el sistema operativo **Windows** y descarga la versión de 64-bit (**AMD64**) en formato `.zip`.

  <!-- SOLO MODIFICAR EL NOMBRE DE LA IMAGEN /img1.png -->
  {% include step_image.html %}

### Tarea 1.2. Extraer el archivo ZIP

- **Paso 3.** Una vez descargado el archivo, haz clic derecho sobre el archivo `.zip` y selecciona **"Extraer todo..."**.

- **Paso 4.** Extrae el contenido en una carpeta de fácil acceso, por ejemplo:  

  <!-- NOTA PARA RESALTAR ALGO GENERAL -->
  > **NOTA:** Si la carpeta `Terraform` no existe, creala en el directorio `C:\`
  {: .lab-note .info .compact}

  <!-- NOTA PARA RESALTAR ALGO IMPORTANTE -->
  > **IMPORTANTE:** Si la carpeta **TERRALABS** no existe creala en el Escritorio.
  {: .lab-note .important .compact}

  <!-- NOTA PARA RESALTAR ALGO IMPORTANTE -->
  > **IMPORTANTE:** Si la carpeta **TERRALABS** no existe creala en el Escritorio.
  {: .lab-note .warning .compact}

  <!-- NOTA PARA RESALTAR ALGO IMPORTANTE -->
  > **IMPORTANTE:** Si la carpeta **TERRALABS** no existe creala en el Escritorio.
  {: .lab-note .success .compact}

  <!-- NOTA PARA RESALTAR ALGO IMPORTANTE -->
  > **salida esperada:** Si la carpeta **TERRALABS** no existe creala en el Escritorio.
  {: .lab-note .output .compact}


  ```
  C:\Terraform
  ```

  {% include step_image.html %}

### Tarea 1.3. Agregar Terraform al PATH del sistema

- **Paso 5.** Presiona `Win + R`, escribe `sysdm.cpl` y presiona Enter para abrir las **Propiedades del sistema.**

- **Paso 6.** Ve a la pestaña **"Opciones avanzadas"** o **"Avanzadas"** y haz clic en **"Variables de entorno..."**.

  {% include step_image.html %}

- **Paso 7.** En la sección **"Variables del sistema"**, busca la variable llamada `Path` y haz clic en **Editar**.

  {% include step_image.html %}

- **Paso 8.** En la ventana que aparece, haz clic en **"Nuevo"** y agrega la ruta donde descomprimiste Terraform, por ejemplo:

  ```
  C:\Terraform
  ```
  {% include step_image.html %}

- **Paso 9.** Haz clic en **OK** o **Aceptar** en todas las ventanas para guardar los cambios.

<!-- CODIGO PARA OBTENER LOS TEXTOS DE TAREA FINALIZADA COPIAR Y PEGAR, SOLO CAMBIAR EL VALOR DE LA POSICION 0,1,2,3 -->
{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

<!-- ESTRUCTURA DE UNA TAREA COPIA Y PEGAR, MODIFICAR CUANTAS TAREAS SEAN NECESARIAS -->
## Tarea 1. TITULO DE LA TAREA

DESCRIPCIÓN DE LA TAREA

### 1.1. TITULO SUBTAREA

- **Paso 1.** 

- **Paso 2.** 

{% include step_image.html %}

- **Paso 3.** 

{% include step_image.html %}

- **Paso 4.** 

{% include step_image.html %}

- **Paso 5.** 

### 1.2. TITULO SUBTAREA

- **Paso 15.** Con la Terminal abierta ejecuta el siguiente comando:

  ```bash
  terraform -version
  ```
    
  {% include step_image.html %}

  {% include step_image.html %}
  {% include step_image.html %}
  {% include step_image.html %}
  {% include step_image.html inline="true" %}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[#] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
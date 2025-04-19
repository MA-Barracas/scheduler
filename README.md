## Google Cloud Platform (GCP) para automatizar la ejecución de scripts de Python

La combinación más común y eficiente para esto en GCP suele ser **Cloud Functions** (para ejecutar tu código sin gestionar servidores) junto con **Cloud Scheduler** (para disparar la ejecución en el horario que definas).

Aquí tienes un tutorial paso a paso asumiendo que ya tienes tu cuenta de GCP y un proyecto creado:

**Objetivo:** Desplegar tu script de Python en Cloud Functions y programar su ejecución diaria u horaria usando Cloud Scheduler.

**Componentes de GCP a usar:**

1.  **Cloud Functions:** Servicio de cómputo serverless que ejecuta tu código en respuesta a eventos.
2.  **Cloud Scheduler:** Servicio de cron job totalmente gestionado para programar tareas.
3.  **Pub/Sub (opcional pero recomendado):** Servicio de mensajería que puede actuar como intermediario entre Scheduler y Functions, desacoplando los servicios. Scheduler publica un mensaje en un tema de Pub/Sub, y la función se activa al recibir ese mensaje. Esta es la forma más robusta.
4.  **(Opcional) Cloud Storage:** Si tu script necesita leer/escribir archivos (como los PDF que mencionas), Cloud Storage es el lugar ideal para almacenarlos.
5.  **(Opcional) Secret Manager:** Si tu script usa claves API u otras credenciales sensibles, guárdalas de forma segura aquí.

---

**Tutorial Paso a Paso:**

**Paso 1: Preparar tu Script de Python para Cloud Functions**

1.  **Estructura del Código:** Cloud Functions requiere un "punto de entrada" (entry point), que es una función específica en tu script que será llamada cuando se active la función.
    * Si usas un disparador Pub/Sub (recomendado con Scheduler), tu función principal debería aceptar dos argumentos: `event` y `context`. El `event` contiene datos del mensaje Pub/Sub (que puede estar vacío si solo lo usas como disparador).
    * Adapta tu script para que la lógica principal esté dentro de esta función o sea llamada desde ella.

    *Ejemplo (`main.py`):*
    ```python
    # main.py
    import base64
    import os
    # Importa aquí tus librerías necesarias (pdfplumber, pytesseract, etc.)
    # from google.cloud import storage # Ejemplo si usas GCS
    # from google.cloud import vision # Ejemplo si usas Vision AI OCR

    def process_automation(event, context):
        """
        Función de entrada que Cloud Functions ejecutará.
        Activada por un mensaje de Pub/Sub enviado por Cloud Scheduler.
        """
        print("Iniciando ejecución automática del script...")

        # --- Aquí va la lógica principal de tu script ---
        # Ejemplo: leer un PDF de Cloud Storage, hacer OCR, etc.
        # pdf_bucket_name = 'tu-bucket-pdfs'
        # pdf_file_name = 'documento_a_procesar.pdf'
        # output_bucket_name = 'tu-bucket-resultados'

        try:
            # (Opcional) Decodificar datos del mensaje Pub/Sub si envías algo
            # pubsub_message = base64.b64decode(event['data']).decode('utf-8')
            # print(f"Mensaje recibido: {pubsub_message}")

            # --- LÓGICA DE TU SCRIPT ---
            print("Leyendo PDF...")
            # Tu código para leer PDF (quizás descargándolo de GCS)

            print("Extrayendo texto con OCR...")
            # Tu código para OCR

            print("Procesando datos extraídos...")
            # Tu código para manejar los resultados

            print("Guardando resultados...")
            # Tu código para guardar resultados (quizás en GCS o una BD)

            print("¡Automatización completada exitosamente!")

        except Exception as e:
            print(f"Error durante la ejecución: {e}")
            # Considera implementar un manejo de errores más robusto
            # (e.g., enviar una notificación, reintentar, etc.)

        return 'OK' # Cloud Functions espera una respuesta

    # --- Aquí puedes tener el resto de tus funciones auxiliares ---
    # def leer_pdf_de_gcs(bucket, archivo): ...
    # def extraer_ocr(imagen_o_pdf): ...
    # etc.
    ```

2.  **Dependencias (`requirements.txt`):** Crea un archivo `requirements.txt` en el mismo directorio que tu `main.py` y lista todas las librerías externas que tu script necesita (incluyendo las de GCP si las usas, como `google-cloud-storage`, `google-cloud-vision`, etc.).
    *Ejemplo (`requirements.txt`):*
    ```txt
    # requirements.txt
    google-cloud-pubsub
    google-cloud-storage # Si usas GCS
    # pdfplumber # O la librería PDF que uses
    # pytesseract # O la librería OCR que uses
    # Pillow # A menudo necesaria para OCR
    # numpy
    # pandas
    # ...cualquier otra dependencia
    ```

3.  **Acceso a Archivos:** Si tu script necesita leer o escribir archivos temporales, usa el directorio `/tmp` dentro de Cloud Functions, que es el único sistema de archivos escribible (es un disco en memoria). Para almacenamiento persistente, usa Cloud Storage.

**Paso 2: Crear un Tema de Pub/Sub (Canal de Comunicación)**

1.  Ve a la consola de GCP -> Pub/Sub -> Temas.
2.  Haz clic en "Crear tema".
3.  Dale un ID al tema (por ejemplo, `trigger-mi-automatizacion`).
4.  Deja las opciones por defecto y haz clic en "Crear".

**Paso 3: Desplegar la Cloud Function**

Puedes hacerlo desde la consola de GCP o usando la herramienta de línea de comandos `gcloud`.

* **Usando la Consola GCP (más visual):**
    1.  Ve a la consola de GCP -> Cloud Functions.
    2.  Haz clic en "Crear función".
    3.  **Configuración básica:**
        * Nombre de la función: `mi-funcion-automatizada` (o el que prefieras).
        * Región: Elige la más cercana a ti o a tus recursos.
        * Entorno: Elige "1ª gen" o "2ª gen" (2ª gen es más nueva y basada en Cloud Run, puede tener arranques en frío más largos pero más flexibilidad). Para empezar, 1ª gen suele ser más sencilla.
    4.  **Activador:**
        * Tipo de activador: `Pub/Sub`.
        * Tema de Cloud Pub/Sub: Selecciona el tema que creaste en el Paso 2 (`trigger-mi-automatizacion`).
        * Reintentar en caso de error: Habilítalo si quieres que la función se reintente automáticamente si falla.
    5.  **Código fuente:**
        * Código fuente: `Subida de ZIP` (si tienes `main.py` y `requirements.txt` en un zip) o `Editor directo` (para pegar el código). Si usas el editor, asegúrate de crear también el archivo `requirements.txt` en la interfaz. También puedes conectar un repositorio de Cloud Source Repositories, GitHub o Bitbucket.
        * Entorno de ejecución: Selecciona la versión de Python que usas (e.g., Python 3.9, 3.10, etc.).
        * Punto de entrada: Escribe el nombre de la función principal en tu `main.py` (en el ejemplo, `process_automation`).
    6.  **Variables de entorno, red, seguridad (Avanzado):**
        * **Tiempo de espera:** Ajusta el *timeout* (por defecto 60s). Si tu script tarda más, increméntalo (máximo 540s en 1ª gen, más en 2ª gen).
        * **Memoria asignada:** Ajusta la memoria si tu script consume muchos recursos (especialmente con OCR o PDFs grandes).
        * **Cuenta de servicio:** Por defecto, usa la cuenta de servicio de App Engine. Si tu función necesita acceder a otros servicios de GCP (como Cloud Storage), asegúrate de que esta cuenta de servicio tenga los permisos IAM necesarios (por ejemplo, roles como `roles/storage.objectAdmin` para GCS, `roles/cloudfunctions.invoker`). Es **muy importante** configurar los permisos correctamente.
        * **(Recomendado) Secret Manager:** Si usas claves API, añádelas como secretos en Secret Manager y referencia esas variables secretas aquí en lugar de ponerlas en el código.
    7.  Haz clic en "Desplegar" y espera a que la función se cree y despliegue (puede tardar unos minutos).

* **Usando `gcloud` CLI (más rápido para repetir despliegues):**
    1.  Abre tu terminal o Cloud Shell.
    2.  Navega al directorio donde tienes `main.py` y `requirements.txt`.
    3.  Ejecuta el comando de despliegue:
        ```bash
        gcloud functions deploy mi-funcion-automatizada \
          --runtime python39 \
          --trigger-topic trigger-mi-automatizacion \
          --entry-point process_automation \
          --region tu-region \
          --memory 512MB \
          --timeout 300s \
          # --service-account=TU_CUENTA_DE_SERVICIO@tu-proyecto.iam.gserviceaccount.com (Opcional si no usas la default)
          # --set-env-vars MI_VARIABLE=valor (Opcional)
          # --update-secrets /path/to/secret=MI_SECRETO:latest (Opcional, si usas Secret Manager)
        ```
        * Reemplaza `python39`, `trigger-mi-automatizacion`, `process_automation`, `tu-region`, `512MB`, `300s` con tus valores.

**Paso 4: Crear el Trabajo en Cloud Scheduler**

1.  Ve a la consola de GCP -> Cloud Scheduler.
2.  Haz clic en "Crear trabajo".
3.  **Definir el trabajo:**
    * Nombre: `programador-mi-automatizacion` (o el que prefieras).
    * Región: Elige la misma región que tu función o tema de Pub/Sub.
    * Descripción: Opcional.
    * Frecuencia: Define el horario usando formato [cron](https://cloud.google.com/scheduler/docs/configuring/cron-job-schedules).
        * Diariamente a las 3:15 AM: `15 3 * * *`
        * Cada hora en punto: `0 * * * *`
        * Cada día laborable (L-V) a las 9:00 AM: `0 9 * * 1-5`
    * Zona horaria: Selecciona tu zona horaria (importante para que se ejecute cuando esperas).
4.  **Configurar la ejecución:**
    * Objetivo: `Pub/Sub`.
    * Tema: Selecciona el tema que creaste (`trigger-mi-automatizacion`).
    * Mensaje: Puedes dejarlo vacío o enviar un pequeño JSON si tu función lo espera (por ejemplo, `{ "parametro": "valor" }`). Este mensaje llegará codificado en base64 en el campo `event['data']` de tu función Python.
5.  **Configuración opcional:** Puedes definir políticas de reintento si el envío al tema Pub/Sub falla.
6.  Haz clic en "Crear".

**Paso 5: Probar y Monitorizar**

1.  **Prueba Manual:** En Cloud Scheduler, busca tu trabajo recién creado y haz clic en "Ejecutar ahora" para forzar una ejecución inmediata.
2.  **Verificar Logs:** Ve a Cloud Functions, selecciona tu función y ve a la pestaña "Registros" (Logs). Deberías ver los mensajes `print` de tu script, indicando si se ejecutó correctamente o si hubo errores. También puedes ir a Cloud Logging directamente.
3.  **Verificar Estado del Scheduler:** En Cloud Scheduler, puedes ver el historial de ejecuciones del trabajo y si fueron exitosas (enviando el mensaje a Pub/Sub) o fallaron.

---

**Consideraciones Adicionales:**

* **Permisos (IAM):** Este es a menudo el punto más problemático. Asegúrate de que la *cuenta de servicio* que ejecuta tu Cloud Function tenga los permisos necesarios para interactuar con cualquier otro servicio de GCP que uses (Cloud Storage, Vision AI, Secret Manager, etc.).
* **Costos:** Cloud Functions, Cloud Scheduler y Pub/Sub tienen capas gratuitas generosas, pero revisa la [calculadora de precios de GCP](https://cloud.google.com/products/calculator) para estimar costos si esperas un uso intensivo o si tu script requiere mucha memoria/tiempo de ejecución.
* **Manejo de Errores:** Implementa un buen manejo de `try...except` en tu script para capturar errores y registrarlos adecuadamente. Considera configurar alertas en Cloud Monitoring si una función falla repetidamente.
* **Alternativa: Cloud Run + Cloud Scheduler:** Si tu script es muy pesado, necesita más de 9 minutos para ejecutarse, requiere dependencias complejas que son difíciles de instalar en Cloud Functions, o prefieres trabajar con contenedores Docker, puedes empaquetar tu aplicación en un contenedor y desplegarla en Cloud Run. Luego, Cloud Scheduler puede activar el servicio de Cloud Run (a través de una solicitud HTTP o un trabajo de Cloud Run Job).


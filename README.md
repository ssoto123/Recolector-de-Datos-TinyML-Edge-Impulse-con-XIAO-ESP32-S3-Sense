# 📷 Recolector de Datos TinyML (Edge Impulse) con XIAO ESP32-S3 Sense

Este repositorio contiene el código fuente nativo para implementar un servidor web de captura de imágenes por lotes, diseñado específicamente para la creación de Datasets (conjuntos de datos) orientados al entrenamiento de modelos de Machine Learning en [Edge Impulse](https://www.edgeimpulse.com/).

Desarrollado para la placa **Seeed Studio XIAO ESP32-S3 Sense**, este proyecto aborda y resuelve los clásicos problemas de contención de bus (SPI vs Cámara) y desbordamiento de memoria (Brownout y FB-OVF) que suelen ocurrir al utilizar librerías genéricas de terceros.

---

## 🎯 Objetivo de la Práctica
Que el estudiante comprenda la interacción a bajo nivel entre múltiples periféricos (Cámara, Memoria SD y Radio WiFi) compartiendo recursos de hardware limitados. El sistema permite al usuario acceder a una interfaz web mediante la IP local del ESP32, solicitar un lote dinámico de fotografías (ej. 10, 20 o 50) y visualizarlas en una galería generada en tiempo real desde la memoria MicroSD.

Las imágenes se extraen automáticamente en formato **240x240 pixeles**, el estándar cuadrado exigido por Edge Impulse para modelos de clasificación de imágenes eficientes (TinyML).

---

## 🛠️ Requisitos de Hardware y Entorno

1. **Placa:** Seeed Studio XIAO ESP32-S3 Sense (con placa de expansión conectada).
2. **Almacenamiento:** Tarjeta MicroSD (Recomendado 16GB o 32GB) formateada estrictamente en **FAT32**.
3. **Alimentación:** Cable USB-C de transferencia de datos con soporte para corriente alta (evitar cables delgados para prevenir reinicios por *Brownout*).

### Configuración crítica en Arduino IDE
Para que el ESP32 pueda acceder a sus 8MB de memoria externa y procesar las imágenes sin colapsar, debes configurar las herramientas exactamente así antes de compilar:

* **Board:** `Seeed XIAO ESP32S3`
* **PSRAM:** `OPI PSRAM` *(Obligatorio, si se deja en Disabled el código fallará con error de asignación DMA)*.
* **Flash Mode:** `QIO 80MHz`
* **USB CDC On Boot:** `Enabled`

---

## 🚀 Funcionamiento de la Arquitectura Web

El sistema opera sobre la librería nativa `esp_http_server.h` e implementa tres *endpoints* principales (manejadores):

1. **Endpoint Raíz (`/`) - El Panel de Control:**
   Entrega un formulario HTML puro que permite al usuario definir el tamaño del lote (batch size) a capturar.
2. **Endpoint de Captura (`/guardar?lote=N`) - El Motor de Procesamiento:**
   Secuestra el hilo de ejecución para disparar el sensor `N` cantidad de veces con 150ms de descanso entre tomas. Guarda el flujo binario (`fb->buf`) directamente en la FAT32 de la SD y retorna un documento HTML estructurado mediante Flexbox (CSS).
3. **Endpoint de Lectura Dinámica (`/ver_foto?id=X`) - El Proyector:**
   Recibe peticiones asíncronas desde la página de resultados, busca el archivo `/muestreo_X.jpg` en la SD y lo despacha en fragmentos (chunks) usando el encabezado `image/jpeg`.

---

## 💡 Notas de Ingeniería Estudiantil (Resolución de Problemas)

Durante el diseño de esta práctica se resolvieron los siguientes conflictos clásicos de hardware:

* **Conflicto del GPIO 9:** La SD y la Cámara comparten este pin en el hardware de la placa XIAO. Se resolvió usando el mapeo de pines oficial actualizado (`Y9_GPIO_NUM = 48`) aislando los buses de comunicación.
* **Error FB-OVF (Frame Buffer Overflow):** Ocurre cuando la cámara envía datos más rápido de lo que la memoria puede procesar. Se mitigó modificando el pulso del reloj `xclk_freq_hz` de 20MHz a **10MHz** y ajustando la compresión a `jpeg_quality = 12`.
* **Asfixia de Búfer:** Se habilitó `CAMERA_GRAB_LATEST` y `fb_count = 2` para asegurar que el procesador siempre tenga un "escritorio de trabajo" con una imagen fresca disponible.

---

## 🧹 Gestión de Memoria Integrada

El sistema ahora cuenta con un endpoint adicional de limpieza de datos (`/borrar`) diseñado para iterar sobre la estructura de archivos FAT32.
Al presionar el botón de eliminación en la página principal, el ESP32 abrirá el directorio raíz, identificará mediante filtrado de *strings* exclusivamente los archivos que correspondan al formato del dataset (`muestreo_*.jpg`) y los eliminará. Al finalizar, restablecerá el puntero global de numeración (`pictureNumber`) a 1, permitiendo al usuario iniciar una nueva recolección de datos sin necesidad de manipular físicamente la tarjeta de memoria.

---

## 🧠 Integración con Edge Impulse (Flujo de Trabajo TinyML)

Una vez que hayas recolectado tu *Dataset* en la tarjeta MicroSD utilizando nuestra interfaz web, el siguiente paso es entrenar un modelo de clasificación visual y desplegarlo de vuelta en el hardware.

### 1. Carga de Datos (Data Acquisition)
Dado que nuestras imágenes ya están pre-procesadas (240x240 px, JPEG) gracias al código del ESP32, usaremos el método de carga directa:
1. Extrae la MicroSD de la placa XIAO y conéctala a tu computadora.
2. Inicia sesión en [Edge Impulse Studio](https://studio.edgeimpulse.com/) y crea un nuevo proyecto.
3. Dirígete a la pestaña **Data acquisition** y haz clic en el botón **Upload data**.
4. Selecciona los archivos `muestreo_*.jpg` de tu SD.
5. Asigna una etiqueta (*Label*) correspondiente a la clase de ese lote (ej. `manzana`, `defecto`, `vacio`) y asegúrate de que el destino sea **Training** (o deja que el sistema haga un split automático). Haz clic en **Upload data**.

### 2. Diseño del Modelo (Impulse Design)
Con los datos cargados, debemos definir la arquitectura del modelo de Machine Learning:
1. Ve a **Impulse design > Create impulse**.
2. **Image data:** Ajusta la resolución. Aunque recolectamos en 240x240, para inferencia rápida en el ESP32-S3 se recomienda redimensionar a `96 x 96` px (modo *Squash*).
3. **Processing block:** Selecciona **Image**.
4. **Learning block:** Selecciona **Transfer Learning (Images)**.
5. Guarda el *Impulse*.

### 3. Procesamiento y Entrenamiento
1. **Generate Features:** Ve a *Image* en el menú lateral. Selecciona *Color depth: RGB*, guarda los parámetros y haz clic en **Generate features**. Aquí podrás ver el explorador 3D para verificar si tus clases son linealmente separables.
2. **Transfer Learning:** Ve a la pestaña de entrenamiento. Selecciona el modelo pre-entrenado (recomendado: **MobileNetV2 96x96 0.1** para sistemas embebidos de bajos recursos). Define los ciclos de entrenamiento (*Epochs*) y la tasa de aprendizaje (*Learning Rate*), y haz clic en **Start training**.
3. Al finalizar, revisa la **Matriz de Confusión** (*Confusion Matrix*) para evaluar la precisión (*Accuracy*) del modelo.

### 4. Despliegue (Deployment) como Librería de Arduino
Una vez satisfecho con el rendimiento del modelo, vamos a empaquetarlo para nuestra placa XIAO:
1. Dirígete a **Deployment** en el menú de Edge Impulse.
2. Selecciona **Arduino Library** bajo la sección de *Deploy your impulse*.
3. (Opcional) Activa el compilador EON (*EON Compiler*) para optimizar el uso de memoria RAM/Flash.
4. Haz clic en **Build** y descarga el archivo `.zip` generado.

---

## 💻 Implementación de la Inferencia en la XIAO ESP32-S3

Para que el modelo procese imágenes en tiempo real directamente en la placa, debemos integrar la librería generada con el entorno de desarrollo:

1. **Instalar la librería:** En Arduino IDE, ve a *Sketch > Include Library > Add .ZIP Library...* y selecciona el archivo descargado de Edge Impulse.
2. **Cargar el ejemplo:** Ve a *File > Examples > [Nombre de tu Proyecto Edge Impulse] > esp32 > esp32_camera*.
3. **Configuración Crítica de Pines de Cámara:**
   El código de ejemplo viene configurado por defecto para la placa "AI Thinker". Debes modificar las definiciones al inicio del código para habilitar los pines correctos de la XIAO ESP32-S3 Sense. Busca la sección de `#define` y déjala exactamente así:

   ```cpp
   // Selecciona el modelo de cámara correcto comentando los demás
   //#define CAMERA_MODEL_WROVER_KIT
   //#define CAMERA_MODEL_ESP_EYE
   //#define CAMERA_MODEL_M5STACK_PSRAM
   //#define CAMERA_MODEL_M5STACK_WIDE
   //#define CAMERA_MODEL_AI_THINKER
   #define CAMERA_MODEL_XIAO_ESP32S3 // <--- DESCOMENTAR ESTA LÍNEA
   ´´´

## 💻 Paso 4: Compilación y Carga
Manteniendo las mismas configuraciones de hardware en Arduino IDE usadas durante la recolección de datos (OPI PSRAM activado, QIO 80MHz), compila y sube el código a la placa.

## 💻 Paso 5: Prueba en Vivo
Abre el Serial Monitor a 115200 baudios. El ESP32 tomará fotos continuamente, ejecutará la red neuronal y mostrará en consola el porcentaje de coincidencia con cada una de tus etiquetas (inferencia).

Desarrollado para la academia TIC - Prácticas de Internet de las Cosas (IoT).

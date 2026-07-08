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
*Desarrollado para la academia TIC - Prácticas de Internet de las Cosas (IoT).*

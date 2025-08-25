# Informe Técnico — Detector de Deepfakes (Versión Pro Local)

Este informe documenta en detalle el funcionamiento del sistema de detección de deepfakes implementado en un archivo **HTML + CSS + JavaScript**. Su objetivo es permitir a un lector (profesor o revisor técnico) comprender **qué hace el sistema, cómo se calculan las métricas, cómo están organizados los métodos/clases y cómo se integrarían los modelos IA externos (ONNX/MediaPipe)**.

---

## 1. Introducción

El sistema es un **detector de deepfakes local** que corre 100% en el navegador. No necesita backend, y analiza un video cargado por el usuario mediante:

1. Extracción de frames (hasta 5 fps).
2. Procesamiento de cada frame para calcular métricas visuales.
3. Opcional: detección de rostro y clasificación con un modelo IA en formato ONNX.
4. Análisis del audio (si existe) para comprobar sincronía labios–voz.
5. Análisis del nombre del archivo para pistas contextuales.
6. Integración de todas las métricas en un **score final 0–100%**.

El usuario recibe:

* Un **gauge circular** con porcentaje de sospecha.
* **Barras por métrica**, con tooltips explicativos.
* La opción de **exportar un reporte JSON**.

---

## 2. Flujo general del sistema

### 2.1 Entrada

* **Archivo de video** (`.mp4/.mov/.webm`).
* **Nombre de archivo** (string).

### 2.2 Procesamiento

* Conversión del video a frames en un `<canvas>` oculto.
* Preprocesamiento de audio con `AudioContext`.
* Ejecución de métricas (visuales, temporales y de audio).
* Combinación ponderada de métricas en un score final.

### 2.3 Salida

* Probabilidad de manipulación (0–100%).
* Estado interpretado (baja sospecha, incierto, alta sospecha).
* Reporte JSON opcional con desglose técnico.

---

## 3. Arquitectura del código

### 3.1 Componentes principales

* **UI principal (HTML+CSS)**: interfaz de carga, controles, gauge y resultados.
* **Módulo de captura de frames (`captureFrames`)**: obtiene `ImageData` de cada frame.
* **Funciones de métricas**: calculan valores específicos de cada frame o secuencia.
* **Agregador de métricas (`analyzeVideo`)**: integra resultados y genera un reporte.
* **Renderizador (`renderReport`)**: dibuja gauge y barras de métricas.

### 3.2 Clases y métodos clave

* `frameStats(img)`: devuelve luminancia media, nitidez (Laplaciano) y blockiness.
* `dctHighFreq(img)`: calcula energía en altas frecuencias via DCT.
* `skinChromVar(patch)`: evalúa variabilidad cromática en la piel.
* `temporalMotion(frames)`: mide inestabilidad del movimiento entre frames.
* `blinkStats(earSeries, fps)`: estima frecuencia de parpadeo a partir de EAR.
* `lipsyncCorrelation(times, marSeries, audioBuffer)`: correlaciona energía de audio con actividad oral.
* `filenameRisk(name)`: busca palabras clave sospechosas en el nombre del archivo.

---

## 4. Explicación de métricas

### 4.1 Flicker de brillo

* **Método:** `stdDev(diff(meanY))` sobre luminancia media entre frames.
* **Rango esperado:** estable en videos reales, alto en deepfakes.
* **Motivo:** parpadeos de iluminación suelen ser artefactos de generación.

### 4.2 Alta frecuencia DCT

* **Método:** DCT 2D de un recorte central 64×64; se calcula proporción de energía en frecuencias altas.
* **Rango esperado:** natural \~0.2–0.4; deepfakes pueden subir a >0.6.
* **Motivo:** síntesis genera texturas con exceso de detalle artificial.

### 4.3 Inestabilidad de movimiento

* **Método:** diferencias absolutas en frames consecutivos tras aplicar blur.
* **Rango esperado:** suave en reales; errático en falsos.
* **Motivo:** warping/re-iluminación inconsistente.

### 4.4 Bloques de compresión

* **Método:** variaciones en bordes de bloques 8×8.
* **Rango esperado:** uniforme en videos reales; elevado en falsos.
* **Motivo:** regiones manipuladas muestran distinta compresión que el resto.

### 4.5 Nitidez anómala

* **Método:** energía Laplaciana; se penalizan valores muy bajos (desenfoque) o muy altos (oversharpen).
* **Motivo:** blending y escalado alteran nitidez natural.

### 4.6 Labios vs audio

* **Método:** correlación Pearson entre RMS de audio y serie MAR (apertura de boca).
* **Rango esperado:** >0.5 en reales; <0.3 en falsos.
* **Motivo:** deepfakes suelen desincronizar labios y voz.

### 4.7 Consistencia de piel

* **Método:** varianza de crominancia U/V.
* **Motivo:** regiones sintéticas presentan inconsistencias de tono.

### 4.8 Pistas del nombre

* **Método:** regex + keywords ("deepfake", "faceswap", etc.).
* **Motivo:** ayuda a sumar contexto de riesgo.

## 🔹 Cómo está hecho el modelo (parte IA)

El sistema soporta dos formas de “modelo”:

1. **Heurístico puro (sin IA)**

   * Funciona siempre, solo con **JavaScript + Canvas** en el navegador.
   * Se basa en **métricas numéricas** calculadas sobre los frames y el audio (si lo hay).
   * Ventaja: corre offline, rápido y sin dependencias.
   * Limitación: menos preciso ante deepfakes muy sofisticados.

2. **Modelo IA en ONNX (opcional)**

   * Se carga un modelo ligero exportado a **ONNX** (por ej. XceptionNet entrenado en FaceForensics++).
   * Pipeline:

     1. Se detecta y recorta la **cara** (224×224 px).
     2. Se normaliza a un tensor `Float32` en formato `[1, 3, 224, 224]`.
     3. Se pasa al **ONNX Runtime Web** en el navegador.
     4. El modelo devuelve un **logit o probabilidad** → se aplica `sigmoid` si hace falta.
     5. Ese valor (0–1) se integra en el **score global** con más peso que el resto de métricas.

👉 En tu versión final, si no hay modelo cargado, el sistema sigue en modo heurístico. Si hay ONNX válido, ese valor domina el resultado (≈55% del peso).

---

## 🔹 Cómo se mide cada métrica (heurísticas)

### 1. Flicker de brillo

* **Cómo:** se calcula la luminancia media (`meanY`) de cada frame y se mide la desviación estándar entre diferencias sucesivas (`stdDev(diff(meanY))`).
* **Qué indica:** parpadeos o variaciones de luz anormales.

### 2. Alta frecuencia DCT

* **Cómo:** se hace un **DCT 2D** de un recorte central 64×64 en escala de grises.
* Se mide la proporción de energía en frecuencias altas (`high / total`).
* **Qué indica:** texturas artificiales o exceso de detalle.

### 3. Inestabilidad de movimiento

* **Cómo:** se difuminan dos frames consecutivos y se calcula la diferencia promedio en escala de grises.
* **Qué indica:** saltos o warping entre cuadros.

### 4. Bloques de compresión

* **Cómo:** se mide la diferencia de luminancia en bordes de bloques 8×8.
* **Qué indica:** compresión desigual entre cara y fondo.

### 5. Nitidez anómala

* **Cómo:** se aplica un filtro Laplaciano y se mide su energía.
* **Qué indica:** excesivo sharpening o desenfoque raro por mezcla de capas.

### 6. Consistencia de piel

* **Cómo:** se convierte a espacio YUV y se mide la varianza de crominancia U/V en la cara.
* **Qué indica:** variaciones de tono no naturales en piel generada.

### 7. Parpadeo (EAR)

* **Cómo:** con landmarks faciales (MediaPipe) se mide el **Eye Aspect Ratio**.
* Se cuentan parpadeos/minuto y se compara con lo fisiológico (6–30/min).
* **Qué indica:** pocos parpadeos = sospecha.

### 8. Labios vs audio

* **Cómo:** se mide el **Mouth Aspect Ratio (MAR)** y se correlaciona con la energía RMS del audio.
* **Qué indica:** baja correlación cuando hay voz = lipsync falso.

### 9. Pistas del nombre

* **Cómo:** regex + keywords en el nombre del archivo (`deepfake`, `faceswap`, `ai`, etc.).
* **Qué indica:** metadatos sospechosos.

---

📊 **Score final:**

$$
\\text{Score} = \\sum (\\text{métrica} \\times \\text{peso})
$$

* Sin IA: pesos distribuidos (≈10–18% cada métrica).
* Con IA: el modelo ONNX pesa ≈55%, y las demás ajustan el resto.


## 5. Modelos IA (opcionales)

### 5.1 ONNX (Xception/EfficientNet preentrenado)

* Se recorta la cara (224×224), se normaliza a tensor `[1,3,224,224]`.
* Se pasa al `InferenceSession` de ONNX Runtime Web.
* Se aplica `sigmoid` si la salida es logit.
* El valor resultante (0–1) se pondera fuertemente en el score.

### 5.2 MediaPipe Face Landmarker

* Detecta landmarks (ojos, boca, contorno facial).
* Calcula EAR (Eye Aspect Ratio) y MAR (Mouth Aspect Ratio).
* Permite parpadeo y lipsync más precisos.

---

## 6. Cálculo del score final

### 6.1 Fórmula

```js
score01 = Σ (métrica.valor × peso)
```

* Pesos ajustables según disponibilidad de ONNX/landmarks.
* Ejemplo sin IA: cada métrica pesa entre 10–18%.
* Ejemplo con IA: ONNX pesa \~55%.

### 6.2 Interpretación

* **0–35%:** auténtico probable.
* **35–65%:** incierto (revisión manual).
* **65–100%:** potencial deepfake.

---

## 7. Limitaciones

* Dependencia de calidad: videos muy comprimidos pueden falsear métricas.
* Métricas heurísticas son aproximadas.
* Modelo ONNX recomendado para reforzar análisis.
* Audio opcional: no siempre disponible.

---

## 8. Posibles mejoras

* Integrar Grad-CAM para visualizar zonas sospechosas.
* Guardar thumbnails de frames sospechosos.
* Ajustar pesos dinámicamente según dataset.
* Integrar un backend (FastAPI + PyTorch) para modelos más pesados.

---

## 9. Conclusiones

El sistema combina **métricas heurísticas** con la posibilidad de usar **modelos preentrenados** para ofrecer un detector flexible y educativo. Es un prototipo útil para comprender los riesgos de los deepfakes y los métodos de detección.

# Práctica 3 · Visión por Computador

Conteo de dinero con monedas y clasificación de microplásticos, usando únicamente visión clásica (sin aprendizaje profundo).

## Autoría

| | |
|---|---|
| **Autores** | Carmelo Aidán Romero Pérez y David Sosa Domínguez |
| **Asignatura** | Visión por Computador |
| **Fecha** | 22/10/2026 |

## Contenido

- `VC_P3_1.ipynb`: notebook con el código de las dos tareas.

**Librerías:** `opencv-python`, `numpy`, `matplotlib`, `pandas`, `scikit-image`, `scikit-learn`

---

## Tarea 1: Dinero en monedas (demostrador en vivo)

El programa lee la cámara, detecta las monedas y muestra el total en euros sobre el vídeo.

### Método

1. **Segmentación:** gris → desenfoque gaussiano → umbral de Otsu invertido → cierre (9×9) y apertura (41×41) morfológicos.
2. **Contornos:** solo externos, se aplica el casco convexo y se filtran por **área** (1500–25000 px) y **circularidad** (> 0,75).
3. **Tamaño:** el radio de cada moneda se obtiene con `minEnclosingCircle`.
4. **Calibración:** al pulsar `c`, se hace clic en una moneda y se indica su valor con una tecla. Con su diámetro real se calcula la escala píxeles/mm:

   `escala = radio_px / (diámetro_mm / 2)`

5. **Clasificación:** el diámetro de cada moneda en mm se compara con la tabla y se le asigna la denominación más cercana (tolerancia de 1 mm). Se suman los valores.

### Manejo

| Tecla | Acción |
|:-:|---|
| `c` | Calibrar: clic en una moneda y tecla `1`–`8` (1c, 2c, 5c, 10c, 20c, 50c, 1€, 2€) |
| `Esc` | Salir |

### Diámetros usados (mm)

| 1c | 2c | 5c | 10c | 20c | 50c | 1€ | 2€ |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| 16,25 | 18,75 | 21,25 | 19,75 | 22,25 | 24,35 | 23,25 | 25,75 |

> **Nota:** el valor de 50 cent (24,35 mm) es el del enunciado; el oficial es 24,25 mm.

### ¿Funciona correctamente?

Funciona bien con fondo claro y uniforme (caso perfecto); falla con sombras, reflejos o si cambia la distancia a la cámara.

**Limitaciones conocidas del método:**
- La escala solo es válida si la cámara no se mueve tras calibrar.
- Monedas de diámetro parecido (20c, 1€, 50c) pueden confundirse.
- Monedas solapadas se detectan como un único objeto y se descartan por circularidad.
- Los objetos redondos que no son monedas se contarían como tales.

---

## Tarea 2: Clasificación de microplásticos

Se clasifican las partículas de `MPs_test.jpg` en tres clases: **FRA** (fragmentos), **PEL** (pellets) y **TAR** (alquitrán).

### Método

1. **Segmentación:** se pasa la imagen a espacio **LAB**. El fondo se estima con un filtro de mediana grande y se marca como partícula todo píxel cuya distancia de color al fondo supera 10. Después se aplica una apertura 3×3 y se eliminan los bordes.
2. **Características** por partícula (se descartan las de área < 40 px):
   - **Circularidad:** `4π·área / perímetro²`
   - **Brillo relativo:** brillo medio de la partícula dividido entre el brillo mediano de la imagen.
3. **Reglas de clasificación:**

   | Condición | Clase |
   |---|:-:|
   | Brillo < umbral de Otsu (calculado sobre las partículas) | **TAR** |
   | Circularidad > 0,76 | **PEL** |
   | Resto | **FRA** |

4. **Evaluación:** cada anotación (*bounding box*) se empareja con la partícula cuyo centroide cae dentro, y se calculan la *accuracy* y la matriz de confusión con `scikit-learn`.

### Resultados

**Accuracy:** 0.66 (74 de 112 partículas bien clasificadas)

| Real \ Predicha | FRA | PEL | TAR |
|:-:|:-:|:-:|:-:|
| **FRA** | 38 | 12 | 12 |
| **PEL** | 8 | 22 | 6 |
| **TAR** | 0 | 0 | 14 |

Métricas por clase:

| Clase | Muestras | Precision | Recall |
|:-:|:-:|:-:|:-:|
| **FRA** | 62 | 0.83 | 0.61 |
| **PEL** | 36 | 0.65 | 0.61 |
| **TAR** | 14 | 0.44 | 1.00 |

### Análisis

El clasificador acierta en el 66 % de las partículas. El comportamiento es muy distinto según la clase:

- **TAR (alquitrán):** es la clase mejor detectada, ya que las 14 partículas reales se clasifican correctamente (recall de 1.00). Sin embargo, su precisión es baja (0.44), porque 18 partículas de otras clases se clasifican como alquitrán (12 fragmentos y 6 pellets). La regla de brillo se evalúa en primer lugar, de modo que cualquier plástico oscuro o con sombra se considera alquitrán antes de comprobar su forma.
- **FRA (fragmentos):** es la clase más numerosa y tiene una precisión alta (0.83), pero un recall de 0.61. Los errores se reparten a partes iguales: 12 se clasifican como TAR (por oscuros) y 12 como PEL (por tener un contorno muy redondeado, con circularidad superior a 0,76).
- **PEL (pellets):** 22 de 36 se clasifican bien. Los fallos son 8 pellets considerados fragmentos, probablemente porque la segmentación deforma su contorno y reduce la circularidad, y 6 considerados alquitrán por su brillo bajo.

Todos los errores provienen de solo dos características (brillo y circularidad). Fragmentos y pellets comparten rangos de ambas, por lo que no es posible separarlos con reglas tan simples. Además, las anotaciones pueden contener errores, lo que también afecta a las métricas.

**Posibles mejoras:**
- Añadir más descriptores de SMACC (relación entre ejes de la elipse, área/contenedor, relación entre distancias del centroide al contorno).
- Usar el color (canales a y b de LAB) además del brillo para distinguir mejor el alquitrán.
- Mejorar la segmentación para que los contornos de las partículas sean más fieles.

---

## Conclusiones

- Con segmentación y descriptores sencillos (tamaño, circularidad, brillo) se resuelven ambos problemas sin aprendizaje profundo.
- La calidad de la segmentación y de la iluminación condiciona el resultado.

## Referencias

- [Documentación de OpenCV: características de contornos](https://docs.opencv.org/4.x/dd/d49/tutorial_py_contour_features.html)
- [SMACC: A System for Microplastics Automatic Counting and Classification](https://doi.org/10.1109/ACCESS.2020.2970498)

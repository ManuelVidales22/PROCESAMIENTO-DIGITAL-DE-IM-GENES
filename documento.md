# Proyecto DICOM — Guía de presentación

Explicación completa de lo que hace [ProyectoDicom.ipynb](ProyectoDicom.ipynb): **qué** se hizo, **cómo** se hizo y **por qué**. Al final hay una sección de preguntas probables y otra de puntos a revisar antes de presentar.

---

## Índice

0. [Idea general del proyecto](#0-idea-general-del-proyecto)
1. [Carga de imágenes y funciones auxiliares](#1-carga-de-imágenes-y-funciones-auxiliares)
2. [Imagen plana (foto)](#2-imagen-plana-foto)
3. [Resonancia magnética (NIfTI)](#3-resonancia-magnética-nifti)
4. [Tomografía (DICOM)](#4-tomografía-dicom)
5. [Anonimización](#5-anonimización-solo-tc)
6. [Teoría de color](#6-teoría-de-color)
7. [Histograma y CDF](#7-histograma-y-cdf)
8. [Ecualización del histograma](#8-ecualización-del-histograma)
9. [Modelamiento del ruido](#9-modelamiento-del-ruido-vía-histograma)
10. [Kernel y convolución](#10-kernel-y-convolución) — incluye [10.2 Comparación 3×3 vs 5×5](#102-comparación-kernel-33-vs-55)
11. [Filtros de paso alto (bordes)](#11-filtros-de-paso-alto-detección-de-bordes)
12. [Operadores basados en la varianza](#12-operadores-basados-en-la-varianza)
13. [Pipeline final y conclusiones](#13-pipeline-final-y-conclusiones)
14. [Preguntas probables y cómo responderlas](#14-preguntas-probables-y-cómo-responderlas)
15. [Puntos a revisar antes de presentar](#15-puntos-a-revisar-antes-de-presentar)
16. [Guion de la presentación (8 min, 4 personas, en el notebook)](#16-guion-de-la-presentación-8-min-4-personas-en-el-notebook)

---

## 0. Idea general del proyecto

Se trabajan **tres imágenes de naturaleza distinta** para comparar cómo se comportan frente a las mismas técnicas de procesamiento:

| Fila | Imagen | Formato | Qué representan los valores |
|---|---|---|---|
| 0 | **Foto** plana (`gatos.JPG`) | JPG, RGB, 8 bits por canal | Luz captada por el sensor de la cámara (0–255) |
| 1 | **RM** de cerebro (T1 MPRAGE) | NIfTI (volumen 3D), corte axial z = 90 | Intensidad de señal de radiofrecuencia — **sin unidades físicas** |
| 2 | **TC** de cráneo sin contraste | Serie DICOM, corte 9 | Atenuación de rayos X en **Unidades Hounsfield (HU)** |

**Organización por temas, no por modalidad.** Las tres imágenes se cargan una sola vez al inicio y cada sección (histograma, ecualización, ruido, filtros…) las muestra **juntas en la misma figura, una fila por imagen**.

- **Por qué:** así la comparación es directa. Se ve en una sola figura que la misma técnica funciona distinto según el tipo de imagen.
- La única sección exclusiva de la foto es **Teoría de color**, porque es la única con color. Después de esa sección la foto se convierte a **gris** para compararla con la RM y la TC, que tienen un solo canal.

**Librerías** (`requirements.txt`):

| Librería | Para qué |
|---|---|
| `numpy` | Las imágenes son matrices; todas las operaciones numéricas |
| `matplotlib` | Mostrar imágenes, histogramas y gráficas |
| `Pillow (PIL)` | Abrir el JPG |
| `pydicom` | Leer archivos DICOM (encabezado + píxeles) y guardar la versión anonimizada |
| `nibabel` | Leer el volumen NIfTI de la resonancia |
| `scipy.stats` | Pruebas estadísticas (Kolmogorov-Smirnov, Shapiro-Wilk, chi², distribuciones) |

**Principio del proyecto:** lo central de la materia se implementó **a mano** (histograma, CDF, ecualización, conversión de color, convolución con ciclos `for`, Sobel, Laplaciano, Lee). Las librerías se usan para leer archivos, graficar y para las pruebas estadísticas.

---

## 1. Carga de imágenes y funciones auxiliares

### Qué se hace
Una sola celda define imports, **rutas relativas** del proyecto (no Google Drive), el corte escogido de cada volumen y las **zonas homogéneas de 50×50** que se usan después en ecualización, ruido y filtros.

```python
CORTE_NIFTI = 90   # corte axial de la RM
CORTE_TC    = 9    # corte axial de la TC (tras ordenar por z)
TAM_PARCHE  = 50   # zona homogénea de 50x50 para medir el ruido
PARCHES = {"Foto": (200, 600), "RM": (200, 200), "TC": (330, 170)}  # (fila, columna) esquina sup. izq.
```

**Por qué todo en una celda:** si se quiere cambiar el corte o el parche, se cambia en un solo lugar y todo el notebook se recalcula de forma coherente.

### Funciones auxiliares (se reutilizan en todo el notebook)

| Función | Qué hace | Por qué |
|---|---|---|
| `a_hu(ds)` | `HU = pixel_array × RescaleSlope + RescaleIntercept` | El DICOM no guarda HU directamente; hay que reescalar |
| `primer_valor(v)` | Toma el primer valor si `WindowCenter`/`WindowWidth` vienen como lista | Algunos DICOM traen varias ventanas |
| `mostrar(ax, img, nombre)` | Muestra cada imagen con **su propia ventana** (`vmin`, `vmax`) | Cada modalidad tiene un rango distinto; sin ventana la RM y la TC se ven mal |
| `calcular_histograma(imagen)` | Cuenta píxel por píxel con listas; **se desplaza por el mínimo** | Así acepta valores **negativos** (HU del aire ≈ −1000) |
| `calcular_cdf(conteo)` | Suma acumulada normalizada (0 → 1) | Base de la ecualización y de la lectura por percentiles |
| `valor_en_percentil` | Primer valor donde la CDF ≥ p | Para leer "el 50 % de los píxeles ≤ X" |
| `dibujar_histograma` | Barras; si hay muchos valores, **solo para dibujar** agrupa de a varios | Con 16 bits hay miles de valores; el conteo real no se altera |
| `parche(img, nombre)` | Extrae la zona 50×50 definida en `PARCHES` | Misma zona en todas las secciones → resultados comparables |

### Carga de cada imagen

- **Foto:** `Image.open` + `np.array`. Si viene con 4 canales (RGBA) se quita el canal alfa. Resultado: `(2018, 1170, 3)`, `uint8`.
- **RM:** `nib.load(...).get_fdata()` → volumen `(256, 256, 240)`. Se toma el corte `[:, :, 90]` y se rota 90° (`np.rot90`) para verlo en orientación anatómica habitual. Valores **originales, sin normalizar** (0 a 663).
- **TC:** se leen todos los `.dcm` de la serie (excluyendo los `_anonimo.dcm` generados) y **se ordenan por `ImagePositionPatient[2]` (posición z real en mm)**, no por nombre de archivo.
  - **Por qué:** el nombre del archivo (UID) no garantiza el orden anatómico; la posición z sí.
  - Resultado: 10 tajadas, corte 9 → `(512, 512)`, de −1024 a 1949 HU.

### Ventanas de visualización
```python
VENTANA = {"Foto": (0, 255),
           "RM":   (0, percentil 99.5),          # → (0, 530)
           "TC":   (WC − WW/2, WC + WW/2)}        # → (−15, 75) HU, ventana de cerebro del propio DICOM
```
- **RM — percentil 99,5:** unos pocos píxeles muy brillantes comprimirían todo lo demás a negro; se deja saturar el 0,5 % más brillante.
- **TC — ventana del DICOM:** el propio archivo trae `WindowCenter = 30` y `WindowWidth = 90` (ventana de cerebro). El cerebro ocupa ~20–45 HU, una franja minúscula del rango total; sin ventana se vería gris uniforme.

---

## 2. Imagen plana (foto)

### Qué se hace
Se inspecciona la foto como matriz:

| Propiedad | Valor |
|---|---|
| Modo PIL | RGB |
| Tamaño (ancho, alto) | (1170, 2018) |
| Shape numpy | (2018, 1170, 3) → **(alto, ancho, canales)** |
| dtype | `uint8` |
| Rango | 0 a 255 |
| Píxel [0,0] | `[60, 32, 8]` → 3 valores (R, G, B), sin canal alfa |

### Conceptos para explicar
- **8 bits por canal** → 2⁸ = **256 niveles** por canal.
- Colores posibles: 256³ = **16 777 216** (~16,7 millones).
- PIL da el tamaño como (ancho, alto) pero numpy como (alto, ancho): filas primero.
- Si el píxel tuviera 4 valores, el 4.º sería el **canal alfa** (transparencia).

---

## 3. Resonancia magnética (NIfTI)

### Qué es el formato NIfTI
- Guarda **todo el volumen 3D en un solo archivo** (normalmente convertido desde DICOM). Se abre con `nibabel`.
- **No trae encabezado DICOM** (no hay datos de paciente, fabricante, etc.), por eso no se anonimiza.
- La RM **no tiene color ni unidades físicas**: son intensidades relativas de la señal de radiofrecuencia.

### Encabezado — lo que importa
| Campo | Valor | Significado |
|---|---|---|
| `dim` | `[256, 256, 240]` | Número de vóxeles en x, y, z |
| `pixdim` | `[0.9375, 0.9375, 0.94]` mm | Tamaño de cada vóxel en mm |
| `datatype` | `int16` (16 bits) | Tipo de dato guardado |
| `qform/sform` | `scanner` | Matriz que ubica el volumen en el espacio del escáner |

- **Volumen de un vóxel** = 0,9375 × 0,9375 × 0,94 ≈ **0,826 mm³** (casi isotrópico: por eso "iso" en el nombre del archivo).
- **Toda medida física se hace con `pixdim`:** *n.º de vóxeles × tamaño del vóxel*. Ej.: 10 vóxeles en x = 9,375 mm. El tamaño del vóxel **cambia entre pacientes/equipos**, así que nunca se mide en píxeles.

### Función `process_dicom()`
Se define aquí (aunque se usa con la TC): imprime los atributos clave del encabezado DICOM (modalidad, fabricante, bits, espaciado, ventana, rescale, parámetros de RM como TR/TE/FlipAngle si existen) y estadísticas de la matriz (shape, mín, máx, media, σ, percentiles).

### Elección del corte (z = 90)
- Se muestran cortes axiales de z = 60 a z = 120 cada 10 para justificar la elección.
- Se muestran los planos **coronal** y **sagital** con una línea roja que marca la altura del corte axial.
- **Por qué z = 90:** se ven **cerebelo, protuberancia, lóbulos temporales y órbitas**, aproximadamente la misma altura anatómica del corte de TC escogido → la comparación entre modalidades es más justa.

---

## 4. Tomografía (DICOM)

### Cómo se lee
- Cada `.dcm` es **una tajada**. `pydicom.dcmread()` devuelve el encabezado como un **diccionario** (atributo → valor); la imagen se obtiene con `.pixel_array`.
- Tajadas ordenadas por z: de −8,88 mm a 13,56 mm, separadas ~2,49 mm. El corte 9 es el más alto (z = 13,56 mm, `InstanceNumber 10`).

### Atributos clave del encabezado

| Atributo | Valor | Significado |
|---|---|---|
| `Modality` | CT | Tomografía computarizada |
| `Manufacturer / Model` | GE MEDICAL SYSTEMS, LightSpeed Pro 32 | Equipo |
| `KVP` | 120 | Voltaje del tubo de rayos X |
| `ConvolutionKernel` | **SOFT** | Reconstrucción para **tejido blando** |
| `Rows × Columns` | 512 × 512 | Tamaño de la matriz |
| `PixelSpacing` | 0,3965 mm | mm reales por píxel |
| `SliceThickness` | 2,44 mm | Grosor de la tajada |
| `BitsStored` | 16 | Hasta 65 536 valores |
| `PixelRepresentation` | 1 | Enteros **con signo** |
| `PhotometricInterpretation` | MONOCHROME2 | Escala de grises, 0 = negro |
| `WindowCenter / Width` | 30 / 90 | Ventana de cerebro sugerida |
| `RescaleSlope / Intercept` | 1 / −1024 | Conversión a HU |

**Por qué importa el kernel SOFT:** existe otra serie ("Ossoux", kernel BONEPLUS) reconstruida para hueso: más nitidez en hueso pero **más ruido** en tejido blando. Para estudiar el ruido del cerebro conviene la serie SOFT.

### Conversión a Unidades Hounsfield
```
HU = valor_guardado × RescaleSlope + RescaleIntercept  =  valor − 1024
```
- Crudo: 0 a 2973 → HU: **−1024 a 1949**.
- El mínimo −1024 HU es **aire recortado** (4,3 % de la imagen). En esta serie **no hay** relleno de −3024 fuera del círculo de escaneo (FOV).
- Valores de referencia: **aire ≈ −1000**, **agua = 0**, **cerebro ≈ 20–45**, **hueso > 300** HU.
- Centro del corte [256,256] = 30 HU → tejido cerebral.

Se muestran tres versiones: crudo con rango completo, HU con **ventana de cerebro** [−15, 75] y HU con **ventana de hueso** [−1000, 1500]. Así se ve que la misma matriz "revela" tejidos distintos según la ventana.

### Elección del corte 9
- La serie tiene 10 tajadas de la **base del cráneo**. Las primeras están dominadas por hueso (peñascos, senos) con poco parénquima.
- El corte 9 es el más alto: muestra **cerebelo y lóbulos temporales** con la menor proporción de hueso → zonas amplias de **tejido blando homogéneo** para medir ruido.

---

## 5. Anonimización (solo TC)

### Por qué
Antes de compartir o entregar imágenes médicas hay que **quitar los datos que identifican al paciente** (protección de datos / ética). Solo se aplica a la TC: la foto no tiene datos personales y el NIfTI de la RM no tiene encabezado DICOM.

### Cómo
```python
anon = copy.deepcopy(ds)                 # el original en memoria NO se toca
for campo in CAMPOS_ANONIMIZAR: anon.<campo> = ""
anon.remove_private_tags()               # tags privados del fabricante
anon.save_as("..._anonimo.dcm")          # copia junto al original
```
- **Se vacían:** datos del paciente (nombre, ID, fecha de nacimiento, sexo, edad, peso, dirección, teléfono, comentarios, historia), del **personal médico** (médico remitente, lector, operador) y de la **institución** (nombre, dirección).
- **Se eliminan los tags privados** del fabricante (pueden contener información identificable no estándar).
- **Se conservan** los campos técnicos (bits, espaciado, rescale, ventana, fabricante, kernel) y el `Pixel Data`, porque se necesitan para procesar.
- Se usa `deepcopy` porque una asignación normal (`anon = ds`) solo copia la referencia y se modificaría el original.

### Verificación
Se releen del disco el original y el anonimizado y se comparan campo por campo:
- `PatientName`: `Doe^Pierre` → vacío; `PatientID`: `54879843` → vacío; `PatientSex`: `M` → vacío; `PatientAge`: `025Y` → vacío.
- Campos técnicos: **idénticos**.
- Tags privados: se reducen a 0; `Pixel Data` idéntico: `True`.

---

## 6. Teoría de color

### Idea
Una imagen a color son **3 matrices superpuestas** (R, G, B). La RM y la TC **no tienen color**: miden señal o atenuación; el gris es solo cómo el monitor las pinta. Las fórmulas se implementaron **a mano, en forma matricial** (sin OpenCV).

### Conversiones
**Gris (luminancia):**
```
Y = 0,299 R + 0,587 G + 0,114 B
```
No es el promedio simple: el ojo humano es **más sensible al verde**, luego al rojo y mucho menos al azul.

**YUV** — cada píxel [R, G, B] multiplicado por una matriz 3×3:
```
| Y |   |  0,299   0,587   0,114 | | R |
| U | = | −0,147  −0,289   0,436 | | G |
| V |   |  0,615  −0,515  −0,100 | | B |
```
- **Y** = luminancia (casi toda la información estructural).
- **U, V** = crominancia (diferencias de color); se pueden **comprimir** sin que se note → se usa en transmisión de video y compresión.

**HSV:**
- **H** (tono) = ángulo en el círculo cromático, 0–360°; se calcula según cuál canal es el máximo.
- **S** (saturación) = (máx − mín) / máx.
- **V** (brillo) = máx(R, G, B).
- Se evita dividir por 0 en píxeles grises (máx = mín) con `np.where`.

### Verificación con un píxel
Píxel central [1009, 585] = (4, 4, 2): Gris = 0,299·4 + 0,587·4 + 0,114·2 = **3,77**, coincide con la función. HSV: H = 60°, S = 0,5, V = 0,016.

### Qué se observa
- **Y** es prácticamente la imagen en gris; **U y V** tienen poco detalle → por eso se comprimen.
- **HSV separa color de brillo:** H agrupa por tono aunque cambie la iluminación; S distingue colores vivos de grises. En zonas sin saturación (blancos/negros/grises) H no está definido y sale arbitrario.
- **Gris ponderado vs promedio:** diferencia media 3,9 niveles, máxima **31** (sobre todo en zonas muy verdes o azules).
- **Histogramas R, G, B:** cargados a la izquierda (escena **nocturna, subexpuesta**); el canal R está más a la derecha que el B por la **luz cálida**.

A partir de aquí: `imgs = {"Foto": gris, "RM": img_rm, "TC": img_tc}`.

---

## 7. Histograma y CDF

### Conceptos
- **Histograma:** gráfico de conteo. Eje X = valores de intensidad posibles; eje Y = cuántos píxeles tienen ese valor.
- **Una sola función** para las tres imágenes: foto (0–255), RM (16 bits) y TC (HU con negativos).
- **Eje Y logarítmico** en RM y TC: el fondo/aire es un pico enorme; sin log no se ve el resto.
- **CDF:** suma acumulada del histograma de izquierda a derecha; siempre crece de 0 a 1. Se lee: **"el p % de los píxeles tiene intensidad ≤ X"**. Se marcan los percentiles 50 % y 80 %.

### Lectura de los picos de la TC (rangos Hounsfield)

| Tejido | Rango HU | % de la imagen |
|---|---|---|
| Relleno fuera del FOV | −3100 a −2000 | 0,0 % |
| **Aire** | −1100 a −900 | **37,0 %** |
| Grasa | −150 a −50 | 2,9 % |
| Agua / LCR | −5 a 15 | 3,1 % |
| **Tejido cerebral** | 20 a 45 | **23,7 %** |
| **Hueso** | 300 a 3100 | **12,3 %** |

### Cómo se lee cada histograma
- **Foto:** izquierda = sombras, centro = medios tonos, derecha = altas luces. Esta foto nocturna se concentra en las sombras; los pocos píxeles en 255 son los bombillos.
- **RM:** 615 valores distintos (0 a 663), una parte muy pequeña del rango de 16 bits; pico grande en el fondo (≈ 0) y jorobas de los tejidos. Por eso se visualiza recortando en el percentil 99,5.
- **TC:** cada pico es un tejido: aire (≈ −1000), cerebro (20–45 HU) y la cola del hueso (> 300). El cerebro ocupa una franja estrechísima → **hace falta ventana** para verlo.

---

## 8. Ecualización del histograma

### Qué es
Redistribuye las intensidades para que el histograma quede lo más **plano** posible (la CDF se vuelve una **recta**) y se aproveche todo el rango de salida.

### Fórmula (implementada a mano)
$$s = \left\lfloor (L-1)\,\frac{CDF(v) - CDF_{min}}{1 - CDF_{min}} \right\rfloor, \quad L = 256$$

- Se construye una **tabla de búsqueda** (LUT): para cada valor posible *v* se calcula *s*, y luego cada píxel se reemplaza por `tabla[v − mínimo]`.
- **Por qué restar CDF_min:** el fondo tiene muchísimos píxeles con el valor mínimo; con la fórmula básica ese nivel quedaría gris en lugar de negro.
- **Efecto:** los valores **muy frecuentes** quedan separados entre sí (se estiran) y los **poco frecuentes** se juntan (se comprimen).
- La salida es de **8 bits para las tres** imágenes → comparables en la misma escala.
- Se aplica a la foto **en gris**: ecualizar R, G y B por separado altera los colores.
- En la TC, tras ecualizar **los valores ya no son HU** (se pierde el significado físico).

Para comparar, la original se estira **linealmente** a 0–255 (`a_8bits`) y se dibujan ambos histogramas, ambas CDF y la recta ideal.

### ¿La ecualización aumenta el ruido? (medición)
Se mide en la zona homogénea: **contraste** = σ de toda la imagen; **ruido** = σ del parche; **SNR** = μ/σ (en la TC, (μ − aire)/σ porque en HU el 0 es el agua, no la ausencia de señal).

| | Contraste σ (orig → ecual.) | Niveles distintos | σ del parche (orig → ecual.) | SNR parche |
|---|---|---|---|---|
| Foto | 44,8 → 74,6 | 256 → 142 | 9,76 → 3,73 (×0,4) | 14,8 → 63,8 |
| RM | 47,8 → 73,5 | 615 → 112 | 0,59 → 33,66 (**×57**) | 1,2 → 1,9 |
| TC | 57,5 → 76,3 | 2897 → 193 | 0,58 → 15,77 (**×27**) | 150 → 7,8 |

### Conclusión
- El **contraste aumenta** en las tres; el número de niveles distintos **baja** (varios niveles de entrada caen en el mismo de salida).
- Qué pasa con el ruido depende de **dónde cae la zona homogénea en el histograma**:
  - **TC:** el cerebelo está en los niveles **más frecuentes** → la ecualización los estira → la σ del ruido se multiplica ×27 y la SNR cae.
  - **RM:** el parche cae en el **fondo** (valores ≈ 0–6), que es el nivel más frecuente de toda la imagen → se estira muchísimo: σ ×57.
  - **Foto:** la pared (≈ 144) está en medios tonos **poco frecuentes** en esta foto nocturna → se comprimen → la σ baja. **No se quitó ruido**: se aplastó todo el detalle de esa zona.
- La ecualización es una transformación **punto a punto**: no distingue señal de ruido, así que **no reduce el ruido**. Para eso están los **filtros**. En clínica se prefiere el **windowing** (ventana/nivel) porque conserva los HU.

---

## 9. Modelamiento del ruido vía histograma

### Por qué
Para elegir **qué filtro** usar, primero hay que saber **qué tipo de ruido** hay. Cada filtro está pensado para un tipo de ruido; usar el equivocado borra información útil.

### Método
Se toma una **zona homogénea de 50×50** (sin bordes ni textura). Si no hubiera ruido, todos sus píxeles valdrían lo mismo y el histograma sería una sola barra; **la forma que aparece es la distribución del ruido**.

| Imagen | Dónde se mide | Ruido esperado y por qué |
|---|---|---|
| Foto | Pared lisa iluminada (filas 200–249, col. 600–649) | **Gaussiano:** ruido electrónico/térmico del sensor |
| RM | Parche en (200, 200) | **Rician:** la RM es la magnitud √(R² + I²) de dos canales con ruido gaussiano. Con SNR alto (> 3) se parece a una **gaussiana**; en el fondo sin señal tiende a **Rayleigh** |
| TC | Cerebelo (filas 330–379, col. 170–219) | **Poisson** en las proyecciones (el detector cuenta fotones); tras la reconstrucción (FBP) cada píxel combina miles de mediciones y, por el teorema del límite central, tiende a **gaussiano** |

### Estadísticas del parche
| | μ | σ | Asimetría | SNR |
|---|---|---|---|---|
| Foto | 144,0 | 9,76 | −0,54 | 14,8 |
| RM | 3,12 | 1,32 | 1,19 | 2,4 |
| TC | 31,1 HU | 5,59 | −0,02 | 184,5 |

- **μ** = señal (centro de la campana). **σ** = cantidad de ruido (ancho de la campana).
- **Asimetría** ≈ 0 → simétrica (gaussiana); positiva → cola a la derecha (típica Rician/Rayleigh).
- **SNR** = μ/σ; en la TC (μ + 1000)/σ: la señal se mide desde el aire.

### Ajuste y prueba de Kolmogorov-Smirnov (K-S)
- Candidatas: **Foto** → Gaussiana, Uniforme. **RM** → Gaussiana, Uniforme, Rician (y Rayleigh si fuera fondo). **TC** → Gaussiana, Uniforme, Poisson.
- Parámetros estimados con los propios datos:
  - Normal: μ y σ del parche.
  - Uniforme: mínimo y máximo **del parche**.
  - Rician: máxima verosimilitud (`stats.rice.fit`) **con valor inicial** b₀ = √(E[x²] − 2σ²)/σ; sin él el ajuste se va a escala 0 y D = 1.
  - Poisson: λ = media, desplazando los HU para que el mínimo sea 0 (Poisson solo existe para enteros ≥ 0). Además se hace **chi-cuadrado** (adecuado para distribuciones discretas).
- **D** = máxima distancia entre la CDF empírica y la del modelo. **El mejor ajuste es el de menor D.**
- **p-valor (α = 0,05):** H₀ = "los datos siguen la distribución". p > 0,05 → no se rechaza.
- Detalle técnico: en SciPy reciente se pasa `stats.norm(mu, sigma).cdf` en lugar del texto `"norm"` con `args`, que falla.

### Resultados

| Imagen | Gaussiana D | Uniforme D | Otra D | Mejor ajuste |
|---|---|---|---|---|
| Foto | **0,054** | 0,383 | — | **Gaussiana** |
| RM | 0,233 | 0,566 | Rician **0,215** | **Rician** |
| TC | **0,054** | 0,275 | Poisson 0,147 | **Gaussiana** |

TC: λ = 16,08 vs varianza = 31,23 → **Poisson exige λ = varianza**, no se cumple → se descarta.

### Por qué los p-valores salen ≈ 0
Con **2500 píxeles de valores enteros** K-S casi siempre rechaza:
1. Muchos valores repetidos (**empates**): la CDF empírica avanza en escalones y no coincide con ninguna curva continua.
2. Con muestras grandes la prueba detecta diferencias minúsculas.
3. μ y σ se estiman con los mismos datos que se prueban.

Por eso **se compara D entre candidatas** (junto con el ajuste visual de la curva sobre el histograma) en vez de fiarse del p-valor.

### 9.1 Análisis complementario del ruido residual
Se calcula el ruido puro **n(x,y) = I(x,y) − media del parche** y se evalúa:
- **Forma:** asimetría, curtosis, **Shapiro-Wilk**, % de valores > 3σ y > 4σ (en una gaussiana ≈ 0,27 % y ≈ 0,006 %).
- **Correlación espacial** con el vecino horizontal y vertical: ≈ 0 si el ruido es independiente; alta si algo lo suavizó.
- **Factor de Fano** D = varianza / media sobre los **valores crudos** (sin HU). Un proceso de **Poisson** tiene D ≈ 1.
- **Q-Q plot** contra la normal: una gaussiana perfecta cae sobre la recta.

Resultados clave (TC):
- Asimetría −0,018, curtosis −0,035 → muy cercano a gaussiano.
- |z| > 3σ = 0,28 % (≈ 0,27 % teórico de una gaussiana).
- **Fano = 0,0296 ≪ 1** → se **descarta Poisson**; consistente con ruido **gaussiano de reconstrucción (FBP)**.
- Correlación con el vecino ≈ 0,84–0,91 → ruido **correlacionado** (la reconstrucción y el reformateo lo suavizan).

Foto: correlación 0,87–0,93 (procesado de la cámara y JPEG), curtosis ≈ 1 (colas algo más pesadas por la leve variación de iluminación de la pared).

RM: asimetría 1,19 y curtosis 2,48 (cola a la derecha, lejos de la gaussiana), Shapiro-Wilk W = 0,879 (el más bajo de los tres), correlación ≈ 0,65–0,67 y Fano 0,56. Esta forma asimétrica es la esperada de un parche de **fondo** (Rayleigh), no de tejido (ver sección 15).

| | Foto | RM | TC |
|---|---|---|---|
| σ del ruido | 9,77 | 1,32 | 5,59 |
| Asimetría | −0,54 | 1,19 | −0,02 |
| Curtosis | 0,96 | 2,48 | −0,03 |
| Shapiro-Wilk W | 0,984 | 0,879 | 0,996 |
| \|z\| > 3σ | 0,84 % | 0,76 % | 0,28 % |
| Correlación h / v | 0,93 / 0,87 | 0,64 / 0,67 | 0,84 / 0,91 |
| Fano σ²/μ (crudo) | 0,66 | 0,56 | **0,03** |

**Shapiro-Wilk** también rechaza la normalidad estricta por las mismas razones que K-S (tamaño de muestra y discretización); los escalones en el Q-Q plot son los valores enteros repetidos.

**Conclusión de la sección:** el ruido es **aproximadamente gaussiano y aditivo** → los filtros adecuados son **media** y **gaussiano** (y el adaptativo de Lee).

---

## 10. Kernel y convolución

### Implementación a bajo nivel
Motor de convolución **solo con ciclos `for` y listas de Python** (sin `scipy.ndimage` ni `cv2`), para controlar explícitamente los bordes.

**Algoritmo:**
1. Validar: imagen 2D, kernel **cuadrado** y de tamaño **impar** (para que tenga centro).
2. Opcional: rellenar bordes.
3. Para cada posición (i, j): multiplicar kernel × vecindario elemento a elemento, sumar, y escribir el resultado **en una imagen nueva** en la posición del centro.

```python
for i in range(alto_out):
    for j in range(ancho_out):
        suma = 0.0
        for u in range(k):
            for v in range(k):
                suma += ker[u][v] * img[i + u][j + v]
        salida[i][j] = suma
```

**Por qué una imagen nueva:** si se escribiera sobre la misma, los píxeles ya modificados afectarían el cálculo de los siguientes.

### Manejo de bordes
- `recuperar_bordes=False`: no se calcula donde el kernel se sale → se pierden ⌊k/2⌋ filas/columnas por lado. Kernel 3×3: 512×512 → **510×510**.
- `recuperar_bordes=True`: se **rellena** antes y la salida conserva el tamaño:
  - `replicar`: repite el píxel del borde (`a a | a b c`). **Es el que se usa.**
  - `reflejar`: espejo (`c b a | a b c`).
  - `ceros`: rellena con 0 (oscurece los bordes en un filtro de suavizado).

### Otros detalles
- Un kernel de suavizado debe **sumar 1** para no cambiar el brillo medio.
- Copiar matrices con `.copy()` / `.tolist()`: `y = x` solo copia la referencia.
- `voltear_kernel`: la convolución matemática voltea el kernel 180°; sin voltear es **correlación**. Para kernels simétricos (media, gaussiano, Laplaciano) da igual; en Sobel solo cambia el signo.

### 10.1 Filtros de paso bajo: media y gaussiano

**El tamaño del kernel se elige en el código.** En la celda de los kernels hay tres bloques (`# --- 3x3 ---`, `# --- 5x5 ---`, `# --- 7x7 ---`) con el gaussiano σ = 1 **sin normalizar**. Se deja descomentado **solo uno** y el resto se ajusta solo:
```python
TAM_KERNEL = kernel_gaussianoSNM.shape[0]                              # 3, 5 o 7
kernel_gaussiano = kernel_gaussianoSNM / np.sum(kernel_gaussianoSNM)   # ganancia unitaria
kernel_media = np.ones((TAM_KERNEL, TAM_KERNEL)) / TAM_KERNEL ** 2     # 1/n²
NOMBRE_GAUSS = f"Gaussiano {TAM_KERNEL}x{TAM_KERNEL}"                  # títulos y claves
```
El motor de convolución ya aceptaba cualquier kernel cuadrado impar; lo que estaba fijo eran los pesos y los nombres "3x3" en títulos y claves. Ahora el suavizado escogido se propaga a Sobel, al Laplaciano (sección 11) y al pipeline final (sección 13). Sobel y el Laplaciano siguen siendo 3×3 porque son derivadas, no suavizados. La comparación 3×3 vs 5×5 está en la [sección 10.2](#102-comparación-kernel-33-vs-55).

Dejan pasar las **bajas frecuencias** (zonas lisas) y atenúan las **altas** (bordes y ruido). Compromiso: **más suavizado = menos ruido, pero menos detalle**.

- **Media 3×3:** todos los pesos = 1/9. Trata igual un píxel de borde que uno de zona lisa.
- **Gaussiano 3×3 (σ = 1):** $G(x,y)=\frac{1}{2\pi\sigma^2}e^{-\frac{x^2+y^2}{2\sigma^2}}$. Pesa más el centro.
  - Sin normalizar los pesos suman **0,779** → la imagen se oscurecería un 22 %. Por eso **se divide por la suma** (ganancia unitaria).

**Resultado:** ambos bajan la σ del parche en las tres imágenes; la **media** reduce un poco más el ruido pero difumina más; el **gaussiano** conserva mejor el detalle.

**Por qué la reducción es menor de lo teórico:** para ruido independiente, la media 3×3 dejaría σ/3 (33 %). Aquí deja ~80–90 % porque el ruido está **correlacionado espacialmente** (sección 9.1): los vecinos se parecen entre sí y promediarlos cancela menos ruido.

### 10.2 Comparación kernel 3×3 vs 5×5

Se ejecutó el notebook completo dos veces: una con el bloque 3×3 activo y otra con el 5×5. Todo lo anterior a la sección 10 (carga, histogramas, ruido) da **idéntico**; cambia solo lo que usa el suavizado.

**Kernel gaussiano (σ = 1)**

| | 3×3 | 5×5 |
|---|---|---|
| Suma de los pesos sin normalizar | 0,779 | 0,982 |
| Peso del centro (normalizado) | 0,204 | 0,162 |
| Peso de la media | 1/9 | 1/25 |
| Píxeles perdidos sin recuperar bordes (512×512) | 510×510 | 508×508 |

La suma sin normalizar sube porque el 5×5 abarca más de la campana gaussiana (el 3×3 la corta en ±1σ y pierde el 22 %).

**Ruido restante y nitidez conservada (menor ruido = mejor, mayor nitidez = mejor)**

| Imagen | Filtro | Ruido 3×3 | Ruido 5×5 | Nitidez 3×3 | Nitidez 5×5 |
|---|---|---|---|---|---|
| Foto | Media | 90 % | **80 %** | 79 % | **55 %** |
| Foto | Gaussiano | 91 % | 87 % | 83 % | 75 % |
| RM | Media | 80 % | 77 % | 71 % | **45 %** |
| RM | Gaussiano | 81 % | 79 % | 75 % | 66 % |
| TC | Media | 86 % | **74 %** | 93 % | 81 % |
| TC | Gaussiano | 88 % | 84 % | 94 % | 91 % |

**Efecto sobre la detección de bordes** (σ de la respuesta en la zona homogénea; menos = menos "bordes falsos")

| | Sobel 3×3 | Sobel 5×5 | Lapl. 4 vec. 3×3 | Lapl. 4 vec. 5×5 | Lapl. 8 vec. 3×3 | Lapl. 8 vec. 5×5 |
|---|---|---|---|---|---|---|
| Foto | 14,74 (77 %) | 12,84 (67 %) | 3,9 | 3,2 | 11,3 | 9,2 |
| RM | 1,46 (54 %) | 1,18 (43 %) | 0,6 | 0,4 | 1,5 | 1,0 |
| TC | 8,70 (74 %) | 7,40 (63 %) | 2,7 | 2,1 | 7,6 | 6,0 |

(El % de Sobel es respecto a la magnitud sin suavizar: Foto 19,06, RM 2,73, TC 11,74.)

**Qué no cambia:** el filtro de **Lee** (sus ventanas 3×3, 5×5, 7×7 son independientes del kernel elegido) y todo lo anterior a la sección 10.

**Conclusiones de la comparación**
1. **El 5×5 quita más ruido pero borra más detalle.** Es el compromiso de los filtros de paso bajo: más vecinos promediados → más suavizado en todas partes, también en los bordes.
2. **La media sufre mucho más que el gaussiano al crecer.** La media 5×5 da el mismo peso (1/25) a 25 píxeles, incluidos los de las esquinas, y la nitidez cae fuerte (foto 79 → 55 %, RM 71 → 45 %). El gaussiano 5×5 casi no cambia (pierde 3–9 puntos) porque los pesos de la orilla son muy pequeños (0,003–0,022): el centro sigue mandando.
3. **El gaussiano 5×5 reduce poco más ruido que el 3×3** (2–4 puntos) por la misma razón: el anillo exterior pesa poco.
4. **Con 5×5 hay menos bordes falsos** en Sobel (−10 a −11 puntos) y en el Laplaciano, porque la imagen de entrada llega más suave; a cambio, los bordes reales salen más gruesos y difusos.
5. **Costo:** la convolución está hecha con `for`; 5×5 hace 25 multiplicaciones por píxel en vez de 9 (≈ 2,8 veces más operaciones) en las celdas de media y gaussiano. En el notebook completo la diferencia fue pequeña (≈ 2 min 30 s frente a ≈ 2 min 40 s), porque gran parte del tiempo se va en otras celdas (Sobel, Laplaciano, varianza local, Lee).
6. **El 7×7 con σ = 1 casi no difiere del 5×5 en el gaussiano** (sus pesos extra son ≤ 0,002; suma sin normalizar 0,999). Para que un gaussiano 7×7 suavice más habría que subir σ. La media 7×7 sí suaviza mucho más (pesos 1/49).
7. **Recomendación:** dejar **gaussiano 3×3** como suavizado previo a los bordes (mejor nitidez). Si se quiere menos ruido, es mejor usar el filtro de **Lee** que agrandar la media: Lee baja más el ruido sin perder nitidez, mientras que la media 5×5 la destruye.

---

## 11. Filtros de paso alto (detección de bordes)

### Idea
Un borde es un **cambio brusco de intensidad** (alta frecuencia) y se detecta con **derivadas**. Los filtros de paso alto resaltan bordes… **y también el ruido** (que son cambios bruscos píxel a píxel). Por eso se aplican sobre la imagen **ya suavizada** con el gaussiano 3×3.

### Métricas para comparar filtros con números
- **Ruido restante** = σ del parche filtrado / σ del parche original × 100 (menor = mejor).
- **Nitidez conservada** = magnitud de Sobel en los píxeles que en la original son borde fuerte (10 % más alto), como % de la original (mayor = mejor).

### 11.1 Sobel (primera derivada)
```
Gx = [[−1, 0, 1],      Gy = Gxᵀ
      [−2, 0, 2],
      [−1, 0, 1]]
```
- **Gx** detecta cambios horizontales → **bordes verticales**; **Gy** → bordes horizontales.
- El 2 central da más peso a la fila/columna del centro (suaviza un poco en la dirección perpendicular).
- **Magnitud** = √(Gx² + Gy²): qué tan fuerte es el borde.
- **Dirección** = arctan(Gy/Gx), con `arctan2` para que no falle cuando Gx = 0. Se muestra solo en el 15 % de píxeles con mayor magnitud (en zonas lisas el ángulo es aleatorio).

**¿Por qué suavizar primero? (σ de la magnitud en zona homogénea = "bordes falsos")**

| | Sin suavizar | Suavizada | % |
|---|---|---|---|
| Foto | 19,06 | 14,74 | 77 % |
| RM | 2,73 | 1,46 | 54 % |
| TC | 11,74 | 8,70 | 74 % |

### 11.2 Laplaciano (segunda derivada)
Por serie de Taylor, la segunda derivada discreta es [1, −2, 1]; sumando horizontal y vertical:

- **4 vecinos:** `[[0, 1, 0], [1, −4, 1], [0, 1, 0]]`
- **8 vecinos:** centro −8, resto 1 (incluye diagonales).
- **Con α:** `[[α, 1−α, α], [1−α, −4, 1−α], [α, 1−α, α]] / (1+α)`. α = 0 → 4 vecinos; al subir α pesan más las diagonales.

Todos **suman 0** → en zona plana la respuesta es 0. Responde en todas las direcciones a la vez (no da dirección) y el borde está donde la respuesta **cruza por cero** (doble línea clara/oscura).

**σ de la respuesta en zona homogénea:**

| | 4 vec. sin suavizar | 4 vec. suavizada | 8 vec. suavizada | α = 0,5 suavizada |
|---|---|---|---|---|
| Foto | 7,3 | 3,9 | 11,3 | 3,8 |
| RM | 3,1 | 0,6 | 1,5 | 0,5 |
| TC | 4,4 | 2,7 | 7,6 | 2,5 |

**Lectura:** el Laplaciano es **más sensible al ruido que Sobel** (segunda derivada amplifica más las altas frecuencias). El de **8 vecinos** es el más sensible; el de **α = 0,5** el más estable.

---

## 12. Operadores basados en la varianza

### Varianza local
Se calcula en una ventana deslizante con la misma convolución:
```
Var = E[x²] − (E[x])²       (ambas esperanzas = filtro de media n×n)
```
- **Baja** en tejido uniforme, **alta** en bordes y texturas.
- Se muestra `log(1 + Var)` porque la varianza de un borde es miles de veces la de una zona lisa.
- Ventanas 3×3, 5×5, 7×7: al crecer, el mapa es más suave pero los bordes se ensanchan.
- Se guarda en **caché** (`_LOCAL`) porque Lee reutiliza las mismas ventanas y la convolución con `for` es lenta.

### Filtro de uniformidad
```
R = 1 − 1 / (1 + Var)
```
0 en zona perfectamente uniforme, → 1 cuando la varianza crece.

**Tropiezo encontrado:** R **depende de la escala** de la imagen. Con valores originales la varianza del propio ruido ya es grande y R se satura (> 0,9 en gran parte de la imagen) → no distingue bordes de zonas lisas. **Solución:** normalizar antes (escala 0–16); así R ≈ 0 en zonas uniformes y solo crece en bordes.

### Filtro de Lee (adaptativo)
```
x̂ = μ_local + k · (x − μ_local)
k = máx(0, (σ²_local − σ²_ruido) / σ²_local)
```
- **Zona uniforme:** σ²_local ≈ σ²_ruido → k ≈ 0 → deja la **media local** (suaviza fuerte).
- **Borde:** σ²_local ≫ σ²_ruido → k ≈ 1 → **conserva el píxel** original.
- La **σ² del ruido** sale del parche homogéneo de la sección 9. En la TC se mide **dentro del tejido** (cerebelo) y no en el aire: así Lee reconoce correctamente el ruido del parénquima.
- Se muestra el **mapa k**: blanco = se conserva el píxel, negro = se reemplaza por la media.

### Filtro de Frost (no aplicado)
Es adaptativo, pensado para ruido **speckle multiplicativo** (radar, ultrasonido). No se aplica porque el ruido de las tres imágenes resultó **aditivo y gaussiano**, y en la TC la media local puede ser negativa (HU).

### Comparación de todos los filtros

| Imagen | Filtro | Ruido restante % | Nitidez % |
|---|---|---|---|
| Foto | Media 3×3 | 90 | 79 |
| Foto | Gaussiano 3×3 | 91 | 83 |
| Foto | Lee 3×3 / 5×5 / 7×7 | 90 / 80 / 75 | 84 / 79 / 79 |
| RM | Media 3×3 | 80 | 71 |
| RM | Gaussiano 3×3 | 81 | 75 |
| RM | Lee 3×3 / 5×5 / 7×7 | 81 / 78 / **77** | 100 / 100 / **100** |
| TC | Media 3×3 | 86 | 93 |
| TC | Gaussiano 3×3 | 88 | 94 |
| TC | Lee 3×3 / 5×5 / 7×7 | 86 / 75 / **68** | 100 / 100 / **100** |

**Conclusión:** Lee logra lo que no logran la media ni el gaussiano: **bajar más el ruido en zonas lisas sin difuminar más los bordes**. En la TC, Lee 7×7 deja el 68 % del ruido con el **100 %** de nitidez. En la RM conserva el 100 % de nitidez pero reduce poco el ruido (77 %), porque la σ² del ruido se midió en el fondo, donde es muy pequeña. En la foto gana menos: tiene mucha textura fina (rejas, malla) cuya varianza se parece a la del ruido, y Lee la suaviza como si fuera ruido.

---

## 13. Pipeline final y conclusiones

**Figura final:** original → **Lee 5×5** → **bordes** (Sobel sobre gaussiano 3×3), para las tres imágenes.

### Conclusiones (para decir en la presentación)
1. **Cada modalidad guarda valores distintos y necesita su propia visualización.** Foto: 8 bits por canal. RM: sin unidades físicas, se muestra recortando en el percentil 99,5. TC: se convierte a HU y se usa ventana (WC 30 / WW 90).
2. **El color es exclusivo de la foto.** Gris ponderado ≠ promedio (hasta 31 niveles de diferencia). Y concentra la estructura; U, V se comprimen; HSV separa tono de brillo.
3. **Anonimización:** datos de paciente, personal e institución vaciados; tags privados eliminados; campos técnicos y píxeles idénticos.
4. **El histograma permite leer la imagen.** En la TC cada pico es un tejido (aire 37 %, cerebro 23,7 %, hueso 12,3 %).
5. **La ecualización mejora el contraste pero no el ruido** (σ del parche ×27 en la TC, ×57 en la RM), y en la TC destruye el significado de los HU. En clínica se usa la ventana.
6. **El ruido es aproximadamente gaussiano** (menor D de K-S en foto y TC; en la TC Fano 0,03 ≪ 1 descarta Poisson). El p-valor ≈ 0 se debe al tamaño de muestra y a los valores enteros, por eso se compara D.
7. **El ruido está correlacionado espacialmente** (0,8–0,9) → la media 3×3 no llega al 33 % teórico.
8. **El tipo de ruido decide el filtro.** Para gaussiano aditivo: media y gaussiano; el **mejor compromiso es Lee**. Agrandar el kernel (3×3 → 5×5) quita algo más de ruido pero cuesta nitidez, sobre todo con la media (sección 10.2).
9. **Los bordes se detectan sobre la imagen suavizada** (Sobel y Laplaciano amplifican el ruido). El Laplaciano de 8 vecinos es el más sensible.
10. **Los operadores de varianza dependen de la escala:** la uniformidad R solo funciona normalizando antes.

**Pipeline recomendado para imágenes médicas:**
```
HU / ventana  →  filtro de Lee 5×5 (σ² del ruido medida en tejido homogéneo)  →  Sobel para bordes
```
La ecualización queda solo como apoyo visual.

---

## 14. Preguntas probables y cómo responderlas

**¿Por qué ordenaron las tajadas de TC por `ImagePositionPatient` y no por nombre?**
El nombre del archivo es un UID que no garantiza el orden anatómico. La coordenada z de `ImagePositionPatient` es la posición real en mm.

**¿Qué son las Unidades Hounsfield?**
Escala de atenuación de rayos X normalizada al agua: agua = 0, aire ≈ −1000. Se obtiene con `valor × RescaleSlope + RescaleIntercept`.

**¿Qué es la ventana (windowing)?**
Elegir qué rango de valores se mapea a los 256 grises del monitor: `[WC − WW/2, WC + WW/2]`. Lo que está por debajo se ve negro, por encima blanco. Permite ver tejidos de contraste muy bajo (cerebro en ~25 HU de diferencia).

**¿Por qué la RM no tiene unidades?**
La intensidad depende del equipo, la bobina, la secuencia (T1/T2) y la ganancia; no es una magnitud física calibrada como los HU.

**¿Por qué no se anonimiza la RM?**
Se usa en NIfTI, que no tiene encabezado DICOM con datos del paciente.

**¿Por qué el gris es 0,299R + 0,587G + 0,114B y no el promedio?**
Porque el ojo es más sensible al verde; los coeficientes reproducen la luminancia percibida (estándar ITU-R BT.601).

**¿Por qué restar CDF_min en la ecualización?**
Para que el valor mínimo (fondo, muy frecuente) quede en 0 (negro) y no en un gris.

**¿La ecualización elimina ruido?**
No. Es punto a punto, no distingue señal de ruido. Puede amplificarlo (TC ×27) si la zona homogénea cae en niveles frecuentes.

**¿Por qué el ruido de la TC es gaussiano si los rayos X son Poisson?**
Poisson aplica al conteo de fotones en las **proyecciones**. La reconstrucción (FBP) combina miles de mediciones por píxel; por el teorema del límite central la suma tiende a gaussiana. El Fano ≪ 1 y λ ≠ varianza lo confirman.

**¿Por qué el ruido de RM es Rician?**
La imagen es la magnitud √(R² + I²) de una señal compleja con ruido gaussiano en cada canal. Con SNR alto se aproxima a gaussiana; sin señal (fondo) es Rayleigh.

**Si todos los p-valores dan 0, ¿cómo deciden la distribución?**
Comparando el estadístico **D** (menor = mejor) y el ajuste visual. Con 2500 valores enteros K-S rechaza casi siempre por los empates y la sensibilidad a diferencias minúsculas.

**¿Qué es el factor de Fano?**
Varianza / media. Un proceso de Poisson tiene Fano = 1. Lejos de 1 indica que el proceso no es Poisson puro.

**¿Por qué implementar la convolución con `for` si es lenta?**
Para controlar y demostrar explícitamente el algoritmo y el manejo de bordes. Las librerías lo hacen igual pero optimizado.

**¿Qué pasa si usan un kernel 5×5 en vez de 3×3?**
Quita más ruido pero difumina más los bordes. La media 5×5 pierde mucha nitidez (foto 79 → 55 %) porque pesa igual las esquinas; el gaussiano 5×5 casi no cambia porque sus pesos de la orilla son pequeños. En el código se cambia comentando el bloque 3×3 y descomentando el 5×5 (sección 10.2).

**¿Por qué el gaussiano 7×7 sale casi igual al 5×5?**
Con σ = 1, los pesos a 3 píxeles del centro son ≈ 0,0002–0,002: casi no aportan. El tamaño útil de un gaussiano lo decide σ (regla práctica: radio ≈ 3σ).

**¿Por qué el kernel debe ser impar?**
Para tener un píxel central donde escribir el resultado.

**¿Por qué normalizar el gaussiano?**
Sus pesos sumaban 0,779; sin normalizar la imagen se oscurecería un 22 %. Un kernel de suavizado debe sumar 1.

**¿Por qué la media 3×3 no reduce el ruido a σ/3?**
Esa predicción supone ruido independiente; aquí está correlacionado (≈ 0,8–0,9 con el vecino), así que promediar vecinos parecidos cancela menos.

**¿Diferencia entre Sobel y Laplaciano?**
Sobel = 1.ª derivada, da magnitud **y dirección**. Laplaciano = 2.ª derivada, isotrópico (sin dirección), el borde está en el cruce por cero, y es más sensible al ruido.

**¿Por qué Lee es mejor que la media?**
Es adaptativo: usa la varianza local para decidir cuánto suavizar. En zonas lisas promedia, en bordes conserva el píxel.

**¿Por qué no Frost?**
Está diseñado para ruido speckle multiplicativo (ultrasonido, radar). El ruido aquí es aditivo gaussiano.

**¿Por qué la uniformidad R falló al principio?**
Depende de la escala: con valores grandes, hasta el ruido tiene varianza grande y R se satura cerca de 1. Hay que normalizar antes.

---

## 15. Puntos a revisar antes de presentar

Al revisar el notebook encontré **inconsistencias entre algunos textos (markdown) y las salidas que muestra el código**. Parece que algunos textos se escribieron con otra imagen de RM (una RM pélvica T2 con vejiga, en DICOM de 16 bits) y no se actualizaron al cambiar al NIfTI de cerebro. Conviene corregirlos antes de presentar, porque un profesor puede notarlo.

Comparado con una ejecución limpia de todo el notebook (kernel 3×3):

| Sección | Lo que dice el texto | Lo que da la ejecución |
|---|---|---|
| 7 (lectura de histogramas) | RM con picos en ~2000 y ~7000, "orina de la vejiga", "T2", saturados en 65535 | RM de cerebro T1, valores de 0 a 663, 615 valores distintos, 0 saturados |
| 8 (conclusión) | σ del parche de RM "se multiplica ≈ ×10", "parche de vejiga" | σ de la RM **×57** (0,59 → 33,66) |
| 9 (tabla) | Parche de RM en "interior de la vejiga (filas 150–199)" | El código usa `(200, 200)` marcado como **PROVISIONAL** |
| 9 (parche RM) | Se asume tejido con SNR alto | μ = 3,1, σ = 1,3, asimetría 1,19, curtosis 2,5 → **es fondo/aire**, no tejido. En fondo el modelo correcto sería **Rayleigh** (`ES_FONDO["RM"]` está en `False`) |
| 9 (interpretación) | RM: "no se rechaza Gaussiana ni Rician, p ≈ 0,08", "SNR ≈ 10,7" | RM: ambas rechazadas (p = 0), D = 0,233 y 0,215, SNR = 2,4 |
| 9.1 (lectura) | RM interpolada "de 320×224 a 512×512" | La RM actual es 256×256; correlación 0,64–0,67 |
| 12 (uniformidad) | σ² del ruido "≈ 440 000 en la RM"; "contorno de la vejiga y del recto" | σ² del ruido de la RM ≈ 1,7; imagen de cerebro |
| 12 (Lee) | RM: "Lee 7×7 deja 75 % con 90 % nitidez (media 3×3: 87 % y 80 %)" | RM: Lee 7×7 77 % con 100 %; media 3×3 80 % y 71 % |
| 13 (conclusiones 1, 5, 6, 8) | RM "16 bits, 6512 niveles entre 2 y 65535"; RM σ 20,3 → 75,4; ×9,5; D RM 0,025; Lee RM 75 % / 90 % | RM 0 a 663 (615 valores); σ 47,8 → 73,5; ×57; D RM 0,215; Lee RM 77 % / 100 % |
| 13 (conclusión 9) | Sobel suavizado al "65–77 %", Laplaciano al "53–61 %" | Sobel 54–77 %; Laplaciano: Foto 53 %, RM 19 %, TC 61 % |

**Recomendación:**
1. Elegir un parche de RM que esté **dentro del tejido cerebral** (por ejemplo en sustancia blanca) y verificarlo con la figura de la sección 9 (rectángulo rojo).
2. Volver a ejecutar todo el notebook (`Run All`).
3. Actualizar los textos de las secciones 7, 8, 9, 9.1, 12 y 13 con los números nuevos.
4. Si se quiere mostrar el caso Rayleigh, se puede medir un **segundo parche en el fondo** de la RM con `ES_FONDO["RM"] = True`.

Los números de este documento salen de una **ejecución limpia del notebook completo** (con kernel 3×3, y otra con 5×5 para la sección 10.2). Si se cambia el parche de la RM, hay que actualizar las filas de la RM en las secciones 8, 9, 9.1, 10.2, 11 y 12.

---

## 16. Guion de la presentación (8 min, 4 personas, en el notebook)

La presentación es **sin diapositivas**: se muestra directamente `ProyectoDicom.ipynb`. Son **2 minutos por persona**, en el mismo orden del notebook. Así nadie tiene que saltar hacia atrás: cada persona sigue bajando desde donde quedó la anterior.

### Preparación (antes de empezar)

1. **Ejecutar todo antes** (`Run All`, tarda ≈ 2–3 min) y **guardar**. Durante la presentación **no se ejecuta ninguna celda**: la convolución está hecha con `for` y tarda; solo se muestran las salidas ya generadas.
2. Dejar activo el **kernel 3×3** (sección 10.1). La comparación con 5×5 se cuenta con la tabla de la sección 10.2 de este documento.
3. **Ocultar el código** para que solo se vean figuras y textos. En VS Code: paleta (`Ctrl+Shift+P`) → *Notebook: Collapse All Cell Inputs*. En Jupyter: *View → Collapse All Code*.
4. Usar el **índice de encabezados** para saltar rápido entre secciones (en VS Code, el panel *Outline*; en Jupyter Lab, *Table of Contents*).
5. **Zoom** del editor al 110–125 % para que se lean los títulos de las figuras.
6. Que **una sola persona maneje el computador** y las demás le digan "baja a la sección X", o que cada una tome el mouse en su turno. En cualquier caso, ensayar los cambios.
7. Corregir antes los textos de la RM que no cuadran (sección 15 de este documento), o hablar solo de la foto y la TC en la parte de ruido.

### Orden y reparto

| Orden | Persona | Temas | Secciones del notebook | Tiempo |
|---|---|---|---|---|
| 1 | Persona 1 | Introducción + **espacios de color** (RGB, HSV, YUV) | Encabezado, 1, 6 | 0:00 – 2:00 |
| 2 | Persona 2 | **Histograma** + **ecualización** | 7, 8 | 2:00 – 4:00 |
| 3 | Persona 3 | **Modelamiento del ruido** + **filtros de paso bajo** | 9, 10, 10.1 | 4:00 – 6:00 |
| 4 | Persona 4 | **Paso alto** (Sobel, Laplaciano) + **operadores de varianza** + conclusión | 11, 12, 13 | 6:00 – 8:00 |

Las secciones 2 a 5 del notebook (detalle de la foto, encabezados NIfTI/DICOM, anonimización) **no tienen tiempo propio**: la persona 1 las menciona en una frase al pasar. Si preguntan por ellas, las respuestas están en las secciones 2–5 de este documento.

---

### Persona 1: Introducción y espacios de color (0:00 – 2:00)

**Qué mostrar:**
1. La **tabla del encabezado** del notebook (foto, RM, TC).
2. La **figura de las 3 imágenes** al final de la sección 1.
3. Bajar a la **sección 6**: la figura grande de 3×4 (canales R, G, B; gris; Y, U, V; H, S, V) y los histogramas R, G, B.

**Qué decir:**
- (≈ 30 s) "Trabajamos tres imágenes distintas y les aplicamos las mismas técnicas para compararlas: una **foto** JPG de 8 bits por canal, una **resonancia** de cerebro en NIfTI que no tiene unidades físicas, y una **tomografía** DICOM que convertimos a **Unidades Hounsfield** con valor × slope + intercept y vemos con la ventana de cerebro del propio archivo. Antes de usarla, la TC se anonimizó."
- (≈ 1:30) Espacios de color:
  - "Una imagen a color son 3 matrices: R, G y B. La RM y la TC no tienen color; el gris es solo cómo las pinta el monitor."
  - "El **gris** no es el promedio: es 0,299 R + 0,587 G + 0,114 B, porque el ojo es más sensible al verde. En esta foto la diferencia con el promedio llega a 31 niveles."
  - "En **YUV**, la Y es prácticamente la imagen en gris: ahí está la estructura. U y V tienen poco detalle, por eso se comprimen en video."
  - "En **HSV** el color queda separado del brillo: H es el tono en grados, S la saturación y V el brillo."
  - "Los histogramas R, G, B están cargados a la izquierda: es una foto nocturna, subexpuesta."

**Frase de paso:** "Desde aquí trabajamos la foto en gris, para compararla con la RM y la TC. [Persona 2] va a mostrar cómo se leen sus valores con el histograma."

---

### Persona 2: Histograma y ecualización (2:00 – 4:00)

**Qué mostrar:**
1. **Sección 7**: la figura de 3×3 (imagen, histograma, CDF) y la salida con los porcentajes por tejido de la TC.
2. **Sección 8**: la figura de 3×4 (original, ecualizada, histogramas, CDF) y la tabla "contraste σ / σ del parche".

**Qué decir:**
- (≈ 1:00) Histograma:
  - "El histograma cuenta cuántos píxeles tiene cada valor. Lo implementamos a mano y acepta negativos, porque en HU el aire vale ≈ −1000."
  - "En la TC cada pico es un tejido: **aire 37 %, cerebro 23,7 %, hueso 12,3 %**. El cerebro ocupa una franja muy estrecha; por eso hace falta la ventana."
  - "En la RM y la TC usamos escala logarítmica porque el fondo es un pico enorme."
  - "La **CDF** es la suma acumulada y se lee como: el 50 % de los píxeles tiene un valor ≤ X."
- (≈ 1:00) Ecualización:
  - "La ecualización usa la CDF para repartir los valores en todo el rango; la CDF queda casi como una recta."
  - "Restamos CDF_min para que el fondo quede negro y no gris."
  - "El **contraste sube** en las tres imágenes, pero en la zona homogénea **el ruido de la TC se multiplica ×27**: la ecualización estira los niveles frecuentes y no distingue señal de ruido. Además, en la TC ecualizada los valores dejan de ser HU. Por eso en clínica se usa la ventana."

**Frase de paso:** "La ecualización no quita ruido. Para quitarlo primero hay que saber qué tipo de ruido tenemos, y eso lo muestra [Persona 3]."

---

### Persona 3: Modelamiento del ruido y filtros de paso bajo (4:00 – 6:00)

**Qué mostrar:**
1. **Sección 9**: la figura con los **rectángulos rojos** (parches de 50×50) y la figura de los **histogramas del ruido con las curvas ajustadas**. Señalar en la salida la línea "Mejor ajuste por K-S".
2. **Sección 9.1**: la línea del **Fano** de la TC.
3. **Sección 10.1**: la figura original / media / gaussiano con la σ del parche en los títulos.

**Qué decir:**
- (≈ 1:10) Ruido:
  - "Tomamos una zona homogénea de 50×50. Si no hubiera ruido, todos sus píxeles valdrían lo mismo: la forma del histograma **es** la distribución del ruido."
  - "Ajustamos varias distribuciones y las comparamos con **Kolmogorov-Smirnov**. Nos quedamos con la de **menor D**, porque con 2500 valores enteros el p-valor casi siempre da 0."
  - "En la foto y la TC gana la **gaussiana**. En la TC esperábamos Poisson, porque los rayos X cuentan fotones, pero el **factor de Fano da 0,03**, lejos de 1: la reconstrucción vuelve el ruido gaussiano."
- (≈ 50 s) Paso bajo:
  - "Para ruido gaussiano sirven la **media** y el **gaussiano**. Los implementamos con nuestra propia convolución, con ciclos for y manejo de bordes."
  - "El gaussiano pesa más el centro y conserva mejor los bordes. Su kernel 3×3 sin normalizar suma 0,779, así que lo normalizamos para no oscurecer la imagen."
  - "El tamaño del kernel se cambia en el código. Con **5×5** se quita más ruido, pero la media pierde mucha nitidez: en la foto pasa de 79 % a 55 %."
  - **Mediana** (no está en el notebook; decirlo solo si está en el temario o la preguntan): "La mediana sirve para ruido **sal y pimienta**. No la aplicamos porque nuestro ruido resultó gaussiano."

**Frase de paso:** "Ya con la imagen suavizada podemos buscar bordes, y eso lo muestra [Persona 4]."

---

### Persona 4: Paso alto, operadores de varianza y conclusión (6:00 – 8:00)

**Qué mostrar:**
1. **Sección 11.1**: la figura de Sobel (Gx, Gy, magnitud, dirección) y luego la comparación "sin suavizar vs suavizada".
2. **Sección 11.2**: la figura del Laplaciano (solo un vistazo).
3. **Sección 12**: la figura del **filtro de Lee** con el **mapa k** y la tabla de "ruido % / nitidez %".
4. **Sección 13**: la figura final original → Lee → bordes.

**Qué decir:**
- (≈ 50 s) Paso alto:
  - "Un borde es un cambio brusco, y se detecta con derivadas. **Sobel** es la primera derivada: Gx encuentra bordes verticales y Gy horizontales. Con los dos sacamos la magnitud y la dirección del borde."
  - "El **Laplaciano** es la segunda derivada: no da dirección y es todavía más sensible al ruido."
  - "Por eso siempre se aplican **sobre la imagen suavizada**: aquí se ve que, sin suavizar, el ruido aparece como bordes falsos."
- (≈ 50 s) Varianza:
  - "La **varianza local** es baja en zonas lisas y alta en bordes."
  - "El **filtro de Lee** la usa para decidir cuánto suavizar. En este mapa, lo blanco son bordes y ahí conserva el píxel; lo negro son zonas lisas y ahí promedia."
  - "En la TC deja el **68 % del ruido conservando el 100 % de los bordes**, cosa que no logran ni la media ni el gaussiano."
- (≈ 20 s) Cierre: "Nuestra recomendación para imágenes médicas es: **HU y ventana, después filtro de Lee, después Sobel**. La ecualización queda solo como apoyo visual. Gracias."

---

### Consejos para cumplir los 8 minutos

- **Ensayar con cronómetro** al menos dos veces. Cada persona debe saber qué figura muestra y en qué sección está.
- **No leer los textos del notebook en voz alta**: decir la idea con una cifra y pasar.
- Si alguien se pasa de tiempo, lo primero que se recorta es: el detalle de HSV (persona 1), la CDF (persona 2), la comparación 5×5 (persona 3) y el Laplaciano (persona 4).
- Para las preguntas del final, cada persona revisa antes las de su tema en la **sección 14** de este documento.

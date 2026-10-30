# Procesamiento de Imágenes Médicas (DICOM)

## Integrantes

| Nombre | Código |
|---|---|
| Alejandro Marin Hoyos | 2259353 |
| Cristhian Leonardo Albarracin Zapata | 1968253 |
| Yessica Fernanda Villa Nuñez | 2266301 |
| Manuel Antonio Vidales Duran | 2155481 |

Proyecto de análisis de imágenes médicas en formato DICOM (tomografía de cráneo y resonancia magnética). El notebook [ProyectoDicom.ipynb](ProyectoDicom.ipynb) incluye:

- Lectura de archivos DICOM y extracción de metadatos (`pydicom`).
- Conversión a Unidades Hounsfield (HU).
- Análisis del ruido residual en ROIs (Factor de Fano, normalidad con Shapiro-Wilk, asimetría y curtosis).
- Filtrado espacial por convolución implementado a bajo nivel (filtros gaussianos, manejo de bordes).

## Estructura

```
procesamiento de imagenes/
├── ProyectoDicom.ipynb        # Notebook principal
├── requirements.txt           # Dependencias
└── imagenes/
    ├── imagen plana/          # Imágenes JPG
    ├── resonancia magentica/  # Archivos .dcm y .nii
    └── tomografia craneo/     # Series DICOM de CT
```

## Requisitos

- Python 3.10 o superior (probado con 3.10.11)
- pip

## Instalación con entorno virtual `.venv`

Abre una terminal en la carpeta del proyecto.

### 1. Crear el entorno virtual

```bash
python -m venv .venv
```

### 2. Activar el entorno

**Windows (PowerShell):**

```powershell
.venv\Scripts\Activate.ps1
```

> Si PowerShell bloquea el script, ejecuta una vez:
> `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`

**Windows (CMD):**

```cmd
.venv\Scripts\activate.bat
```

**Linux / macOS:**

```bash
source .venv/bin/activate
```

Al activarse, verás `(.venv)` al inicio de la línea de la terminal.

### 3. Instalar las dependencias

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

## Ejecución

### Opción A: Jupyter Notebook (navegador)

Con el entorno activado:

```bash
jupyter notebook ProyectoDicom.ipynb
```

Se abrirá el navegador; ejecuta las celdas en orden (`Shift + Enter`) o usa **Run → Run All Cells**.

### Opción B: VS Code

1. Instala las extensiones **Python** y **Jupyter**.
2. Abre la carpeta del proyecto y luego `ProyectoDicom.ipynb`.
3. En la esquina superior derecha, haz clic en **Select Kernel** → **Python Environments** → elige `.venv`.
4. Ejecuta las celdas en orden o pulsa **Run All**.

> **Importante:** el notebook usa rutas relativas (por ejemplo `imagenes/tomografia craneo/...`), así que debe ejecutarse con la carpeta del proyecto como directorio de trabajo.

## Desactivar el entorno

```bash
deactivate
```

# TP2 – Reconstrucción 3D con cámara estéreo

I308 Visión Artificial – Universidad de San Andrés

**Integrantes:** Bianca Paz, Sofía Bottino y Agustina Barrena

Reconstrucción 3D de un objeto (un muñeco stormtrooper sentado) a partir de 31 pares estéreo tomados
alrededor de él, usando un tablero ChArUco como referencia global. El resultado es una nube de puntos
métrica (en mm), optimizada y exportada en `.ply`.

**Entregable principal:** [`salidas/final/stormtrooper_final.ply`](salidas/final/stormtrooper_final.ply)
(250.277 puntos, 6.8 MB, binario; se puede abrir con [MeshLab](https://www.meshlab.net/)).

**Altura estimada del muñeco:** 155.8 mm (percentil 99.9) vs. 152 mm medidos con regla → error del 2.5%.

## Pipeline

Todo el código está en [`reconstruccion3d.ipynb`](reconstruccion3d.ipynb), dividido en partes:

| Parte | Secciones | Qué hace |
|---|---|---|
| 1. Disparidad y nubes locales | 1–5 | carga la calibración, rectifica con `alpha=-1`, calcula la disparidad con SGBM (`blockSize=9`, `numDisparities=256`) y la pasa a 3D con `reprojectImageTo3D` |
| 2. Poses y marco global | 6–10 | detecta el tablero ChArUco, estima la pose con `solvePnP` y lleva cada nube al marco del tablero (26 de 31 pares con pose) |
| 3. Fusión y optimización | 11–15 | fusión → caja 3D → descarte de píxeles saturados → voxel 1 mm → outliers por radio (100 puntos en 5 mm) → exportación `.ply` |
| 4. Medición métrica | 16 | altura = −Z del punto más alto (el tablero está en Z = 0) |
| 5. Opcionales | 17–18 | normales + malla de Poisson, refinamiento de poses con ICP point-to-plane |

## Instalación

Requiere Python 3.12 (open3d todavía no tiene versiones para Python 3.13+).

```bash
conda create -n vision python=3.12
conda activate vision
pip install -r requirements.txt
```

Después, en VS Code o Jupyter, abrir `reconstruccion3d.ipynb` y elegir el kernel `vision`.

## Versiones

Los resultados del informe se generaron con:

| paquete | versión |
|---|---|
| Python | 3.12.13 |
| opencv-python | 5.0.0.93 |
| open3d | 0.20.0 |
| numpy | 2.5.1 |
| matplotlib | 3.11.1 |
| pyyaml | 6.0.3 |
| ipykernel | 7.3.0 |

## Ejecución

Correr el notebook de arriba hacia abajo (**Run All**). Tarda unos minutos: el cálculo de SGBM para los 31 pares
es lo más lento.

- Las celdas con `o3d.visualization.draw_geometries` abren una ventana 3D aparte y **la celda no termina hasta cerrarla**.
  Cerrarla con `Q` o `Esc` (no con "Forzar salida", que mata el kernel).
- Las Partes 1 y 2 guardan resultados intermedios en `salidas/` (nubes locales, nubes en el marco global y `poses.npy`).
  Una vez generados, las Partes 3 a 5 se pueden correr sin repetir las anteriores (solo hace falta la celda de imports).

## Estructura

```
Informe.pdf                 informe del trabajo
data/
  calib/                    pares del tablero de calibración (28 pares, cuadros de 24.2 mm)
  objeto1/                  pares estéreo del muñeco: left_{i}.jpg / right_{i}.jpg, i = 0..30
  stereo_calibration.pkl    resultado de calib-stereo: K, dist, R, T de las dos cámaras
reconstruccion3d.ipynb      todo el pipeline
figuras/                    figuras del informe 
salidas/                    resultados 
  final/stormtrooper_final.ply   nube final optimizada 
  final/stormtrooper_malla.ply   malla de Poisson 
```

## Notas

- **Versión de OpenCV:** usar 5.0. Con 4.13 el detector ChArUco no devuelve esquinas en los pares 25 y 28 y quedan
  24 poses en lugar de 26, lo que cambia todos los resultados siguientes.
- **Mapas de rectificación:** el notebook los recalcula con `alpha=-1`. Los que genera `calib-stereo` usan
  `alpha=0`, que hace un zoom de ×4.5 y deja el objeto fuera de la imagen rectificada.
- **Poisson** (sección 17) se llama con `n_threads=1`: con varios hilos, Open3D 0.20 se cuelga en esta nube
  ("Failed to close loop") y mata el kernel.

## Calibración

La calibración estéreo se hizo con la herramienta de la cátedra
[i308-calib](https://github.com/udesa-vision/i308-calib) (`calib-stereo`, configuración `cfg/stereo.yaml`
para la cámara ELP-USB3D1080P02). Su resultado está en `data/stereo_calibration.pkl`, así que no hace falta
volver a calibrar para correr el notebook.

---

# Anexo: herramienta de calibración (i308-calib)

Documentación original de la herramienta de la cátedra con la que se hizo la calibración estéreo.

## i308-calib
Calibration tool


## Installation:

Can be installed via pip:
    
    pip install -qq git+https://github.com/udesa-vision/i308-calib.git


## Camera Calibration

### Quick Start

run the monocular calibration tool cli command:

    calib-tool

### Arguments

- **video** str or int. 
Specifies the video device, on linux might be something like `/dev/video<N>`

- **resolution** str (optional)
 the requested resolution in the format "`<width>`x`<height>`" in pixels

- **checkerboard** str (optional) default=10x7
 the checkerboard layout in the format "`<width>`x`<height>`" in number of squares

- **square-size** str (optional) default=24.2
 the checkerboard square size in millimeters 

- **config** str (optional)
a .yaml configuration file


### Examples

calibrate camera 0 with default parameters:

    calib --video 0


calibrate with custom parameters:

    calib --video /dev/video3 --resolution 640x480 --checkerboard 9x6 --square-size 32.0 --data data_dir


calibrate with capture configuration file:

    calib --config cfg/capture.yaml



### Configuration File

Some configuration files are provided.

In order to copy the configuration files to the working directory run the command:

```bash

 copy-configs

```

This should create the folder `cfg/` with some configuration files.

Example of capture configuration file (.yaml):

```yaml

    # video device, on linux might be /dev/video<N>
    video: 0

    # name: str (optional)
    #   a name to identify the device
    name: My Cute Camera

    # resolution: str (optional)
    #   requested resolution in the format "<width>x<height>" in pixels
    resolution: 640x480


```


### Overwriting configuration parameters
Some configuration file parameters can be overwritten with command line arguments.

For example, the following command will use the configuration file but in the video 3.

    calib --video 3 --config cfg/capture.yaml 


## Stereo Calibration

run the stereo calibration tool cli command:

```bash
    calib-stereo
```

You can use the same arguments of the monocular tool for the stereo tool, for example:

```bash

    calib-stereo --video /dev/video3 --resolution 1280x480 --checkerboard 9x6 --square-size 32.0 --data data/stereo

```

### Configuration file
A configuration file for the Stereo camera ELP-USB3D1080P02 is provided.

To get the configuration files, run the command:

```bash

 copy-configs

```

And then edit in `cfg/stereo.yaml` and then ddapt it to your needs.

After that you can run the stereo tool using that configuration file:

```bash

    calib-stereo --config cfg/stereo.yaml

```

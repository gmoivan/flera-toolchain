# flera-toolchain

Repositorio de automatización de compilación y aprovisionamiento de dependencias binarias y SDKs versionados para Notas para OBS en [Flera](https://github.com/gmoivan/flera) (`apps/notas`).

## Propósito

Aprovisionar binarios precompilados, SDKs, cabeceras de desarrollo y entornos de compilación de forma determinista, rápida y reproducible para los pipelines de CI y release de Notas para OBS, evitando sobrecargas de compilación nativa en los runners de GitHub Actions.

## Dependencias y Workflows

| Dependencia / Workflow | Plataformas | Workflow | Formato de Release / Tag | Archivo generado / Propósito |
|---|---|---|---|---|
| **OBS Studio SDK** | Windows x64 | `.github/workflows/build-obs-sdk.yml` | `obs-<version>-r<run>` (ej. `obs-32.2.1-r9`) | `obs-sdk-win64.zip` |
| **OpenCV** | Linux x64, Windows x64 | `.github/workflows/build-opencv.yml` | `opencv-<version>-r<run>` (ej. `opencv-4.10.0-r12`) | `opencv-<version>-linux.tar.gz`, `opencv-<version>-windows-x64.zip` |
| **FFmpeg** | Windows x64 | `.github/workflows/build-ffmpeg.yml` | `ffmpeg-<version>-r<run>` (ej. `ffmpeg-7.1-r3`) | `ffmpeg-<version>-windows-x64.zip` |
| **Qt** | Linux x64, Windows x64 | `.github/workflows/build-qt-whisper.yml` | `qt-<version>-r<run>` (ej. `qt-6.11.0-r1`) | `qt-<version>-linux-x64.tar.gz`, `qt-<version>-windows-x64.zip` |
| **Whisper Model** | Agnóstico (binario) | `.github/workflows/build-qt-whisper.yml` | `whisper-base-r<run>` (ej. `whisper-base-r1`) | `ggml-base.bin` |
| **Canary Windows** | Windows x64 | `.github/workflows/canary-windows.yml` | N/A (programado semanal / manual) | Verificación continua de compatibilidad de Qt, MSVC, SDK de Windows y OBS Studio sobre `windows-2025` |

Los releases son inmutables: cada ejecución publica en un tag nuevo con su `run_number` (`-r<run>`) y nunca reemplaza assets existentes. Cada asset va acompañado de `<asset>.manifest.json` con su origen upstream, parámetros, versiones de herramientas y SHA-256 ([contrato](CONTRIBUTING.md#contrato-de-release)). Para adoptar un build nuevo, Flera actualiza la URL y el SHA-256 fijados.

## Acciones reutilizables

### `setup-windows-native` (`.github/actions/setup-windows-native`)

Composite action que prepara el entorno nativo de compilación en runners Windows (`windows-2025`):
- Configuración de Qt con `jurplel/install-qt-action` (usando el fork con soporte para la estructura de repositorios de Qt 6.11).
- Activación del toolchain MSVC x64 con `ilammy/msvc-dev-cmd`.
- Detección y exportación del Windows SDK completo más reciente (>= 10.0.20348.0) con soporte para componentes de usuario (`um`).
- Aprovisionamiento opcional de compilador `sccache`.
- Aprovisionamiento opcional de dependencias completas (OpenCV y FFmpeg precompilados desde releases de `flera-toolchain`, y modelo Whisper base verificado por SHA-256).

#### Consumo y fijación por SHA

Los flujos consumidores deben fijar la acción mediante el SHA de commit completo (40 caracteres) para garantizar inmutabilidad y reproducibilidad:

```yaml
- name: Setup Windows native build environment
  id: windows-native
  uses: gmoivan/flera-toolchain/.github/actions/setup-windows-native@<COMMIT_SHA>
  with:
    qt_version: "6.11.0"
```

#### Inputs

| Input | Tipo | Default | Requerido | Descripción |
|---|---|---|---|---|
| `qt_version` | string | `"6.11.0"` | No | Versión de Qt a instalar. |
| `qt_modules` | string | `"qtnetworkauth qtwebsockets"` | No | Módulos adicionales de Qt a instalar. |
| `qt_cache` | string | `"true"` | No | Activar caché de instalación de Qt. |
| `msvc_arch` | string | `"x64"` | No | Arquitectura destino de MSVC. |
| `select_windows_sdk` | string | `"true"` | No | Buscar y exportar la versión más reciente del Windows SDK completo (>= 10.0.20348.0). |
| `enable_sccache` | string | `"false"` | No | Instalar y exportar sccache para acelerar la compilación MSVC. |
| `sccache_version` | string | `"v0.8.2"` | No | Versión de sccache a descargar si está habilitado. |
| `enable_full_deps` | string | `"false"` | No | Descargar y configurar dependencias completas (OpenCV, FFmpeg y modelo Whisper). |
| `opencv_version` | string | `"4.10.0"` | No | Versión de OpenCV para paquete completo. |
| `opencv_release` | string | `"opencv-4.10.0-r1"` | No | Tag de release en `flera-toolchain` del archivo de OpenCV. |
| `opencv_sha256` | string | `"9e07cf01e78beec97ff336573827fc5c5d28b7222322867bd6718593cd3cf6ee"` | No | SHA-256 del archivo zip de OpenCV Windows x64. |
| `ffmpeg_version` | string | `"7.1"` | No | Versión de FFmpeg para paquete completo. |
| `ffmpeg_release` | string | `"ffmpeg-7.1-r1"` | No | Tag de release en `flera-toolchain` del archivo de FFmpeg. |
| `ffmpeg_sha256` | string | `"9e091d556d0319b17e7730f26af088f18d4c7be85a25a5d90df9bfd52bd76257"` | No | SHA-256 del archivo zip de FFmpeg Windows x64. |
| `whisper_model_revision` | string | `"5359861c739e955e79d9a303bcbc70fb988958b1"` | No | Revisión git del repositorio `whisper.cpp` en Hugging Face. |
| `whisper_model_sha256` | string | `"60ed5bc3dd14eea856493d334349b405782ddcaf0028d4b5df4088345fba2efe"` | No | SHA-256 del archivo `ggml-base.bin`. |

#### Outputs

| Output | Descripción |
|---|---|
| `windows_sdk_version` | Versión seleccionada de Windows SDK (ej. `10.0.26100.0`). |
| `qt_version` | Versión configurada de Qt. |
| `opencv_dir` | Directorio de configuración CMake de OpenCV (`OpenCV_DIR`). |
| `ffmpeg_dir` | Directorio raíz de FFmpeg (`FFMPEG_DIR`). |
| `whisper_model_path` | Ruta absoluta al archivo del modelo `ggml-base.bin`. |

## Módulos y configuración

- **OpenCV**: Compilación ligera con `core`, `imgproc`, `dnn`, `objdetect`.
- **FFmpeg**: Cabeceras (`include/libav*`), librerías de importación (`lib/*.lib`), archivos `pkgconfig` y binarios de ejecución (`bin/*.dll`).
- **OBS Studio SDK**: compila `libobs` y `obs-frontend-api` para la versión de OBS y Qt correspondiente (ej. Qt 6.11.0, MSVC x64) y publica sólo lo que consume el plugin ([`scripts/package-obs-sdk.ps1`](scripts/package-obs-sdk.ps1)): cabeceras de `libobs/` y `frontend/api/`, `build/config/`, las librerías de importación Release (`obs.lib`, `obs-frontend-api.lib`) y `COPYING`. No incluye DLLs: el plugin enlaza contra las librerías de importación y usa en ejecución las de OBS instalado. La estructura conserva las rutas del árbol de OBS (`OBS_STUDIO_DIR=<raíz>`, `OBS_BUILD_DIR=<raíz>\build`).
- **Qt**: Binarios y cabeceras preempaquetados de Qt 6.11.0 para Linux x64 (`gcc_64`) y Windows x64 (`msvc2022_64`), con módulos base y adicionales (`qtnetworkauth`, `qtwebsockets`) y configuración de paths relativos (`bin/qt.conf`).
- **Whisper Model**: Binario upstream del modelo `ggml-base.bin` de `whisper.cpp` (revisión `5359861c739e955e79d9a303bcbc70fb988958b1`) con hash SHA-256 verificado (`60ed5bc3dd14eea856493d334349b405782ddcaf0028d4b5df4088345fba2efe`).

## Consumidor

Flera descarga los activos públicos por URL de release, sin token, y verifica el SHA-256 fijado en `apps/notas/.github/workflows/ci.yml` y `release.yml`. Para el entorno de compilación en Windows, Flera referencia la acción `.github/actions/setup-windows-native` fijada por su commit SHA exacto.

## Contribución y releases

Ownership, flujo de cambios y contrato de release: [CONTRIBUTING.md](CONTRIBUTING.md).

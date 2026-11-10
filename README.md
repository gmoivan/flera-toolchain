# flera-toolchain

Repositorio de automatización de compilación y aprovisionamiento de dependencias binarias y SDKs versionados para Notas para OBS en [Flera](https://github.com/gmoivan/flera) (`apps/notas`).

## Propósito

Aprovisionar binarios precompilados, SDKs y cabeceras de desarrollo de forma determinista, rápida y reproducible para los pipelines de CI y release de Notas para OBS, evitando sobrecargas de compilación nativa en los runners de GitHub Actions.

## Dependencias y Workflows

| Dependencia | Plataformas | Workflow | Formato de Release / Tag | Archivo generado |
|---|---|---|---|---|
| **OBS Studio SDK** | Windows x64 | `.github/workflows/build-obs-sdk.yml` | `obs-<version>-r<run>` (ej. `obs-32.2.1-r9`) | `obs-sdk-win64.zip` |
| **OpenCV** | Linux x64, Windows x64 | `.github/workflows/build-opencv.yml` | `opencv-<version>-r<run>` (ej. `opencv-4.10.0-r12`) | `opencv-<version>-linux.tar.gz`, `opencv-<version>-windows-x64.zip` |
| **FFmpeg** | Windows x64 | `.github/workflows/build-ffmpeg.yml` | `ffmpeg-<version>-r<run>` (ej. `ffmpeg-7.1-r3`) | `ffmpeg-<version>-windows-x64.zip` |

Los releases son inmutables: cada ejecución publica en un tag nuevo con su `run_number` (`-r<run>`) y nunca reemplaza assets existentes. Cada asset va acompañado de `<asset>.manifest.json` con su origen upstream, parámetros, versiones de herramientas y SHA-256 ([contrato](CONTRIBUTING.md#contrato-de-release)). Para adoptar un build nuevo, Flera actualiza la URL y el SHA-256 fijados.

## Módulos y configuración

- **OpenCV**: Compilación ligera con `core`, `imgproc`, `dnn`, `objdetect`.
- **FFmpeg**: Cabeceras (`include/libav*`), librerías de importación (`lib/*.lib`), archivos `pkgconfig` y binarios de ejecución (`bin/*.dll`).
- **OBS Studio SDK**: compila `libobs` y `obs-frontend-api` para la versión de OBS y Qt correspondiente (ej. Qt 6.11.0, MSVC x64) y publica sólo lo que consume el plugin ([`scripts/package-obs-sdk.ps1`](scripts/package-obs-sdk.ps1)): cabeceras de `libobs/` y `frontend/api/`, `build/config/`, las librerías de importación Release (`obs.lib`, `obs-frontend-api.lib`) y `COPYING`. No incluye DLLs: el plugin enlaza contra las librerías de importación y usa en ejecución las de OBS instalado. La estructura conserva las rutas del árbol de OBS (`OBS_STUDIO_DIR=<raíz>`, `OBS_BUILD_DIR=<raíz>uild`).

## Consumidor

Flera descarga los activos públicos por URL de release, sin token, y verifica el SHA-256 fijado en `apps/notas/.github/actions/setup-windows-native/`, `apps/notas/.github/workflows/ci.yml` y `release.yml` (referencia para su M2; hoy no se ejecutan).

## Contribución y releases

Ownership, flujo de cambios y contrato de release: [CONTRIBUTING.md](CONTRIBUTING.md).

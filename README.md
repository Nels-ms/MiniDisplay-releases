# 🎵 MiniDisplay

Mini reproductor / **overlay** de música para Windows hecho en **.NET 8 (WPF)**.
Muestra un panel flotante con la canción que está sonando —carátula, título, artista,
barra de progreso, controles y app de origen— al estilo del aviso de cambio de pista de Windows 10.

Se integra con el sistema mediante las **System Media Transport Controls (SMTC)**, así
que funciona con cualquier reproductor compatible (Spotify, navegadores, etc.). Vive en la
**bandeja del sistema** y aparece automáticamente al cambiar de pista.

> 🖼️ **Vista previa del diseño:** abre [`preview.html`](preview.html) en el navegador
> para ver los presets del display, los temas, la paleta de acentos y la pantalla de configuración.

## ✨ Características

- **Overlay flotante** con fundido de aparición/desaparición y **auto-ocultado** (respeta el ratón encima).
- **Tres disposiciones (presets)** intercambiables del display:
  - **Completo** — carátula grande, controles, app de origen y barra de progreso.
  - **Compacto** — carátula pequeña en una sola fila con los controles.
  - **Mínimo** — solo texto (`Artista · App`) y controles.
- Carátula, **título con marquesina** (se desplaza si es largo), artista y app de origen.
- **Elementos configurables**: mostrar/ocultar carátula, barra de progreso y app de origen.
- Controles de reproducción: anterior / play-pausa / siguiente.
- **Barra de progreso** con búsqueda (seek) por clic o arrastre.
- **Tema claro/oscuro** conmutable en caliente y **42 colores de acento**.
- **Opacidad** ajustable del reproductor.
- Icono en la **bandeja del sistema** con menú contextual.
- Todo se ajusta **en vivo con vista previa** desde la pantalla de Configuración.
- **Iniciar con Windows** opcional (desde Configuración → *Inicio y sistema*).
- **Actualizaciones automáticas**: comprueba al arrancar y desde *bandeja → Buscar actualizaciones*.

## 🛠️ Requisitos

- Windows 10/11
- .NET 8 SDK
- Visual Studio 2022 (o `dotnet` CLI)

## 🚀 Cómo ejecutar

```bash
dotnet run --project MiniDisplay
```

O abre `MiniDisplay.sln` en Visual Studio y pulsa F5.

> ℹ️ Si compilas con la app **abierta** verás errores `MSB3021/MSB3027` (el `.exe` está
> bloqueado). No son errores de código: cierra la instancia (bandeja → Salir) y recompila.

## 📦 Distribución y actualizaciones

La app se distribuye mediante un **instalador** que la registra en el menú de inicio y en
"Aplicaciones instaladas", y se **actualiza sola** usando [**Velopack**](https://velopack.io)
con **GitHub Releases** como origen. También puede generarse un `.exe` autocontenido portable.


## 🧱 Arquitectura

El display se dibuja **por binding** a un único estado observable y su disposición se elige
con **plantillas (`DataTemplate`) intercambiables**, sin duplicar la lógica.

- **`Models/PlayerState.cs`** — estado observable (`INotifyPropertyChanged`), **fuente de
  datos única** a la que se enlazan todos los presets (título, artista, carátula, icono/nombre
  de app, reproducción, posición/duración y flags de visibilidad).
- **`Services/MediaSessionService.cs`** — envuelve SMTC. Handlers **separados** por evento
  (pista nueva / play-pausa / posición) y caché del icono de la app por `appId`.
- **`Views/MainWindow.xaml`** — el overlay. Un `ContentControl` muestra la plantilla del
  preset activo (`TplCompleto` / `TplCompacto` / `TplMinimo`); la barra de progreso queda fija
  debajo. Fundidos y auto-ocultado en el code-behind.
- **`Views/ConfiguracionView.xaml`** — ventana de ajustes (tema, elementos visibles, preset,
  opacidad y color de acento) con vista previa en vivo.
- **`Behaviors/MarqueeBehavior.cs`** — marquesina del título.
- **`Services/SettingsService.cs`** — carga/guarda ajustes y aplica el tema (intercambia el
  diccionario de brushes sin acumularlos).
- **`Services/StartupService.cs`** — arranque con Windows vía `HKCU\...\Run` (el registro es la
  fuente de verdad del estado).
- **`Services/UpdateService.cs`** — comprueba/descarga/aplica actualizaciones desde GitHub
  Releases (inactivo en desarrollo).
- **`Program.cs`** — punto de entrada propio: ejecuta Velopack **antes** de arrancar WPF.

## 📁 Estructura

```
MiniDisplay/
├─ Models/        # PlayerState, MediaInfo, AccentPalette, AppSettings
├─ Services/      # MediaSession (SMTC), Settings, Startup y Update
├─ Behaviors/     # MarqueeBehavior (marquesina del título)
├─ Converters/    # PlayPauseIconConverter
├─ Styles/        # Temas (claro/oscuro) y estilos de controles
├─ Views/         # MainWindow (overlay) y ConfiguracionView (ajustes)
├─ Resources/     # Iconos e imágenes
└─ Program.cs     # Punto de entrada (Velopack antes de WPF)
preview.html      # Vista previa del diseño (presets, temas, paleta, config)
release.ps1       # Script de empaquetado y publicación de versiones
DESPLIEGUE.md     # Guía de despliegue y actualizaciones
```

## 📦 Dependencias

- [H.NotifyIcon.Wpf](https://github.com/HavenDV/H.NotifyIcon) — icono de bandeja
- [Extended.Wpf.Toolkit](https://github.com/xceedsoftware/wpftoolkit) — controles extra
- System.Drawing.Common — extracción de iconos de aplicaciones
- [Velopack](https://velopack.io) — instalador y actualizaciones automáticas

## 🗒️ Notas

Las preferencias del usuario se guardan en
`%AppData%\MiniDisplay\settings.json`.

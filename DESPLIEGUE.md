# 🚀 Despliegue y actualizaciones — MiniDisplay

Guía paso a paso para **empaquetar**, **distribuir** y **actualizar** la aplicación.

---

## 📑 Índice

1. [Cómo funciona (conceptos)](#1-cómo-funciona-conceptos)
2. [Requisitos previos](#2-requisitos-previos)
3. [Publicar la primera versión](#3-publicar-la-primera-versión)
4. [Publicar una actualización](#4-publicar-una-actualización)
5. [Cómo se actualiza la app del usuario](#5-cómo-se-actualiza-la-app-del-usuario)
6. [Alternativa: ejecutable portable](#6-alternativa-ejecutable-portable)
7. [Piezas del sistema (dónde toca cada cosa)](#7-piezas-del-sistema-dónde-toca-cada-cosa)
8. [Problemas frecuentes](#8-problemas-frecuentes)

---

## 1. Cómo funciona (conceptos)

El sistema usa **[Velopack](https://velopack.io)**, que resuelve tres cosas de una vez:
generar el **instalador**, alojar las **versiones** y que la app se **autoactualice**.

El ciclo completo es:

```
   TÚ (al publicar)                          EL USUARIO (al usar la app)
   ─────────────────                          ──────────────────────────
   1. Subes <Version> en el .csproj
   2. Ejecutas release.ps1
        ├─ dotnet publish  → compila
        ├─ vpk pack        → crea Setup.exe + paquetes
        └─ vpk upload      → sube a GitHub Releases
                                    │
                                    ▼
                            GitHub Releases  ──────►  3. La app consulta si hay versión nueva
                       (MiniDisplay-releases)         4. Descarga solo lo que cambió (delta)
                                                      5. Se reinicia ya actualizada
```

### Dos repositorios, y por qué

| Repo | Contenido | Visibilidad |
|------|-----------|-------------|
| `MiniDisplay` | El **código fuente** | 🔒 Privado |
| [`MiniDisplay-releases`](https://github.com/Nels-ms/MiniDisplay-releases) | Solo los **binarios** (Setup.exe, paquetes) | 🌍 Público |

El repo de releases **debe ser público** porque la app consulta las actualizaciones **sin token**
(no se puede incrustar una credencial en un programa que se distribuye). Publicar los binarios
no expone el código: son cosas distintas.

### Qué se sube en cada release

| Archivo | Para qué sirve |
|---------|----------------|
| `MiniDisplay-win-Setup.exe` | **El instalador** — es lo que compartes con la gente |
| `MiniDisplay-<versión>-full.nupkg` | Paquete completo que descarga la app al actualizarse |
| `MiniDisplay-win-Portable.zip` | Versión portable (sin instalar) |
| `releases.win.json` | **Manifiesto**: lo que lee la app para saber si hay versión nueva |
| `RELEASES` | Igual, en formato antiguo (compatibilidad) |

---

## 2. Requisitos previos

**Instalar la herramienta de Velopack** (una sola vez en tu equipo):

```bash
dotnet tool install -g vpk
```

**Token de GitHub** con permiso de escritura sobre el repo de releases:

1. GitHub → *Settings → Developer settings → Personal access tokens*.
2. Crea un token con acceso al repositorio `MiniDisplay-releases` (permiso `repo` / *Contents: write*).
3. Guárdalo en un sitio seguro.

> 🔐 **No pegues el token directamente en la línea de comandos**: queda guardado en el historial
> de PowerShell. Mejor pásalo por variable de entorno:
> ```powershell
> $env:GITHUB_TOKEN = "ghp_..."
> .\release.ps1 -Version 1.0.3 -Token $env:GITHUB_TOKEN
> ```
> Si un token se expone alguna vez, **revócalo** en GitHub y genera otro.

---

## 3. Publicar la primera versión

```powershell
.\release.ps1 -Version 1.0.2 -Token $env:GITHUB_TOKEN
```

El script hace 4 pasos (los verás numerados en pantalla):

| Paso | Qué hace | Nota |
|------|----------|------|
| **1/4** Publicando | `dotnet publish` self-contained, en **carpeta** | No usa single-file: Velopack necesita los archivos sueltos para calcular deltas |
| **2/4** Descargando releases previos | Trae versiones anteriores para poder generar deltas | En la primera publicación avisará `No releases found` — **es normal** |
| **3/4** Empaquetando (`vpk pack`) | Genera `Setup.exe` y los paquetes en `Releases/` | Aquí se crean los accesos directos configurados |
| **4/4** Subiendo | Crea el release en GitHub y sube los assets | Solo si pasaste `-Token` |

Si omites `-Token`, el script se detiene tras el paso 3: tendrás los archivos en `Releases/`
pero no se subirán (útil para probar en local).

**Resultado:** `Releases\MiniDisplay-win-Setup.exe` — ese es el archivo que compartes.

---

## 4. Publicar una actualización

1. **Sube el número de versión** en `MiniDisplay/MiniDisplay.csproj`:
   ```xml
   <Version>1.0.3</Version>
   ```
   > Usa [versionado semántico](https://semver.org/lang/es/): `MAYOR.MENOR.PARCHE`.
   > La versión **debe ser mayor** que la anterior o la app no detectará la actualización.

2. **Ejecuta el script** con la misma versión:
   ```powershell
   .\release.ps1 -Version 1.0.3 -Token $env:GITHUB_TOKEN
   ```

3. Listo. Las apps instaladas detectarán la nueva versión.

> ⚠️ El número de `-Version` y el `<Version>` del `.csproj` deben **coincidir**.

---

## 5. Cómo se actualiza la app del usuario

Hay dos caminos, ambos activos:

- **Automático:** al arrancar, la app comprueba en segundo plano si hay versión nueva.
- **Manual:** clic derecho en el icono de la bandeja → **Buscar actualizaciones**.

En ambos casos, si hay una versión nueva **se pide confirmación** antes de descargar. Al aceptar,
se descarga y la app se reinicia ya actualizada.

> 💡 Los ajustes del usuario viven en `%AppData%\MiniDisplay\settings.json`, **fuera** de la
> carpeta de instalación: sobreviven a las actualizaciones.

> ℹ️ **En desarrollo (F5) no ocurre nada.** Todo el código está protegido por `IsInstalled`, que
> solo es `true` en la versión instalada con el `Setup.exe`. Si pulsas "Buscar actualizaciones"
> desde Visual Studio, te avisará de que solo funciona en la versión instalada.

---

## 6. Alternativa: ejecutable portable

Para pasar la app suelta a alguien sin instalar nada, existe un **`.exe` único autocontenido**
(no requiere tener .NET instalado):

**Desde Visual Studio:** clic derecho en el proyecto → **Publicar…** → perfil
**`win-x64-selfcontained`** → **Publicar**.

**Por CLI:**

```bash
dotnet publish MiniDisplay/MiniDisplay.csproj /p:PublishProfile=win-x64-selfcontained
```

Resultado: `MiniDisplay/bin/Publish/win-x64/MiniDisplay.exe` (un solo archivo).

> ⚠️ La versión portable **no se autoactualiza** (no está "instalada"). Para eso, usa el `Setup.exe`.

---

## 7. Piezas del sistema (dónde toca cada cosa)

| Archivo | Su papel en el despliegue |
|---------|---------------------------|
| `release.ps1` | Script que orquesta publicar → empaquetar → subir. La constante `$RepoUrl` define dónde se suben los binarios |
| `MiniDisplay/Program.cs` | Punto de entrada propio. Ejecuta `VelopackApp.Build().Run()` **antes** de arrancar WPF, para procesar los hooks de instalación/actualización sin abrir la interfaz |
| `MiniDisplay/Services/UpdateService.cs` | Lógica de actualización (comprobar → descargar → aplicar). Su constante `RepoUrl` debe apuntar al **mismo** repo que el script |
| `MiniDisplay/MiniDisplay.csproj` | `<Version>` (número de versión) y `<StartupObject>` (usa nuestro `Main()`) |
| `MiniDisplay/App.xaml` | El elemento *Buscar actualizaciones* del menú de la bandeja |

> Si algún día cambias de repositorio de releases, actualiza **los dos sitios**:
> `$RepoUrl` en `release.ps1` y `RepoUrl` en `UpdateService.cs`.

---

## 8. Problemas frecuentes

**«No releases found» en el paso 2/4**
Normal en la primera publicación: aún no hay versiones previas de las que calcular deltas.
El script continúa y genera un release base completo.

**La app no detecta la actualización**
- ¿El repo de releases es **público**? La app consulta sin token; si es privado, no ve nada.
- ¿La versión nueva es **mayor** que la instalada?
- ¿Estás probando en la versión **instalada** (no con F5)?
- ¿El release en GitHub está **publicado** y no quedó en borrador?

**SmartScreen dice «editor desconocido»**
Esperado: el ejecutable no está firmado digitalmente. Se abre con *Más información → Ejecutar de
todos modos*. Para eliminarlo haría falta un certificado de firma de código.

**Un antivirus corporativo (EDR) bloquea la app**
En equipos gestionados por una empresa, la política de seguridad manda. Firmar el código ayuda a
que el equipo de IT pueda autorizarla por editor, pero el paso necesario es **pedirles que la
incluyan en la lista de permitidos**.

**Errores `MSB3021` / `MSB3027` al compilar**
La app está abierta y bloquea el `.exe`. Ciérrala (bandeja → Salir) y vuelve a compilar.
No son errores de código.

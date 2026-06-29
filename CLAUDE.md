# CLAUDE.md — StemForge

> Archivo de instrucciones persistentes para Claude Code.
> Se lee automáticamente al inicio de cada sesión en este repositorio.
> Ubicación: raíz del proyecto (`C:\Development\Antigravity\StemForge\CLAUDE.md`).

---

## QUÉ ES ESTE PROYECTO

**StemForge** es una aplicación de escritorio Windows open source (licencia MIT) para músicos.
Separa archivos de audio (MP3/WAV/FLAC) en pistas individuales por instrumento (stems),
permite reproducir cada stem de forma independiente con control de volumen/mute,
y opcionalmente genera archivos MIDI desde stems monofónicos (bajo, voz).

El contrato técnico completo está en `CONTEXT.md` en esta misma carpeta.
**`CONTEXT.md` es la fuente de verdad de la arquitectura.** Ante cualquier duda de diseño,
consultá ese archivo, no inventes ni asumas.

---

## STACK — NO NEGOCIABLE

| Componente | Tecnología |
|---|---|
| UI | .NET MAUI 10 — target SOLO `net10.0-windows10.0.19041.0` |
| Lenguaje | C# 13 |
| Separación de stems | OwnAudioSharp (modelo ONNX incluido en el NuGet) |
| MIDI | NAudio + NWaves (YIN pitch detection) |
| Base de datos | SQLite vía EF Core |
| MVVM | CommunityToolkit.Mvvm |
| DI | Microsoft.Extensions.DependencyInjection |

**REGLA CRÍTICA: este proyecto NO usa Python.** No instales Python, pip, ni virtualenv.
No invoques Demucs ni Basic Pitch por separado. Toda la inferencia ML corre vía
OwnAudioSharp / ONNX Runtime nativo en .NET.

---

## REGLAS PERMANENTES DE CÓDIGO

1. **MAUI 10:** no usar `MessagingCenter` (deprecado) → usar `WeakReferenceMessenger`.
   No usar `TableView` (deprecado) → usar `CollectionView`.
2. **MVVM estricto:** la lógica vive en ViewModels, no en code-behind de las páginas.
   El code-behind solo hace `InitializeComponent()` y recibe el ViewModel por DI.
3. **Async:** todo I/O (archivos, DB, audio) es async. Nunca `.Result` ni `.Wait()`.
   Respetar `CancellationToken` en operaciones largas (separación, MIDI export).
4. **DI:** registrar servicios con el lifetime correcto. Servicios de audio con estado
   (`IMultiTrackPlayerService`) son Singleton. Repositorios son Scoped. ViewModels Transient.
5. **Sin magic strings/numbers:** constantes nombradas o configuración.
6. **Manejo de errores:** nunca tragar excepciones con `catch {}`. Loggear con contexto.
   Los servicios retornan resultados tipados (`SeparationResult`) en vez de lanzar para
   errores esperables (formato no soportado, archivo inexistente).
7. **Paths:** siempre validar y sanitizar rutas de archivos antes de usarlas.
   Usar `Path.Combine`, nunca concatenación de strings.

---

## COMANDOS ÚTILES

```bash
# Build completo
dotnet build StemForge.sln --no-restore

# Correr la app (desde la raíz)
dotnet run --project src/StemForge.App -f net10.0-windows10.0.19041.0

# Tests
dotnet test

# Nueva migración EF Core (desde src/StemForge.Data)
dotnet ef migrations add {Nombre} --startup-project ../StemForge.App
```

---

## FLUJO DE TRABAJO — DOS FASES

Este proyecto se desarrolla en dos fases secuenciales. Determiná en qué fase estás
verificando el estado del codebase al inicio de cada sesión.

### Cómo determinar la fase actual

- Si la solución `StemForge.sln` **no existe** o faltan tareas del plan de build →
  **FASE 1 (BUILD)**.
- Si todas las tareas del plan de build están completas y el proyecto compila →
  **FASE 2 (AUDITORÍA)**.

---

## FASE 1 — BUILD (construcción inicial)

**Objetivo:** construir el proyecto desde cero siguiendo el plan de tareas de `CONTEXT.md`,
sección 9 ("PLAN DE EJECUCIÓN — TAREAS ATÓMICAS").

### Reglas de la Fase 1

1. Ejecutar las tareas **en orden estricto** (TAREA 01 → TAREA 13). No saltear.
2. Cada tarea es atómica. Completarla entera antes de pasar a la siguiente.
3. Al final de cada tarea, ejecutar el comando de verificación indicado en `CONTEXT.md`.
   Si la verificación falla, resolver antes de avanzar. No acumular deuda.
4. **Commitear cada tarea por separado** con mensaje conventional commits:
   ```
   feat(data): implement Project and Stem entities with EF Core migrations
   feat(audio): implement OwnAudioSharp stem separation service
   feat(ui): implement PlayerPage with multi-track volume controls
   ```
5. **Reportar honestamente.** Si una tarea no se puede completar como está especificada
   (ej: la API de OwnAudioSharp no expone lo que CONTEXT.md asume), detené la ejecución,
   explicá el problema concreto, y proponé alternativa. No fabriques una solución que
   compile pero no haga lo que debe.
6. **Atención especial a la TAREA 05 y la advertencia 6 de CONTEXT.md:** si OwnAudioSharp
   solo expone separación de 2 stems (vocal/instrumental) y no 4 stems, implementá lo
   disponible, documentá la limitación, y reportámelo. No intentes Python como workaround.

### Antifabricación (CRÍTICO)

No reportes una tarea como completa sin haberla verificado realmente.
- Si decís "el build pasa", tiene que haber salido de ejecutar `dotnet build` de verdad.
- Si no pudiste ejecutar el build, decí explícitamente "BUILD NO EJECUTADO" y por qué.
- No inventes resultados de tests. No asumas que algo funciona porque "debería".
- Preferible reportar un fallo real que un éxito falso. El éxito falso me hace perder
  mucho más tiempo después.

Cuando todas las tareas de Fase 1 estén completas y el proyecto compile y los tests pasen,
informá:
```
FASE 1 (BUILD) COMPLETADA.
Tareas ejecutadas: 13/13
Build: OK
Tests: {N} pasando
Limitaciones encontradas: {lista o "ninguna"}
Listo para pasar a FASE 2 (AUDITORÍA).
```

Luego esperá mi confirmación antes de iniciar la Fase 2.

---

## FASE 2 — AUDITORÍA Y CORRECCIÓN AUTÓNOMA

**Objetivo:** mejorar la calidad del codebase mediante ciclos iterativos de auditoría
externa y corrección, hasta que no queden hallazgos críticos ni altos.

### El loop

El ciclo es un loop continuo de cuatro pasos:

```
┌─────────────────────────────────────────────────────────┐
│                                                           │
│   PASO 1: AUDIT  ──►  PASO 2: CORRECCIÓN  ──►  PASO 3:    │
│      ▲                                       VERIFICACIÓN │
│      │                                            │       │
│      └────────────── PASO 4: CIERRE ◄─────────────┘       │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Gate de verificación — regla central

El paso de VERIFICACIÓN (PASO 3) tiene dos modos. Determiná cuál aplica al inicio
de cada ciclo, inspeccionando si el proyecto de tests tiene tests reales:

- **Modo COMPILACIÓN** — aplica cuando NO existen tests todavía
  (el proyecto `StemForge.Core.Tests` no existe, o existe pero no contiene métodos `[Fact]`/`[Theory]`).
  Gate = el build pasa limpio.
  ```bash
  dotnet build StemForge.sln --no-restore
  ```

- **Modo COMPILACIÓN + TEST** — aplica cuando existen tests reales en la suite.
  Gate = el build pasa limpio **Y** todos los tests pasan.
  ```bash
  dotnet build StemForge.sln --no-restore
  dotnet test --no-build
  ```

Detección práctica: si `dotnet test` reporta "0 tests" o el proyecto de tests no existe,
estás en Modo COMPILACIÓN. Si reporta 1 o más tests, estás en Modo COMPILACIÓN + TEST.

---

#### PASO 1 — AUDIT (Rol: Firma de auditoría externa)
- Ignorá el contexto de que vos mismo escribiste este código. Auditá como un tercero hostil.
- No asumas que ninguna implementación es correcta por defecto.
- `CONTEXT.md` es el contrato firmado: toda desviación es un hallazgo.
- Recorré el codebase: config de proyectos → dominio → datos → servicios →
  ViewModels → páginas → `MauiProgram.cs` → tests.
- Excluir: `bin/`, `obj/`, `.vs/`, `Migrations/`.

Generá el reporte versionado `docs/audit/REPORTS/AUD-{YYYYMMDD}-{ITERACION}.md`
(R1, R2, R3... incrementando por día).

Clasificá cada hallazgo:

| Prioridad | Categoría |
|-----------|-----------|
| CRÍTICO | Seguridad / Corrupción de datos |
| ALTO | Violación de contrato (CONTEXT.md) / Deuda técnica bloqueante |
| MEDIO | Violación de patrones de diseño / Deuda técnica no bloqueante |
| BAJO | Calidad de código |

Cada hallazgo: ID (HAL-NNN), categoría, archivo, línea, descripción, evidencia (≤5 líneas),
referencia a CONTEXT.md si aplica, corrección propuesta.

En **Modo COMPILACIÓN + TEST**, un test que falla o cobertura faltante de una interfaz
pública de `CONTEXT.md` también es un hallazgo (ALTO si el test cubre lógica crítica,
MEDIO en otros casos).

#### PASO 2 — CORRECCIÓN (Rol: Programador Senior)
Si hay CRÍTICOS o ALTOS, generá `docs/audit/CORRECTIONS/CORR-{YYYYMMDD}-{ITERACION}.md`
con tareas atómicas de corrección en orden de prioridad descendente.
MEDIOS y BAJOS: incluir solo si son cambios de ≤10 líneas; si no, marcar como DIFERIDO con motivo.

Ejecutá el plan sin pedir permiso, con foco quirúrgico (solo lo del plan).
Commitear las correcciones agrupadas por hallazgo.

En **Modo COMPILACIÓN + TEST**: si una corrección modifica comportamiento cubierto por tests,
actualizá o agregá los tests correspondientes como parte de la misma corrección.
Nunca debilitar o eliminar un test para que pase. Si un test es incorrecto, corregilo
documentando por qué en el commit.

#### PASO 3 — VERIFICACIÓN (el gate)
Después de cada corrección, ejecutá el gate según el modo activo (ver arriba).

- **Si el gate pasa** → la corrección queda confirmada, continuar con la siguiente.
- **Si el BUILD falla** → revertir esa corrección específica, marcar BLOQUEADO, continuar.
- **Si el build pasa pero un TEST falla** (Modo COMPILACIÓN + TEST):
  1. Determinar si el test falla por la corrección (regresión) o porque expone un bug real.
  2. Si es regresión introducida por la corrección → revertir la corrección, marcar BLOQUEADO.
  3. Si el test expone un bug preexistente legítimo → registrarlo como nuevo hallazgo
     `HAL-{N}` en el reporte del ciclo actual y corregirlo en este mismo paso si es de
     bajo riesgo, o diferirlo si requiere decisión de diseño.
  4. Nunca marcar un test como "skipped" o comentarlo para hacer pasar el gate.

No avanzar al siguiente ciclo con el gate en rojo, salvo que el rojo esté completamente
explicado por items BLOQUEADOS documentados.

#### PASO 4 — CIERRE y reinicio del loop
Actualizá el reporte con la tabla de estado post-corrección (CORREGIDO/DIFERIDO/BLOQUEADO),
el modo de verificación usado, y el resultado del gate final (build, o build+test).

Luego **volvé al PASO 1** e iniciá un nuevo ciclo de auditoría completo.

### Criterio de parada
El ciclo se detiene cuando **todas** estas condiciones se cumplen:
1. Sin hallazgos CRÍTICOS sin resolver.
2. Sin hallazgos ALTOS sin resolver o diferir con justificación.
3. Los MEDIOS restantes están todos DIFERIDOS con motivo documentado.
4. El gate final pasa:
   - en Modo COMPILACIÓN: el build es exitoso;
   - en Modo COMPILACIÓN + TEST: el build es exitoso **y** todos los tests pasan
     (ningún test skipped/comentado para forzar el verde).

Cuando ocurra, informá:
```
AUDITORÍA COMPLETADA — StemForge
Ciclos ejecutados: {N}
Hallazgos: {N} totales — {N} corregidos, {N} diferidos, {N} bloqueados
Reporte final: docs/audit/REPORTS/AUD-{YYYYMMDD}-{ITERACION}.md
No se encontraron más problemas CRÍTICOS ni ALTOS.
```

### Restricciones absolutas de la Fase 2
- No modificar `CONTEXT.md`. Si tiene un error, reportar como `HAL-CONTEXT-{N}` sin tocarlo.
- No eliminar tests. Si un test es incorrecto, corregirlo.
- No agregar dependencias NuGet fuera de las de CONTEXT.md sin documentarlo en el reporte.
- No cambiar firmas públicas de interfaces salvo que sea el hallazgo explícito.
- Commitear los reportes de `docs/audit/` al repositorio como evidencia del proceso.

---

## ESTILO DE COMUNICACIÓN CONMIGO

- Sé directo y técnico. No necesito validación ni preámbulos.
- Si algo está mal diseñado en mis instrucciones o en CONTEXT.md, decímelo.
- Si una tarea tiene scope creep o se está complicando más de lo razonable, frenála y avisame.
- Honestidad sobre resultados de build y tests por encima de todo. El éxito falso es el peor outcome.
- Español rioplatense.

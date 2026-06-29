# CONTEXT.md — StemForge
## Contrato técnico para ejecución por agente (Gemini CLI)

> **Leer completo antes de ejecutar cualquier paso.**
> Cada tarea es atómica. Ejecutar en orden estricto. No inferir ni completar pasos no descriptos.
> Ante ambigüedad: detener y reportar. No asumir.

---

## 1. DESCRIPCIÓN DEL PROYECTO

**StemForge** es una aplicación de escritorio Windows open source para músicos.
Permite cargar un archivo de audio (MP3, WAV, FLAC), separarlo automáticamente en pistas
individuales por instrumento (stems), reproducir cada stem de forma independiente con control
de volumen y silencio, y exportar cada stem como archivo de audio. Opcionalmente genera
archivos MIDI desde stems melódicos (bajo, voz, melodía).

**Repositorio público:** GitHub — nombre: `stemforge` — licencia: MIT

---

## 2. STACK TÉCNICO — NO NEGOCIABLE

| Componente | Tecnología | Versión |
|---|---|---|
| Framework UI | .NET MAUI | 10.0 — target: `net10.0-windows10.0.19041.0` |
| Lenguaje | C# | 13 |
| Motor de audio / separación | OwnAudioSharp | NuGet — última estable |
| Reproducción multi-pista | OwnAudioSharp (sincronizado) | idem |
| Detección de tempo (BPM) | OwnAudioSharp ChordDetection / NWaves | según disponibilidad |
| MIDI output | NAudio | NuGet — última estable |
| Base de datos | SQLite vía EF Core | `Microsoft.EntityFrameworkCore.Sqlite` |
| DI / IoC | Microsoft.Extensions.DependencyInjection | incluido en MAUI |
| MVVM | CommunityToolkit.Mvvm | NuGet — última estable |
| Logging | Microsoft.Extensions.Logging + Serilog sink archivo | NuGet |

**MAUI target:** SOLO `net10.0-windows10.0.19041.0` en Fase 1.
Android/iOS se habilitan en Fase 2 (fuera de scope de este CONTEXT.md).
El archivo `.csproj` NO debe incluir targets de Android/iOS/Mac en esta fase.

---

## 3. ESTRUCTURA DE SOLUCIÓN

```
StemForge/
├── StemForge.sln
├── src/
│   ├── StemForge.App/               ← Proyecto MAUI (UI + shell)
│   ├── StemForge.Core/              ← Lógica de negocio, servicios, modelos de dominio
│   ├── StemForge.Data/              ← EF Core DbContext, Migrations, Repositorios
│   └── StemForge.Audio/             ← Wrapper de OwnAudioSharp y NAudio
├── tests/
│   └── StemForge.Core.Tests/        ← xUnit, sin dependencias de UI ni audio hardware
├── docs/
│   └── LIMITATIONS.md               ← Limitaciones honestas para usuarios
├── .gitignore
├── README.md
└── LICENSE                          ← MIT
```

---

## 4. MODELO DE DATOS (SQLite)

### 4.1 Entidades

```csharp
// StemForge.Data/Entities/

public class Project
{
    public int Id { get; set; }
    public string Name { get; set; }           // Nombre del proyecto
    public string OriginalFilePath { get; set; } // Ruta al archivo fuente
    public string OutputDirectory { get; set; }  // Directorio de stems generados
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public ProcessingStatus Status { get; set; }
    public int? DetectedBpm { get; set; }
    public ICollection<Stem> Stems { get; set; }
}

public class Stem
{
    public int Id { get; set; }
    public int ProjectId { get; set; }
    public Project Project { get; set; }
    public StemType Type { get; set; }         // Vocals, Drums, Bass, Other
    public string FilePath { get; set; }        // Ruta al archivo WAV generado
    public string? MidiFilePath { get; set; }   // Ruta al MIDI (si fue generado)
    public float VolumeLevel { get; set; }      // Último volumen configurado (0.0-1.0)
    public bool IsMuted { get; set; }
    public long FileSizeBytes { get; set; }
    public double DurationSeconds { get; set; }
}

public enum ProcessingStatus { Pending, Processing, Completed, Failed }
public enum StemType { Vocals, Drums, Bass, Other }
```

### 4.2 DbContext

```csharp
// StemForge.Data/AppDbContext.cs
public class AppDbContext : DbContext
{
    public DbSet<Project> Projects { get; set; }
    public DbSet<Stem> Stems { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlite($"Data Source={GetDbPath()}");

    private static string GetDbPath()
    {
        var folder = Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData);
        return Path.Combine(folder, "StemForge", "stemforge.db");
    }
}
```

### 4.3 Migrations

Usar `dotnet ef migrations add InitialCreate` desde `StemForge.Data/`.
La base de datos se crea automáticamente al iniciar la app vía `db.Database.EnsureCreated()`.

---

## 5. CAPA DE AUDIO (StemForge.Audio)

### 5.1 Interfaz principal

```csharp
public interface IAudioSeparationService
{
    Task<SeparationResult> SeparateAsync(
        string inputFilePath,
        string outputDirectory,
        IProgress<SeparationProgress> progress,
        CancellationToken cancellationToken);
}

public record SeparationResult(
    bool Success,
    string? ErrorMessage,
    IReadOnlyList<StemFile> Stems);

public record StemFile(StemType Type, string FilePath, double DurationSeconds);

public record SeparationProgress(int PercentComplete, string StatusMessage);
```

### 5.2 Implementación con OwnAudioSharp

```csharp
// StemForge.Audio/Services/OwnAudioSeparationService.cs
// Usar OwnAudioSharp VocalRemover / HTDemucs para separación 4-stem.
// El modelo ONNX viene incluido en el NuGet — no requiere descarga manual.
// Separación produce: Vocals.wav, Drums.wav, Bass.wav, Other.wav
// Todos los stems en formato WAV 44100Hz stereo — normalizar volumen post-separación.
```

### 5.3 Interfaz de reproducción

```csharp
public interface IMultiTrackPlayerService
{
    Task LoadProjectAsync(IEnumerable<StemFile> stems);
    void Play();
    void Pause();
    void Stop();
    void SetStemVolume(StemType stem, float volume);  // 0.0 a 1.0
    void SetStemMuted(StemType stem, bool muted);
    double CurrentPositionSeconds { get; }
    double TotalDurationSeconds { get; }
    bool IsPlaying { get; }
    event EventHandler<double> PositionChanged;  // cada 100ms
    event EventHandler PlaybackCompleted;
}
```

Usar OwnAudioSharp Multi-Track Synchronized Playback.
Los 4 stems reproducen en sincronía usando el clock central de OwnAudioSharp.

### 5.4 MIDI Export (monofónico — solo Bass y Vocals)

```csharp
public interface IMidiExportService
{
    // Solo aplicar a stems Bass y Vocals — NO a Drums ni Other.
    // Usar NAudio para escritura del archivo MIDI.
    // Detección de pitch: NWaves YIN algorithm sobre el stem WAV.
    // Resultado: archivo .mid con tempo del proyecto si fue detectado.
    Task<string?> ExportToMidiAsync(
        string stemWavPath,
        StemType stemType,
        string outputDirectory,
        int? bpmHint,
        CancellationToken cancellationToken);
}
```

**IMPORTANTE:** El servicio MIDI debe documentar en su XML summary que
los resultados son mejores en líneas monofónicas simples y que material
polifónico complejo puede requerir edición manual en un DAW.

---

## 6. VIEWMODELS (CommunityToolkit.Mvvm)

### 6.1 Lista de ViewModels requeridos

```
MainViewModel          ← Shell / navegación
ProjectListViewModel   ← Lista de proyectos recientes
NewProjectViewModel    ← Importar archivo + iniciar separación
ProcessingViewModel    ← Progreso de separación con cancelación
PlayerViewModel        ← Reproductor multi-stem con controles por pista
ExportViewModel        ← Exportar stems y/o MIDI
```

### 6.2 PlayerViewModel — propiedades clave

```csharp
[ObservableProperty] private double currentPosition;
[ObservableProperty] private double totalDuration;
[ObservableProperty] private bool isPlaying;

// Por stem:
[ObservableProperty] private float vocalsVolume = 1.0f;
[ObservableProperty] private bool vocalsMuted;
[ObservableProperty] private float drumsVolume = 1.0f;
[ObservableProperty] private bool drumsMuted;
[ObservableProperty] private float bassVolume = 1.0f;
[ObservableProperty] private bool bassMuted;
[ObservableProperty] private float otherVolume = 1.0f;
[ObservableProperty] private bool otherMuted;

[RelayCommand] private void Play();
[RelayCommand] private void Pause();
[RelayCommand] private void Stop();
[RelayCommand] private async Task ExportMidiAsync(StemType stemType);
```

---

## 7. UI — PÁGINAS MAUI

### 7.1 Páginas requeridas

| Página | Descripción |
|---|---|
| `MainPage` | Shell con navegación lateral (FlyoutPage o Shell) |
| `ProjectListPage` | Lista de proyectos con fecha, estado, nombre del archivo |
| `NewProjectPage` | FilePicker para MP3/WAV/FLAC + botón "Separar" |
| `ProcessingPage` | ProgressBar + texto de estado + botón Cancelar |
| `PlayerPage` | Reproductor principal — ver sección 7.2 |
| `ExportPage` | Lista de stems con botones de exportar individual o todos |

### 7.2 PlayerPage — layout requerido

```
┌─────────────────────────────────────────────┐
│  [Nombre del proyecto]          [BPM: 124]  │
├─────────────────────────────────────────────┤
│  ◀◀  ▶ / ❚❚   ■■   ──────●────────  4:32  │  ← controles + seekbar
├─────────────────────────────────────────────┤
│  🎤 Vocals   [🔇]  ████████░░  0.8         │
│  🥁 Drums    [🔇]  ██████████  1.0         │
│  🎸 Bass     [🔇]  █████░░░░░  0.5         │
│  🎹 Other    [🔇]  ████████░░  0.8         │
├─────────────────────────────────────────────┤
│  [Exportar MIDI Bass]  [Exportar MIDI Voz]  │
│  [Exportar todos los stems]                 │
└─────────────────────────────────────────────┘
```

Cada fila de stem tiene: ícono, label, botón mute, Slider de volumen (0.0-1.0), valor numérico.
El seekbar permite saltar a posición en la pista.
Los cambios de volumen/mute persisten en SQLite al salir del proyecto.

---

## 8. CONFIGURACIÓN DE LA APP

```csharp
// MauiProgram.cs
builder.Services
    // Data
    .AddDbContext<AppDbContext>()
    .AddScoped<IProjectRepository, ProjectRepository>()
    .AddScoped<IStemRepository, StemRepository>()

    // Audio
    .AddSingleton<IAudioSeparationService, OwnAudioSeparationService>()
    .AddSingleton<IMultiTrackPlayerService, MultiTrackPlayerService>()
    .AddTransient<IMidiExportService, NWavesMidiExportService>()

    // ViewModels
    .AddTransient<ProjectListViewModel>()
    .AddTransient<NewProjectViewModel>()
    .AddTransient<ProcessingViewModel>()
    .AddTransient<PlayerViewModel>()
    .AddTransient<ExportViewModel>()

    // Pages
    .AddTransient<ProjectListPage>()
    .AddTransient<NewProjectPage>()
    .AddTransient<ProcessingPage>()
    .AddTransient<PlayerPage>()
    .AddTransient<ExportPage>();
```

---

## 9. PLAN DE EJECUCIÓN — TAREAS ATÓMICAS PARA EL AGENTE

### TAREA 01 — Crear solución y estructura de proyectos

```bash
dotnet new sln -n StemForge
dotnet new maui -n StemForge.App -f net10.0-windows10.0.19041.0 --no-restore
dotnet new classlib -n StemForge.Core --framework net10.0
dotnet new classlib -n StemForge.Data --framework net10.0
dotnet new classlib -n StemForge.Audio --framework net10.0
dotnet new xunit -n StemForge.Core.Tests --framework net10.0
dotnet sln add src/StemForge.App src/StemForge.Core src/StemForge.Data src/StemForge.Audio tests/StemForge.Core.Tests
```

Editar `StemForge.App.csproj`:
- Remover targets Android/iOS/Mac si los generó el template
- Dejar solo: `<TargetFrameworks>net10.0-windows10.0.19041.0</TargetFrameworks>`

Crear estructura de carpetas vacías con `.gitkeep` donde corresponda.
Crear `.gitignore` estándar para .NET.
Crear `LICENSE` con texto MIT completo.

**Verificación:** `dotnet build StemForge.sln` debe compilar sin errores.

---

### TAREA 02 — Agregar dependencias NuGet

En `StemForge.App`:
```
dotnet add package OwnAudioSharp
dotnet add package CommunityToolkit.Mvvm
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Serilog.Extensions.Logging.File
```

En `StemForge.Audio`:
```
dotnet add package OwnAudioSharp
dotnet add package NAudio
dotnet add package NWaves
```

En `StemForge.Data`:
```
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
```

En `StemForge.Core.Tests`:
```
dotnet add package xunit
dotnet add package FluentAssertions
dotnet add package Moq
```

Agregar referencias entre proyectos:
- `StemForge.App` → Core, Data, Audio
- `StemForge.Audio` → Core
- `StemForge.Data` → Core

**Verificación:** `dotnet restore && dotnet build` sin errores.

---

### TAREA 03 — Implementar entidades y DbContext

Crear en `StemForge.Data/`:
- `Entities/Project.cs` — según sección 4.1
- `Entities/Stem.cs` — según sección 4.1
- `Enums/ProcessingStatus.cs`
- `Enums/StemType.cs`
- `AppDbContext.cs` — según sección 4.2
- `Repositories/IProjectRepository.cs` + `ProjectRepository.cs`
- `Repositories/IStemRepository.cs` + `StemRepository.cs`

Los repositorios implementan CRUD básico usando EF Core async.
No usar patrones Unit of Work — el DbContext es suficiente para este scope.

Correr migration inicial:
```bash
cd src/StemForge.Data
dotnet ef migrations add InitialCreate --startup-project ../StemForge.App
```

**Verificación:** migration generada en `Migrations/` folder.

---

### TAREA 04 — Implementar interfaces de audio en StemForge.Audio

Crear:
- `Interfaces/IAudioSeparationService.cs` — según sección 5.1
- `Interfaces/IMultiTrackPlayerService.cs` — según sección 5.3
- `Interfaces/IMidiExportService.cs` — según sección 5.4
- `Models/SeparationResult.cs`
- `Models/SeparationProgress.cs`
- `Models/StemFile.cs`

Crear implementaciones STUB (no funcionales aún, retornan datos de prueba hardcodeados):
- `Services/OwnAudioSeparationService.cs` — retorna 4 StemFile con paths ficticios
- `Services/MultiTrackPlayerService.cs` — implementa interfaz con métodos vacíos
- `Services/NWavesMidiExportService.cs` — retorna null

**Verificación:** compila sin errores. Los stubs permiten desarrollar UI sin audio real.

---

### TAREA 05 — Implementar separación real con OwnAudioSharp

Completar `OwnAudioSeparationService.cs`:

1. Validar que el archivo de entrada existe y es MP3/WAV/FLAC
2. Crear directorio de salida si no existe (subdirectorio con timestamp + nombre canción)
3. Invocar HTDemucs via OwnAudioSharp API:
   - Usar `VocalRemover` o la API de separación de 4 stems de OwnAudioSharp
   - Separar en: Vocals.wav, Drums.wav, Bass.wav, Other.wav
   - Reportar progreso via `IProgress<SeparationProgress>`
   - Respetar CancellationToken
4. Post-proceso: normalizar volumen de cada stem (peak normalization a -3dBFS)
5. Calcular duración de cada stem
6. Retornar `SeparationResult` con lista de `StemFile`

Manejo de errores:
- Archivo no soportado → `SeparationResult(false, "Formato no soportado: {ext}")`
- OwnAudioSharp lanza excepción → loggear y retornar `SeparationResult(false, ex.Message)`
- Cancelación → limpiar archivos parciales del directorio de salida

**Verificación:** test manual con un MP3 de 3-4 minutos. Confirmar 4 archivos WAV generados.

---

### TAREA 06 — Implementar reproductor multi-pista

Completar `MultiTrackPlayerService.cs`:

1. `LoadProjectAsync`: cargar los 4 stems en OwnAudioSharp multi-track engine
   - Usar clock central sincronizado de OwnAudioSharp
   - Inicializar volúmenes desde los valores guardados en cada `Stem`
2. `Play` / `Pause` / `Stop`: delegar al engine
3. `SetStemVolume(StemType, float)`: ajustar volumen del track correspondiente en tiempo real
4. `SetStemMuted(StemType, bool)`: silenciar/activar track sin perder nivel de volumen
5. Timer interno cada 100ms para disparar `PositionChanged`
6. Liberar recursos en `IAsyncDisposable`

**Verificación:** cargar un proyecto procesado y reproducir con controles de volumen funcionales.

---

### TAREA 07 — Implementar ViewModels

Crear en `StemForge.App/ViewModels/`:

**ProjectListViewModel:**
- `ObservableCollection<ProjectSummary> Projects`
- `LoadProjectsCommand` — carga desde repositorio al navegar
- `OpenProjectCommand(int projectId)` — navega a PlayerPage
- `NewProjectCommand` — navega a NewProjectPage
- `DeleteProjectCommand(int projectId)` — confirma y elimina (archivos + DB)

**NewProjectViewModel:**
- `string? SelectedFilePath`
- `PickFileCommand` — usa `FilePicker.PickAsync` con filtros audio
- `StartSeparationCommand` — crea Project en DB, navega a ProcessingPage

**ProcessingViewModel:**
- `int ProgressPercent`
- `string StatusMessage`
- `CancelCommand` — cancela via CancellationToken
- Al completar: navega a PlayerPage con el ProjectId

**PlayerViewModel:**
- Todas las propiedades de sección 6.2
- `LoadProjectCommand(int projectId)`
- Persiste volumen/mute en DB al cambiar (debounce 500ms para no saturar writes)

**ExportViewModel:**
- `ExportStemCommand(StemType)` — copia WAV a destino elegido por usuario
- `ExportAllStemsCommand` — exporta los 4
- `ExportMidiCommand(StemType)` — llama IMidiExportService, disponible solo para Bass/Vocals

---

### TAREA 08 — Implementar páginas MAUI

Crear en `StemForge.App/Pages/`:

**ProjectListPage.xaml:**
- CollectionView con lista de proyectos
- Cada item: nombre, fecha, estado (ícono), botón abrir y botón eliminar
- Botón flotante "+" para nuevo proyecto

**NewProjectPage.xaml:**
- Label con path seleccionado
- Botón "Seleccionar archivo"
- Información: formatos soportados (MP3, WAV, FLAC)
- Botón "Separar" (habilitado solo cuando hay archivo seleccionado)

**ProcessingPage.xaml:**
- ProgressBar circular o lineal
- Label de porcentaje y mensaje de estado
- Botón "Cancelar"
- NOTA para usuario: "La separación puede tardar varios minutos dependiendo del hardware"

**PlayerPage.xaml:**
- Layout según sección 7.2
- Slider de posición (seekbar) bindeado a CurrentPosition
- 4 filas de stems con Slider de volumen y toggle de mute
- Sección de exportación al pie

**ExportPage.xaml:**
- Lista de stems disponibles
- Para cada stem: nombre, tamaño, botón "Exportar WAV"
- Bass y Vocals: botón adicional "Exportar MIDI"
- Botón "Exportar Todo"

**Binding:** todas las páginas reciben su ViewModel via DI (constructor injection + Shell routing).

---

### TAREA 09 — Implementar MIDI export básico

Completar `NWavesMidiExportService.cs`:

1. Leer el stem WAV con NAudio `AudioFileReader`
2. Detectar pitch usando NWaves `YinPitchDetector` frame a frame (frame: 1024 samples, hop: 512)
3. Cuantizar frecuencias a notas MIDI (A4 = 440Hz → MIDI 69)
4. Filtrar frames sin pitch detectado (silencio) usando threshold de energía
5. Agrupar notas consecutivas iguales en eventos Note On/Off
6. Escribir archivo `.mid` con NAudio `MidiEventCollection`:
   - Tempo: del BPM detectado del proyecto (si existe) o 120 BPM por defecto
   - Canal 0 para todos los eventos
   - Velocity 80 para todas las notas (uniforme — no detectar dinámica en esta versión)
7. Retornar ruta del archivo .mid generado

**Limitaciones a documentar en XML summary del método:**
- Funciona mejor con líneas monofónicas simples
- Bajo con distorsión fuerte puede producir errores de octava
- No detecta dinámica (velocity uniforme en v1.0)

---

### TAREA 10 — Detección de BPM

Crear `StemForge.Audio/Services/BpmDetectionService.cs`:

```csharp
public interface IBpmDetectionService
{
    Task<int?> DetectBpmAsync(string audioFilePath, CancellationToken ct);
}
```

Implementar usando NWaves:
- Leer primeros 60 segundos del audio original (no del stem)
- Aplicar onset detection function
- Usar autocorrelación para estimar BPM en rango 60-200
- Retornar null si la confianza es baja (no inventar BPM)

Integrar en el flujo de separación: detectar BPM del archivo original antes de separar,
guardar en `Project.DetectedBpm`, mostrar en PlayerPage.

---

### TAREA 11 — Crear README.md y LIMITATIONS.md

**README.md debe incluir:**
- Descripción del proyecto en español e inglés
- Screenshot placeholder (agregar nota "screenshot pendiente")
- Requisitos: Windows 10 1809+, .NET 10 runtime, 8GB RAM recomendado (el modelo ONNX pesa 166MB)
- Instrucciones de instalación (desde release / desde código)
- Instrucciones de uso básico (3 pasos: importar, esperar, reproducir)
- Stack técnico (tabla)
- Licencia MIT
- Créditos: OwnAudioSharp (ModernMube), HTDemucs (Meta AI Research), NWaves, NAudio

**docs/LIMITATIONS.md debe incluir:**
- Separación de stems: resultados mejores en mezclas con buena separación de frecuencias
- Stems polifónicos (guitarra de acordes, piano): la separación puede tener artefactos
- MIDI: solo recomendado para bajo y melodía principal monofónica
- MIDI de acordes: NO soportado en v1.0 — el resultado no será utilizable
- Mezclas densas (metal, música electrónica muy procesada): resultados variables
- Velocidad de procesamiento: sin GPU puede tardar 3-10 minutos para una canción de 4 minutos
- GPU: si el sistema tiene CUDA disponible, OwnAudioSharp/ONNX Runtime lo usará automáticamente

---

### TAREA 12 — Tests unitarios

Crear en `StemForge.Core.Tests/`:

- `ProjectRepositoryTests.cs` — CRUD básico con SQLite in-memory
- `StemTypeTests.cs` — validar enums y conversiones
- `BpmDetectionServiceTests.cs` — mock del audio reader, validar rango de salida
- `MidiNoteConversionTests.cs` — validar frecuencia → número MIDI (A4=440Hz→69, etc.)

Mínimo 15 tests. Todos deben pasar con `dotnet test`.

---

### TAREA 13 — Inicialización y primer run

Completar `MauiProgram.cs`:
- Registrar todos los servicios según sección 8
- Configurar Serilog a archivo en `%LocalAppData%\StemForge\logs\stemforge-.log` (rolling daily)
- Al iniciar: ejecutar `db.Database.EnsureCreated()` para crear DB si no existe
- Configurar Shell routing para todas las páginas

Verificar que al ejecutar la app:
1. La base de datos se crea correctamente
2. Se puede navegar a todas las páginas
3. Se puede seleccionar un archivo de audio
4. El proceso de separación arranca y muestra progreso
5. Al completar, el reproductor carga los stems y reproduce

---

## 10. DECISIONES DE DISEÑO — REGISTRAR EN COMMITS

Cada tarea debe commitearse por separado con mensaje descriptivo:
```
feat(data): implement Project and Stem entities with EF Core migrations
feat(audio): implement OwnAudioSharp stem separation service
feat(ui): implement PlayerPage with multi-track volume controls
fix(midi): handle silence frames in YIN pitch detection
```

---

## 11. LO QUE ESTE CONTEXT NO INCLUYE (FASE 2)

Los siguientes items están fuera de scope de este documento:
- Target Android/iOS (`OwnAudioSharp.Mobile`)
- Carga directa desde URL/YouTube
- Slow-down/pitch-shift de stems para ensayo
- Exportar a formato STEMS (Native Instruments)
- Detección de acordes
- Waveform visualizer por stem
- Dark/Light theme switcher

---

## 12. ADVERTENCIAS PARA EL AGENTE

1. **No instalar Python.** Ninguna tarea requiere Python, pip, ni virtualenv.
2. **No modificar targets de plataforma** sin instrucción explícita. Solo Windows.
3. **No usar `MessagingCenter`** — está deprecated en MAUI 10. Usar `WeakReferenceMessenger`.
4. **No usar `TableView`** — deprecado en MAUI 10. Usar `CollectionView`.
5. **OwnAudioSharp incluye el modelo ONNX en el NuGet** — no buscar ni descargar modelos adicionales.
6. Si OwnAudioSharp no expone una API de 4-stem separación directa (solo 2-stem vocal/instrumental),
   documentar el issue, implementar la separación de 2-stem disponible, y crear un issue en GitHub
   del proyecto para rastrear la limitación. NO intentar invocar Python ni Demucs por separado.
7. El campo `VolumeLevel` en `Stem` se persiste solo al salir del proyecto (no en tiempo real durante playback).
   El debounce de 500ms en el ViewModel es para la UX, no implica writes continuos.

# Komikku Architecture Overview

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Presentation Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Activities │  │  Compose UI  │  │  ViewModels  │         │
│  │              │  │              │  │              │         │
│  │ ReaderActivity│  │ ReaderScreen │  │ReaderViewModel│         │
│  │ MainActivity │  │ LibraryScreen│  │LibraryScreen │         │
│  │              │  │ MangaScreen │  │Model         │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                         Domain Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Use Cases   │  │ Repositories │  │   Models     │          │
│  │              │  │              │  │              │          │
│  │GetMangaById  │  │MangaRepository│  │    Manga     │          │
│  │GetChapters   │  │ChapterRepo   │  │   Chapter    │          │
│  │UpdateProgress│  │CategoryRepo  │  │  Category    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                          Data Layer                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Database    │  │   Network    │  │   Cache      │          │
│  │              │  │              │  │              │          │
│  │ SQLDelight   │  │   OkHttp     │  │ DiskLRUCache │          │
│  │   SQLite     │  │   Coil       │  │ ChapterCache │          │
│  │              │  │              │  │ CoverCache   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                       Source API Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Source     │  │ HttpSource   │  │ LocalSource  │          │
│  │  Interface   │  │              │  │              │          │
│  │              │  │ Online       │  │ File System  │          │
│  │ Extensions   │  │ Sources      │  │ Archives     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└──────────────────────────────────────────────────────────────────┘
```

## Reader Component Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         ReaderActivity                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    ReaderViewModel                        │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │  │
│  │  │   State      │  │   Events     │  │   Actions    │   │  │
│  │  │              │  │              │  │              │   │  │
│  │  │ - manga      │  │ - LoadChapter│  │ - setPage    │   │  │
│  │  │ - chapters   │  │ - NextChapter│  │ - toggleMenu │   │  │
│  │  │ - viewer     │  │ - PrevChapter│  │ - showDialog │   │  │
│  │  │ - currentPage│  │ - ChangeMode │  │              │   │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                      │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │                    ChapterLoader                         │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │              PageLoader Factory                    │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │  │  │
│  │  │  │   Http   │  │ Download │  │  Archive │       │  │  │
│  │  │  │PageLoader│  │PageLoader│  │PageLoader│       │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘       │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                      │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │                      Viewer                               │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │   Pager      │  │   Webtoon    │  │   Vertical   │  │  │
│  │  │  Viewers     │  │   Viewer     │  │   Pager      │  │  │
│  │  │              │  │              │  │              │  │  │
│  │  │ L2R/R2L      │  │ Continuous   │  │ Page-by-page │  │  │
│  │  │ Pager        │  │ Scrolling    │  │ Vertical     │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow Diagrams

### Reading Flow

```
User Action: Open Reader
    │
    ▼
ReaderActivity.onCreate()
    │
    ▼
ReaderViewModel.init(mangaId, chapterId)
    │
    ├─→ GetMangaById UseCase
    │       │
    │       ▼
    │   MangaRepository
    │       │
    │       ▼
    │   Database Query
    │       │
    │       ▼
    │   Return Manga
    │
    └─→ GetChapterByUrlAndMangaId UseCase
            │
            ▼
        ChapterRepository
            │
            ▼
        Database Query
            │
            ▼
        Return Chapter
            │
            ▼
    ChapterLoader.loadChapter()
            │
            ▼
    Determine PageLoader Type
            │
            ├─→ HttpPageLoader (Online)
            │       │
            │       ▼
            │   Source.getPageList()
            │       │
            │       ▼
            │   Network Request
            │       │
            │       ▼
            │   Parse Response
            │       │
            │       ▼
            │   Return Pages
            │
            ├─→ DownloadPageLoader (Downloaded)
            │       │
            │       ▼
            │   Read from Storage
            │       │
            │       ▼
            │   Return Pages
            │
            └─→ ArchivePageLoader (Archive)
                    │
                    ▼
                Extract from Archive
                    │
                    ▼
                Return Pages
                    │
                    ▼
    Update ReaderChapter State
            │
            ▼
    Viewer.setChapters()
            │
            ▼
    Display Pages
            │
            ▼
    User Reads
            │
            ▼
    Update Progress
            │
            ▼
    Save to Database
```

### Source Integration Flow

```
Extension APK
    │
    ▼
ExtensionLoader.load()
    │
    ▼
Scan for Source Classes
    │
    ▼
Instantiate Sources
    │
    ▼
Register with SourceManager
    │
    ▼
Available in App
    │
    ├─→ Browse Sources
    │       │
    │       ▼
    │   CatalogueSource.getPopularManga()
    │       │
    │       ▼
    │   Network Request
    │       │
    │       ▼
    │   Parse HTML/JSON
    │       │
    │       ▼
    │   Return Manga List
    │
    ├─→ Search
    │       │
    │       ▼
    │   CatalogueSource.getSearchManga()
    │       │
    │       ▼
    │   Network Request with Query
    │       │
    │       ▼
    │   Parse Results
    │       │
    │       ▼
    │   Return Manga List
    │
    └─→ Read Chapter
            │
            ▼
        Source.getPageList()
            │
            ▼
        Network Request
            │
            ▼
        Parse Page URLs
            │
            ▼
        Return Page List
            │
            ▼
        Load Pages in Reader
```

## Module Dependencies

```
app
├── depends on ──→ domain
├── depends on ──→ data
├── depends on ──→ source-api
├── depends on ──→ presentation-core
├── depends on ──→ core-common
├── depends on ──→ core-archive
└── depends on ──→ core-metadata

domain
├── depends on ──→ (none - pure Kotlin)
└── defines interfaces for ──→ data

data
├── depends on ──→ domain
├── depends on ──→ source-api
└── implements ──→ domain interfaces

source-api
└── depends on ──→ (none - pure Kotlin)

presentation-core
├── depends on ──→ domain
└── depends on ──→ core-common
```

## Key Design Patterns

### 1. Repository Pattern

**Domain Layer** defines repository interfaces:
```kotlin
interface MangaRepository {
    suspend fun getMangaById(id: Long): Manga?
    suspend fun insertManga(manga: Manga)
}
```

**Data Layer** implements repositories:
```kotlin
class MangaRepositoryImpl : MangaRepository {
    override suspend fun getMangaById(id: Long): Manga? {
        // Database query
    }
}
```

### 2. Use Case Pattern

Business logic encapsulated in use cases:
```kotlin
class GetMangaById(
    private val repository: MangaRepository
) {
    suspend fun await(id: Long): Manga? {
        return repository.getMangaById(id)
    }
}
```

### 3. Strategy Pattern

Different page loaders for different sources:
```kotlin
when {
    isDownloaded -> DownloadPageLoader(...)
    source is HttpSource -> HttpPageLoader(...)
    source is LocalSource -> DirectoryPageLoader(...)
}
```

### 4. Factory Pattern

Viewer creation based on reading mode:
```kotlin
ReadingMode.toViewer(preference, activity, seedColor)
    .let { viewer ->
        when (mode) {
            LEFT_TO_RIGHT -> L2RPagerViewer(...)
            WEBTOON -> WebtoonViewer(...)
            // ...
        }
    }
```

### 5. Observer Pattern

State management with StateFlow:
```kotlin
val stateFlow = MutableStateFlow<State>(State.Initial)
stateFlow.collect { state ->
    // React to state changes
}
```

## Component Responsibilities

### ReaderActivity
- **Responsibility**: Activity lifecycle, UI coordination, user input handling
- **Does NOT**: Business logic, data loading, state management

### ReaderViewModel
- **Responsibility**: State management, business logic coordination, use case orchestration
- **Does NOT**: Direct UI manipulation, database access, network requests

### ChapterLoader
- **Responsibility**: Chapter loading coordination, page loader selection
- **Does NOT**: Actual page loading, UI updates

### PageLoader
- **Responsibility**: Page list retrieval, page image loading
- **Does NOT**: UI rendering, state management

### Viewer
- **Responsibility**: Page display, user interaction handling, navigation
- **Does NOT**: Data loading, business logic

### Source
- **Responsibility**: Data retrieval from external sources
- **Does NOT**: UI, state management, caching

## Data Models

### Domain Models (Pure Kotlin)
- `Manga` - Business entity
- `Chapter` - Business entity
- `Category` - Business entity
- `History` - Business entity

### Database Models (SQLDelight)
- `manga` table
- `chapters` table
- `categories` table
- `history` table

### Source Models (Source API)
- `SManga` - Source manga representation
- `SChapter` - Source chapter representation
- `Page` - Page representation

### Reader Models (Reader)
- `ReaderChapter` - Chapter in reader context
- `ReaderPage` - Page in reader context
- `ViewerChapters` - Chapter navigation context

## State Management

### ReaderViewModel State

```kotlin
data class State(
    // Data
    val manga: Manga?,
    val viewerChapters: ViewerChapters?,
    
    // UI State
    val currentPage: Int,
    val viewer: Viewer?,
    val menuVisible: Boolean,
    val dialog: Dialog?,
    
    // Configuration
    val brightnessOverlayValue: Int,
    val doublePages: Boolean,
    val autoScroll: Boolean
)
```

### State Updates

1. **User Action** → ViewModel receives event
2. **ViewModel** → Executes use case or updates state
3. **State Change** → StateFlow emits new state
4. **UI** → Collects state and recomposes

## Error Handling

### Reader Errors

```kotlin
sealed interface State {
    object Wait : State
    object Loading : State
    data class Error(val error: Throwable) : State
    data class Loaded(val pages: List<ReaderPage>) : State
}
```

### Source Errors

- Network errors → Retry mechanism
- Parse errors → Fallback parsing
- Missing data → Default values

## Performance Considerations

### 1. Lazy Loading
- Chapters loaded on demand
- Pages loaded as needed
- Images loaded progressively

### 2. Caching
- Chapter page lists cached
- Manga covers cached
- Network responses cached

### 3. Preloading
- Adjacent chapters preloaded
- Next pages preloaded
- Images preloaded

### 4. Memory Management
- Page loaders recycled
- Images released when not visible
- Viewers destroyed when not active

## Testing Strategy

### Unit Tests
- Domain layer (use cases, models)
- Data layer (repositories, mappers)
- Source API (parsers)

### Integration Tests
- Repository implementations
- Source integrations
- Page loaders

### UI Tests
- Reader interactions
- Navigation flows
- Settings changes

## Extension Points

### Custom Viewer
Implement `Viewer` interface:
```kotlin
class MyViewer : Viewer {
    // Implement interface methods
}
```

### Custom Page Loader
Extend `PageLoader`:
```kotlin
class MyPageLoader : PageLoader() {
    // Implement abstract methods
}
```

### Custom Source
Implement `Source` or extend `HttpSource`:
```kotlin
class MySource : HttpSource() {
    // Implement required methods
}
```

## Security Considerations

### Network Security
- HTTPS enforcement
- Certificate pinning (optional)
- Secure cookie storage

### Data Security
- Encrypted database (SQLCipher)
- Secure preferences
- Biometric authentication for categories

### Extension Security
- Extension validation
- Sandboxed execution
- Permission restrictions

---

This architecture provides a solid foundation for a comic reader application with clear separation of concerns, making it maintainable, testable, and extensible.



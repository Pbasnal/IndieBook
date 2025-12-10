# Komikku Codebase Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Module Structure](#module-structure)
4. [Comic Reader Implementation](#comic-reader-implementation)
5. [Source API](#source-api)
6. [Data Flow](#data-flow)
7. [Key Components](#key-components)
8. [Integration Guide](#integration-guide)
9. [Reusable Components](#reusable-components)

---

## Overview

Komikku is a free and open-source manga/comic reader application for Android, based on TachiyomiSY and Mihon/Tachiyomi. It provides a comprehensive solution for reading comics from various online sources and local storage.

### Key Features

- **Multiple Reading Modes**: Left-to-right, right-to-left, vertical, webtoon, and continuous vertical
- **Multiple Sources**: Support for online sources via extensions and local file reading
- **Library Management**: Categories, favorites, reading history, and progress tracking
- **Download Support**: Download chapters for offline reading
- **Tracker Integration**: Support for MyAnimeList, AniList, Kitsu, MangaUpdates, Shikimori, and Bangumi
- **Backup/Restore**: Library backup and restore functionality
- **Modern UI**: Built with Jetpack Compose and Material Design 3

---

## Architecture

The application follows a **Clean Architecture** pattern with clear separation of concerns:

```
┌─────────────────────────────────────────┐
│         Presentation Layer               │
│  (UI, ViewModels, Compose Screens)     │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│          Domain Layer                   │
│  (Use Cases, Repositories, Models)     │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│           Data Layer                    │
│  (Database, Network, Repositories)      │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         Source API Layer                │
│  (Source Interface, Extensions)         │
└─────────────────────────────────────────┘
```

### Architecture Principles

1. **Dependency Inversion**: High-level modules don't depend on low-level modules
2. **Single Responsibility**: Each module has a clear, single purpose
3. **Separation of Concerns**: UI, business logic, and data are separated
4. **Testability**: Domain layer is independent and easily testable

---

## Module Structure

### Core Modules

#### `app`
The main Android application module containing:
- **UI Components**: Activities, Compose screens, ViewModels
- **Reader Implementation**: `ReaderActivity`, `ReaderViewModel`, viewers
- **Data Management**: Download manager, cache, notifications
- **Extension Management**: Extension loader and installer
- **Tracking**: Tracker implementations (MAL, AniList, etc.)

**Key Packages:**
- `eu.kanade.tachiyomi.ui.reader` - Comic reader implementation
- `eu.kanade.tachiyomi.ui.manga` - Manga detail screens
- `eu.kanade.tachiyomi.ui.library` - Library management
- `eu.kanade.tachiyomi.data.download` - Download functionality
- `eu.kanade.tachiyomi.data.track` - Tracker integrations

#### `domain`
Business logic layer containing:
- **Use Cases (Interactors)**: Business operations
- **Repositories**: Abstract data access interfaces
- **Models**: Domain models (Manga, Chapter, Category, etc.)
- **Services**: Business services (preferences, sorting, etc.)

**Key Packages:**
- `tachiyomi.domain.manga` - Manga domain models and operations
- `tachiyomi.domain.chapter` - Chapter domain models and operations
- `tachiyomi.domain.category` - Category management
- `tachiyomi.domain.source` - Source management
- `tachiyomi.domain.history` - Reading history

#### `data`
Data access layer containing:
- **Repository Implementations**: Concrete implementations of domain repositories
- **Database**: SQLDelight database definitions and adapters
- **Mappers**: Conversion between database and domain models
- **Network**: HTTP client setup and network utilities

**Key Packages:**
- `tachiyomi.data.manga` - Manga data access
- `tachiyomi.data.chapter` - Chapter data access
- `tachiyomi.data.source` - Source data access
- `tachiyomi.data.history` - History data access

#### `source-api`
Source interface definitions:
- **Source Interface**: Base interface for all sources
- **CatalogueSource**: Interface for browseable sources
- **HttpSource**: Base class for HTTP-based sources
- **LocalSource**: Interface for local file sources
- **Models**: Source models (SManga, SChapter, Page)

**Key Files:**
- `eu.kanade.tachiyomi.source.Source` - Base source interface
- `eu.kanade.tachiyomi.source.CatalogueSource` - Browseable source interface
- `eu.kanade.tachiyomi.source.online.HttpSource` - HTTP source base class
- `source-local` - Local file source implementation

#### `presentation-core`
Shared presentation components:
- Reusable Compose components
- Shared UI utilities
- Common presentation logic

#### `core-common`
Common utilities:
- System utilities
- Logging
- Extensions
- Common data structures

#### `core-archive`
Archive handling:
- Archive reading (ZIP, RAR, etc.)
- Archive extraction utilities

#### `core-metadata`
Metadata management:
- Metadata models
- Metadata parsing and storage

#### `i18n`, `i18n-kmk`, `i18n-sy`
Internationalization:
- String resources
- Multi-language support

---

## Comic Reader Implementation

### Overview

The comic reader is the core feature of the application. It supports multiple reading modes and handles both online and offline content.

### Key Components

#### 1. ReaderActivity (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/ReaderActivity.kt`)

The main activity that hosts the reader. It:
- Manages the reader lifecycle
- Handles user interactions (taps, gestures, key events)
- Manages UI overlays (menu, settings, page number)
- Coordinates between ViewModel and Viewers

**Key Responsibilities:**
- Initialize reader with manga and chapter
- Handle orientation changes
- Manage fullscreen mode
- Handle brightness overlay
- Coordinate viewer lifecycle

#### 2. ReaderViewModel (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/ReaderViewModel.kt`)

Manages reader state and business logic:
- Chapter loading and navigation
- Page tracking
- Reading progress updates
- Viewer configuration
- Dialog management

**State Management:**
```kotlin
data class State(
    val manga: Manga?,
    val viewerChapters: ViewerChapters?,
    val currentPage: Int,
    val viewer: Viewer?,
    val dialog: Dialog?,
    val menuVisible: Boolean,
    // ... more state
)
```

#### 3. Viewer Interface (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/viewer/Viewer.kt`)

Abstract interface for different reading modes:

```kotlin
interface Viewer {
    fun getView(): View
    fun destroy()
    fun setChapters(chapters: ViewerChapters)
    fun moveToPage(page: ReaderPage)
    fun handleKeyEvent(event: KeyEvent): Boolean
    fun handleGenericMotionEvent(event: MotionEvent): Boolean
}
```

#### 4. Viewer Implementations

**Pager Viewers** (for traditional page-by-page reading):
- `L2RPagerViewer` - Left-to-right pager
- `R2LPagerViewer` - Right-to-left pager
- `VerticalPagerViewer` - Vertical pager

**Webtoon Viewer**:
- `WebtoonViewer` - Continuous vertical scrolling

**Location:** `app/src/main/java/eu/kanade/tachiyomi/ui/reader/viewer/`

#### 5. Reading Modes (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/setting/ReadingMode.kt`)

Enum defining available reading modes:

```kotlin
enum class ReadingMode {
    DEFAULT,
    LEFT_TO_RIGHT,    // Horizontal, page-by-page
    RIGHT_TO_LEFT,    // Horizontal, page-by-page (manga style)
    VERTICAL,         // Vertical, page-by-page
    WEBTOON,          // Continuous vertical scrolling
    CONTINUOUS_VERTICAL
}
```

#### 6. Page Loading System

**PageLoader** (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/loader/PageLoader.kt`)

Abstract base class for loading pages from different sources:

```kotlin
abstract class PageLoader {
    abstract var isLocal: Boolean
    abstract suspend fun getPages(): List<ReaderPage>
    open suspend fun loadPage(page: ReaderPage) {}
    open fun retryPage(page: ReaderPage) {}
    open fun recycle() {}
}
```

**Implementations:**

1. **HttpPageLoader** - Loads pages from online sources
   - Handles network requests
   - Manages page queue and priority
   - Supports preloading
   - Handles caching

2. **DownloadPageLoader** - Loads downloaded pages
   - Supports both directory and archive formats
   - Reads from local storage

3. **ArchivePageLoader** - Loads pages from archives (ZIP, RAR, etc.)
   - Extracts pages from archive files
   - Supports various archive formats

4. **DirectoryPageLoader** - Loads pages from directories
   - Reads image files from folders
   - Supports various image formats

5. **EpubPageLoader** - Loads pages from EPUB files
   - Parses EPUB structure
   - Extracts images and pages

**ChapterLoader** (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/loader/ChapterLoader.kt`)

Manages chapter loading:
- Determines appropriate PageLoader based on source type
- Handles chapter state (Wait, Loading, Loaded, Error)
- Manages page loader lifecycle

#### 7. Reader Models

**ReaderChapter** (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/model/ReaderChapter.kt`)

Represents a chapter in the reader:
```kotlin
data class ReaderChapter(val chapter: Chapter) {
    val stateFlow: MutableStateFlow<State>
    val pages: List<ReaderPage>?
    var pageLoader: PageLoader?
    var requestedPage: Int
    
    sealed interface State {
        object Wait : State
        object Loading : State
        data class Error(val error: Throwable) : State
        data class Loaded(val pages: List<ReaderPage>) : State
    }
}
```

**ReaderPage** (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/model/ReaderPage.kt`)

Represents a single page:
- Page index
- Image URL
- Status (Queue, Loading, Ready, Error)
- Image stream provider

**ViewerChapters** (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/model/ViewerChapters.kt`)

Manages adjacent chapters for navigation:
- Current chapter
- Previous chapter
- Next chapter
- Chapter list

### Reading Flow

1. **Initialization**
   ```
   User opens reader → ReaderActivity.onCreate()
   → ReaderViewModel.init(mangaId, chapterId)
   → Load manga and chapter from database
   → Create ChapterLoader
   → Load chapter pages
   ```

2. **Page Loading**
   ```
   ChapterLoader.loadChapter()
   → Determine PageLoader type (Http/Download/Archive/etc.)
   → PageLoader.getPages()
   → Create ReaderPage list
   → Update ReaderChapter state to Loaded
   ```

3. **Display**
   ```
   ReaderViewModel updates state
   → Viewer.setChapters()
   → Viewer displays pages
   → User navigates pages
   → Update reading progress
   ```

4. **Page Image Loading**
   ```
   Viewer requests page image
   → PageLoader.loadPage(page)
   → Fetch image (network/local/archive)
   → Update page status
   → Display in viewer
   ```

### Reader Configuration

**ReaderConfig** (`app/src/main/java/eu/kanade/tachiyomi/ui/reader/ReaderConfig.kt`)

Manages reader settings:
- Reading mode
- Orientation lock
- Brightness
- Background color
- Image scale type
- Page transitions
- Tap zones
- And more...

---

## Source API

### Overview

The Source API provides a flexible system for integrating comic sources. Sources can be:
- **Online Sources**: HTTP-based sources (websites, APIs)
- **Local Sources**: File system-based sources
- **Extension Sources**: Dynamically loaded via extensions

### Source Interface Hierarchy

```
Source (base interface)
├── CatalogueSource (browseable sources)
│   └── HttpSource (HTTP-based sources)
│       └── EnhancedHttpSource (SY enhancements)
└── LocalSource (local file sources)
```

### Core Interfaces

#### Source (`source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/Source.kt`)

Base interface for all sources:

```kotlin
interface Source {
    val id: Long
    val name: String
    val lang: String
    
    suspend fun getMangaDetails(manga: SManga): SManga
    suspend fun getChapterList(manga: SManga): List<SChapter>
    suspend fun getPageList(chapter: SChapter): List<Page>
}
```

#### CatalogueSource (`source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/CatalogueSource.kt`)

For sources that support browsing:

```kotlin
interface CatalogueSource : Source {
    val supportsLatest: Boolean
    
    suspend fun getPopularManga(page: Int): MangasPage
    suspend fun getSearchManga(page: Int, query: String, filters: FilterList): MangasPage
    suspend fun getLatestUpdates(page: Int): MangasPage
    fun getFilterList(): FilterList
}
```

#### HttpSource (`source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/online/HttpSource.kt`)

Base class for HTTP-based sources:

```kotlin
abstract class HttpSource : CatalogueSource {
    abstract val baseUrl: String
    open val versionId = 1
    protected val network: NetworkHelper
    protected val headers: Headers
    
    // Request methods
    protected abstract fun popularMangaRequest(page: Int): Request
    protected abstract fun popularMangaParse(response: Response): MangasPage
    
    protected abstract fun searchMangaRequest(page: Int, query: String, filters: FilterList): Request
    protected abstract fun searchMangaParse(response: Response): MangasPage
    
    protected abstract fun mangaDetailsRequest(manga: SManga): Request
    protected abstract fun mangaDetailsParse(response: Response): SManga
    
    protected abstract fun chapterListParse(response: Response): List<SChapter>
    
    protected abstract fun pageListRequest(chapter: SChapter): Request
    protected abstract fun pageListParse(response: Response): List<Page>
    
    protected abstract fun imageUrlParse(response: Response): String
}
```

### Source Models

#### SManga
Represents a manga/comic:
- `url`: Relative URL
- `title`: Title
- `author`: Author name
- `artist`: Artist name
- `description`: Description
- `genre`: Genres
- `status`: Publication status
- `thumbnail_url`: Cover image URL
- `initialized`: Whether details are loaded

#### SChapter
Represents a chapter:
- `url`: Relative URL
- `name`: Chapter name/number
- `date_upload`: Upload date
- `chapter_number`: Chapter number
- `scanlator`: Scanlation group

#### Page
Represents a page:
- `url`: Page URL
- `imageUrl`: Direct image URL (if available)
- `index`: Page index

### Creating a Source

Example HTTP source implementation:

```kotlin
class MySource : HttpSource() {
    override val id = 1234567890L
    override val name = "My Source"
    override val lang = "en"
    override val baseUrl = "https://example.com"
    override val supportsLatest = true
    
    override fun popularMangaRequest(page: Int): Request {
        return GET("$baseUrl/popular?page=$page", headers)
    }
    
    override fun popularMangaParse(response: Response): MangasPage {
        val document = response.asJsoup()
        val mangas = document.select("div.manga-item").map { element ->
            SManga.create().apply {
                title = element.select("h2").text()
                url = element.select("a").attr("href")
                thumbnail_url = element.select("img").attr("src")
            }
        }
        val hasNextPage = document.select("a.next").isNotEmpty()
        return MangasPage(mangas, hasNextPage)
    }
    
    override fun searchMangaRequest(page: Int, query: String, filters: FilterList): Request {
        return GET("$baseUrl/search?q=$query&page=$page", headers)
    }
    
    override fun searchMangaParse(response: Response): MangasPage {
        // Similar to popularMangaParse
    }
    
    override fun mangaDetailsParse(response: Response): SManga {
        val document = response.asJsoup()
        return SManga.create().apply {
            title = document.select("h1.title").text()
            author = document.select("span.author").text()
            description = document.select("div.description").text()
            // ... more fields
        }
    }
    
    override fun chapterListParse(response: Response): List<SChapter> {
        val document = response.asJsoup()
        return document.select("div.chapter").map { element ->
            SChapter.create().apply {
                name = element.select("span.name").text()
                url = element.select("a").attr("href")
                date_upload = parseDate(element.select("span.date").text())
            }
        }
    }
    
    override fun pageListParse(response: Response): List<Page> {
        val document = response.asJsoup()
        return document.select("img.page").mapIndexed { index, element ->
            Page(index, "", element.attr("src"))
        }
    }
    
    override fun imageUrlParse(response: Response): String {
        val document = response.asJsoup()
        return document.select("img#image").attr("src")
    }
}
```

### Source Manager

**SourceManager** (`domain/src/main/java/tachiyomi/domain/source/service/SourceManager.kt`)

Manages all available sources:
- Registers sources
- Provides source lookup
- Handles source lifecycle
- Manages source preferences

**AndroidSourceManager** (`app/src/main/java/eu/kanade/tachiyomi/source/AndroidSourceManager.kt`)

Android-specific implementation:
- Loads extension sources
- Manages source instances
- Handles source updates

### Extension System

Sources are loaded via **extensions** (separate APK files):

1. **Extension Structure**:
   - Contains source implementations
   - Declared in AndroidManifest metadata
   - Loaded dynamically at runtime

2. **Extension Loading**:
   - `ExtensionLoader` scans for source classes
   - Instantiates source objects
   - Registers with SourceManager

3. **Extension Installation**:
   - Download extension APK
   - Install via PackageInstaller or Shizuku
   - Load and register sources

---

## Data Flow

### Reading Flow

```
User Action
    ↓
UI Layer (Compose Screen)
    ↓
ViewModel (ReaderViewModel)
    ↓
Use Case (Domain Layer)
    ↓
Repository (Domain Interface)
    ↓
Repository Implementation (Data Layer)
    ↓
Database / Network / Source
    ↓
Data returned up the chain
    ↓
State updated in ViewModel
    ↓
UI recomposes
```

### Example: Loading a Chapter

1. **User opens reader** → `ReaderActivity.onCreate()`
2. **ViewModel initializes** → `ReaderViewModel.init(mangaId, chapterId)`
3. **Load manga** → `GetMangaById` use case → `MangaRepository` → Database
4. **Load chapter** → `GetChapterByUrlAndMangaId` use case → `ChapterRepository` → Database
5. **Load pages** → `ChapterLoader.loadChapter()` → `PageLoader.getPages()`
6. **If online**: `HttpPageLoader` → `Source.getPageList()` → Network request
7. **If downloaded**: `DownloadPageLoader` → Local file system
8. **Pages loaded** → `ReaderChapter.state = Loaded`
9. **Viewer displays** → `Viewer.setChapters()` → Pages rendered

### Example: Searching for Manga

1. **User enters search query** → Search screen
2. **ViewModel triggers search** → `BrowseSourceScreenModel.search()`
3. **Source search** → `CatalogueSource.getSearchManga()`
4. **Network request** → `HttpSource.searchMangaRequest()`
5. **Parse response** → `HttpSource.searchMangaParse()`
6. **Return results** → `MangasPage` with manga list
7. **Update UI** → Display search results

### Database Schema

The app uses **SQLDelight** for database management. Key tables:

- **manga**: Manga metadata
- **chapters**: Chapter information
- **history**: Reading history
- **categories**: Library categories
- **category_manga**: Manga-category relationships
- **tracks**: Tracker information
- **sources**: Source metadata

---

## Key Components

### 1. Download System

**DownloadManager** (`app/src/main/java/eu/kanade/tachiyomi/data/download/DownloadManager.kt`)

Manages chapter downloads:
- Queue management
- Download progress tracking
- Retry logic
- Storage management

**Downloader** (`app/src/main/java/eu/kanade/tachiyomi/data/download/Downloader.kt`)

Handles actual downloads:
- Downloads pages
- Saves to storage
- Creates chapter directories/archives
- Manages download cache

### 2. Library Management

**LibraryScreenModel** (`app/src/main/java/eu/kanade/tachiyomi/ui/library/LibraryScreenModel.kt`)

Manages library state:
- Manga list
- Categories
- Filtering and sorting
- Display modes

**Category Management**:
- Create/delete categories
- Assign manga to categories
- Category sorting
- Hidden categories

### 3. Backup System

**Backup Creation** (`app/src/main/java/eu/kanade/tachiyomi/data/backup/create/`)

Creates backup files:
- Library data
- Categories
- Reading progress
- Tracker information
- Preferences

**Backup Restore** (`app/src/main/java/eu/kanade/tachiyomi/data/backup/restore/`)

Restores from backup:
- Validates backup file
- Restores library
- Restores categories
- Restores progress

### 4. Tracker Integration

**Tracker Interface** (`app/src/main/java/eu/kanade/tachiyomi/data/track/Tracker.kt`)

Base interface for trackers:
- Login/logout
- Search manga
- Add/update/remove tracking
- Get tracking status

**Implementations**:
- `MyAnimeListTracker`
- `AniListTracker`
- `KitsuTracker`
- `MangaUpdatesTracker`
- `ShikimoriTracker`
- `BangumiTracker`

### 5. Cache System

**ChapterCache** (`app/src/main/java/eu/kanade/tachiyomi/data/cache/ChapterCache.kt`)

Caches chapter page lists:
- Reduces network requests
- Improves loading speed
- Manages cache size

**CoverCache** (`app/src/main/java/eu/kanade/tachiyomi/data/cache/CoverCache.kt`)

Caches manga covers:
- Disk-based caching
- Automatic cleanup
- Size management

---

## Integration Guide

### Using the Comic Reader in Your App

#### 1. Add Dependencies

Add the required modules to your `build.gradle.kts`:

```kotlin
dependencies {
    // Core modules
    implementation(projects.core.common)
    implementation(projects.core.archive)
    implementation(projects.coreMetadata)
    implementation(projects.sourceApi)
    implementation(projects.sourceLocal)
    implementation(projects.data)
    implementation(projects.domain)
    implementation(projects.presentationCore)
    
    // Reader dependencies
    implementation(compose.activity)
    implementation(compose.foundation)
    implementation(compose.material3.core)
    implementation(androidx.viewpager)
    implementation(libs.subsamplingscaleimageview)
    implementation(libs.image.decoder)
    // ... other dependencies
}
```

#### 2. Initialize Dependencies

Set up dependency injection (the app uses Injekt):

```kotlin
// Initialize core dependencies
Injekt.addModule(AppModule)
Injekt.addModule(PreferenceModule)

// Initialize database
val databaseHandler = AndroidDatabaseHandler(context)
Injekt.addSingleton(databaseHandler)

// Initialize source manager
val sourceManager = AndroidSourceManager(context)
Injekt.addSingleton(sourceManager)
```

#### 3. Launch Reader

```kotlin
val intent = Intent(context, ReaderActivity::class.java).apply {
    putExtra("manga", mangaId)
    putExtra("chapter", chapterId)
    putExtra("page", pageIndex) // Optional
}
context.startActivity(intent)
```

#### 4. Reader Configuration

Configure reader settings:

```kotlin
val readerConfig = ReaderConfig().apply {
    readingMode = ReadingMode.LEFT_TO_RIGHT
    orientation = OrientationMode.LANDSCAPE
    // ... more settings
}
```

### Reusable Components

#### 1. Source Integration

To add your own source:

1. Implement `HttpSource` or `LocalSource`
2. Register with `SourceManager`
3. Sources are automatically available in browse/search

#### 2. Viewer Customization

Create custom viewer:

```kotlin
class MyCustomViewer(
    activity: ReaderActivity,
    @ColorInt seedColor: Int?
) : Viewer {
    override fun getView(): View {
        // Return your custom view
    }
    
    override fun setChapters(chapters: ViewerChapters) {
        // Set chapters to display
    }
    
    override fun moveToPage(page: ReaderPage) {
        // Navigate to page
    }
    
    // Implement other methods...
}
```

#### 3. Page Loader

Create custom page loader:

```kotlin
class MyPageLoader : PageLoader() {
    override var isLocal = false
    
    override suspend fun getPages(): List<ReaderPage> {
        // Return page list
    }
    
    override suspend fun loadPage(page: ReaderPage) {
        // Load page image
    }
}
```

---

## Reusable Components

### What You Can Reuse

#### ✅ Highly Reusable

1. **Reader Components**
   - `ReaderActivity` - Full reader implementation
   - `ReaderViewModel` - Reader state management
   - Viewers (Pager, Webtoon) - Reading modes
   - `PageLoader` implementations - Page loading logic

2. **Source API**
   - `Source` interface - Source abstraction
   - `HttpSource` - HTTP source base class
   - Source models (SManga, SChapter, Page)

3. **Domain Layer**
   - Use cases - Business logic
   - Repository interfaces - Data access abstraction
   - Domain models - Business entities

4. **Data Layer**
   - Database schema - SQLDelight definitions
   - Repository implementations - Data access
   - Cache implementations - Caching logic

5. **Utilities**
   - Archive handling - ZIP/RAR/EPUB support
   - Image decoding - Various formats
   - Network utilities - HTTP client setup

#### ⚠️ Requires Adaptation

1. **UI Components**
   - Compose screens - Need UI customization
   - ViewModels - May need modification for your use case
   - Navigation - App-specific navigation

2. **App-Specific Features**
   - Extension system - May not be needed
   - Tracker integrations - Optional
   - Backup system - May need customization

### Integration Strategies

#### Strategy 1: Use Reader Only

If you only need the reader:

1. Copy reader-related modules
2. Implement minimal source interface
3. Provide your own data layer
4. Customize UI as needed

**Modules needed:**
- Reader UI components
- Viewer implementations
- Page loaders
- Core utilities

#### Strategy 2: Use Source API

If you want source integration:

1. Use `source-api` module
2. Implement your sources
3. Use existing source infrastructure
4. Integrate with your data layer

**Modules needed:**
- `source-api`
- Source implementations
- Network utilities

#### Strategy 3: Full Integration

If you want the complete system:

1. Use all core modules
2. Adapt UI to your design
3. Customize features as needed
4. Extend functionality

**Modules needed:**
- All core modules
- Domain and data layers
- Presentation components

### Key Files for Integration

**Reader:**
- `app/src/main/java/eu/kanade/tachiyomi/ui/reader/ReaderActivity.kt`
- `app/src/main/java/eu/kanade/tachiyomi/ui/reader/ReaderViewModel.kt`
- `app/src/main/java/eu/kanade/tachiyomi/ui/reader/viewer/Viewer.kt`
- `app/src/main/java/eu/kanade/tachiyomi/ui/reader/loader/PageLoader.kt`

**Source API:**
- `source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/Source.kt`
- `source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/online/HttpSource.kt`

**Domain:**
- `domain/src/main/java/tachiyomi/domain/manga/`
- `domain/src/main/java/tachiyomi/domain/chapter/`
- `domain/src/main/java/tachiyomi/domain/source/`

**Data:**
- `data/src/main/java/tachiyomi/data/`
- `data/src/main/sqldelight/`

---

## Additional Resources

### Code Locations

- **Reader**: `app/src/main/java/eu/kanade/tachiyomi/ui/reader/`
- **Viewers**: `app/src/main/java/eu/kanade/tachiyomi/ui/reader/viewer/`
- **Page Loaders**: `app/src/main/java/eu/kanade/tachiyomi/ui/reader/loader/`
- **Source API**: `source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/`
- **Domain**: `domain/src/main/java/tachiyomi/domain/`
- **Data**: `data/src/main/java/tachiyomi/data/`

### Key Technologies

- **Kotlin**: Primary language
- **Jetpack Compose**: UI framework
- **SQLDelight**: Database
- **OkHttp**: HTTP client
- **Coil**: Image loading
- **Coroutines**: Async operations
- **RxJava**: Legacy async (being phased out)

### License

Apache License 2.0 - See [LICENSE](LICENSE) file

---

## Conclusion

This documentation provides a comprehensive overview of the Komikku codebase. The architecture is well-structured with clear separation of concerns, making it relatively easy to extract and reuse components.

**For integrating the comic reader:**
- Focus on the `reader` package in the `app` module
- Understand the `Viewer` interface and implementations
- Study the `PageLoader` system for page loading
- Review the `Source` API if you need source integration

**For understanding the full system:**
- Study the domain layer for business logic
- Review the data layer for data access patterns
- Examine the source API for extensibility
- Look at the UI layer for presentation patterns

The codebase is actively maintained and follows modern Android development practices, making it a solid foundation for building comic reader applications.



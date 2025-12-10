# Quick Start Guide - Using Komikku's Comic Reader

This guide provides a quick reference for integrating Komikku's comic reader into your own application.

## Minimal Integration

### Step 1: Add Required Modules

In your `build.gradle.kts`, add:

```kotlin
dependencies {
    // Reader core
    implementation(projects.core.common)
    implementation(projects.core.archive)
    implementation(projects.sourceApi)
    implementation(projects.data)
    implementation(projects.domain)
    
    // Reader UI
    implementation(projects.presentationCore)
    implementation(compose.activity)
    implementation(compose.foundation)
    implementation(compose.material3.core)
    
    // Required libraries
    implementation(androidx.viewpager)
    implementation(libs.subsamplingscaleimageview)
    implementation(libs.image.decoder)
    implementation(libs.okhttp)
    implementation(libs.coil)
}
```

### Step 2: Initialize Dependencies

```kotlin
// In your Application class
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // Initialize dependency injection
        Injekt.addModule(AppModule)
        Injekt.addModule(PreferenceModule)
        
        // Initialize database
        val databaseHandler = AndroidDatabaseHandler(this)
        Injekt.addSingleton(databaseHandler)
        
        // Initialize source manager
        val sourceManager = AndroidSourceManager(this)
        Injekt.addSingleton(sourceManager)
    }
}
```

### Step 3: Launch Reader

```kotlin
// Simple reader launch
fun openReader(context: Context, mangaId: Long, chapterId: Long) {
    val intent = Intent(context, ReaderActivity::class.java).apply {
        putExtra("manga", mangaId)
        putExtra("chapter", chapterId)
    }
    context.startActivity(intent)
}

// With specific page
fun openReaderAtPage(context: Context, mangaId: Long, chapterId: Long, pageIndex: Int) {
    val intent = Intent(context, ReaderActivity::class.java).apply {
        putExtra("manga", mangaId)
        putExtra("chapter", chapterId)
        putExtra("page", pageIndex)
    }
    context.startActivity(intent)
}
```

## Using the Reader Components Directly

### Custom Reader Implementation

If you want more control, you can use the components directly:

```kotlin
class MyReaderActivity : ComponentActivity() {
    private val viewModel: ReaderViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val mangaId = intent.getLongExtra("manga", -1)
        val chapterId = intent.getLongExtra("chapter", -1)
        
        lifecycleScope.launch {
            viewModel.init(mangaId, chapterId)
        }
        
        setContent {
            ReaderScreen(viewModel = viewModel)
        }
    }
}
```

### Using Viewers Directly

```kotlin
// Create a viewer
val viewer = L2RPagerViewer(
    activity = this,
    seedColor = null // Optional theme color
)

// Set chapters
val chapters = ViewerChapters(
    currChapter = readerChapter,
    prevChapter = prevChapter,
    nextChapter = nextChapter
)
viewer.setChapters(chapters)

// Add to layout
val container = findViewById<FrameLayout>(R.id.viewer_container)
container.addView(viewer.getView())
```

## Creating a Simple Source

### HTTP Source Example

```kotlin
class MyComicSource : HttpSource() {
    override val id = 1234567890L
    override val name = "My Comic Source"
    override val lang = "en"
    override val baseUrl = "https://example.com"
    override val supportsLatest = true
    
    override fun popularMangaRequest(page: Int): Request {
        return GET("$baseUrl/popular?page=$page", headers)
    }
    
    override fun popularMangaParse(response: Response): MangasPage {
        val document = response.asJsoup()
        val mangas = document.select("div.comic").map { element ->
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
        return GET("$baseUrl/search?q=${query.encode()}&page=$page", headers)
    }
    
    override fun searchMangaParse(response: Response): MangasPage {
        return popularMangaParse(response) // Reuse parsing logic
    }
    
    override fun latestUpdatesRequest(page: Int): Request {
        return GET("$baseUrl/latest?page=$page", headers)
    }
    
    override fun latestUpdatesParse(response: Response): MangasPage {
        return popularMangaParse(response)
    }
    
    override fun mangaDetailsRequest(manga: SManga): Request {
        return GET(baseUrl + manga.url, headers)
    }
    
    override fun mangaDetailsParse(response: Response): SManga {
        val document = response.asJsoup()
        return SManga.create().apply {
            title = document.select("h1.title").text()
            author = document.select("span.author").text()
            description = document.select("div.description").text()
            thumbnail_url = document.select("img.cover").attr("src")
        }
    }
    
    override fun chapterListParse(response: Response): List<SChapter> {
        val document = response.asJsoup()
        return document.select("div.chapter").map { element ->
            SChapter.create().apply {
                name = element.select("span.name").text()
                url = element.select("a").attr("href")
                date_upload = System.currentTimeMillis()
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

### Local Source Example

```kotlin
class MyLocalSource : LocalSource() {
    override val id = 9876543210L
    override val name = "Local Files"
    override val lang = ""
    
    override fun getMangaList(): List<SManga> {
        // Scan local directories for comics
        val comicDirs = File("/sdcard/Comics").listFiles()?.filter { it.isDirectory }
        return comicDirs?.map { dir ->
            SManga.create().apply {
                title = dir.name
                url = dir.absolutePath
                thumbnail_url = findCoverImage(dir)?.absolutePath ?: ""
            }
        } ?: emptyList()
    }
    
    override fun getChapterList(manga: SManga): List<SChapter> {
        val mangaDir = File(manga.url)
        return mangaDir.listFiles()?.filter { it.isDirectory || isArchive(it) }?.mapIndexed { index, file ->
            SChapter.create().apply {
                name = file.nameWithoutExtension
                url = file.absolutePath
                chapter_number = index.toFloat()
            }
        } ?: emptyList()
    }
    
    override fun getPageList(chapter: SChapter): List<Page> {
        val chapterFile = File(chapter.url)
        return if (chapterFile.isDirectory) {
            // Directory of images
            chapterFile.listFiles()?.filter { it.isImageFile() }?.sorted()?.mapIndexed { index, file ->
                Page(index, file.absolutePath, file.absolutePath)
            } ?: emptyList()
        } else {
            // Archive file
            getPagesFromArchive(chapterFile)
        }
    }
    
    override fun getFormat(chapter: SChapter): Format {
        val file = File(chapter.url)
        return when {
            file.isDirectory -> Format.Directory(file)
            file.extension.equals("zip", ignoreCase = true) -> Format.Archive(file)
            file.extension.equals("epub", ignoreCase = true) -> Format.Epub(file)
            else -> throw UnsupportedOperationException("Unsupported format")
        }
    }
}
```

## Reading Modes

### Available Modes

```kotlin
enum class ReadingMode {
    LEFT_TO_RIGHT,      // Traditional Western comics
    RIGHT_TO_LEFT,      // Manga style
    VERTICAL,           // Vertical page-by-page
    WEBTOON,            // Continuous vertical scroll
    CONTINUOUS_VERTICAL // Continuous vertical (alternative)
}
```

### Setting Reading Mode

```kotlin
// In ReaderViewModel or ReaderConfig
readerConfig.readingMode = ReadingMode.WEBTOON

// Or programmatically
viewModel.setReadingMode(ReadingMode.RIGHT_TO_LEFT)
```

## Page Loading

### Understanding Page Loaders

The app uses different page loaders based on content source:

- **HttpPageLoader**: For online sources
- **DownloadPageLoader**: For downloaded content
- **ArchivePageLoader**: For ZIP/RAR files
- **DirectoryPageLoader**: For image folders
- **EpubPageLoader**: For EPUB files

### Custom Page Loader

```kotlin
class MyCustomPageLoader : PageLoader() {
    override var isLocal = true
    
    override suspend fun getPages(): List<ReaderPage> {
        // Return list of pages
        return listOf(
            ReaderPage(0, "page1.jpg", "file:///path/to/page1.jpg"),
            ReaderPage(1, "page2.jpg", "file:///path/to/page2.jpg")
        )
    }
    
    override suspend fun loadPage(page: ReaderPage) {
        // Load page image
        val imageStream = FileInputStream(File(page.imageUrl))
        page.imageStream = imageStream
        page.status = Page.State.Ready
    }
}
```

## Common Use Cases

### 1. Read from Local Files

```kotlin
// Create local source
val localSource = MyLocalSource()

// Get manga
val manga = localSource.getMangaList().first()

// Get chapters
val chapters = localSource.getChapterList(manga)

// Open reader
openReader(context, manga.id, chapters.first().id)
```

### 2. Read from Online Source

```kotlin
// Get source
val sourceManager = Injekt.get<SourceManager>()
val source = sourceManager.getSource(1234567890L) as HttpSource

// Search for manga
val results = source.getSearchManga(1, "One Piece", FilterList())

// Get manga details
val manga = source.getMangaDetails(results.mangas.first())

// Get chapters
val chapters = source.getChapterList(manga)

// Open reader
openReader(context, manga.id, chapters.first().id)
```

### 3. Custom Reader UI

```kotlin
@Composable
fun CustomReaderScreen(viewModel: ReaderViewModel) {
    val state by viewModel.state.collectAsState()
    
    Box(modifier = Modifier.fillMaxSize()) {
        // Viewer container
        AndroidView(
            factory = { context ->
                state.viewer?.getView() ?: View(context)
            },
            modifier = Modifier.fillMaxSize()
        )
        
        // Custom overlay
        if (state.menuVisible) {
            CustomReaderMenu(
                onClose = { viewModel.toggleMenu() },
                onSettings = { viewModel.showSettings() }
            )
        }
    }
}
```

## Tips

1. **Performance**: Use page preloading for smoother reading
2. **Caching**: Enable chapter cache to reduce network requests
3. **Memory**: Recycle page loaders when not in use
4. **Offline**: Use DownloadPageLoader for offline reading
5. **Customization**: Extend Viewer interface for custom reading modes

## Troubleshooting

### Reader doesn't open
- Check that manga and chapter IDs are valid
- Ensure dependencies are initialized
- Verify source is available

### Pages don't load
- Check network permissions for online sources
- Verify file permissions for local sources
- Check page loader implementation

### Viewer not displaying
- Ensure viewer is properly initialized
- Check that chapters are loaded
- Verify view is added to container

## Next Steps

- Read the full [DOCUMENTATION.md](DOCUMENTATION.md) for detailed information
- Explore the source code in `app/src/main/java/eu/kanade/tachiyomi/ui/reader/`
- Check example implementations in the codebase
- Review the Source API documentation






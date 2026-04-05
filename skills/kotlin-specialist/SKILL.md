---
name: kotlin-specialist
description: Kotlin development: coroutines, flows, Jetpack Compose, multiplatform (KMP), Spring Boot with Kotlin, testing patterns, and migration from Java.
---

# Kotlin Specialist

Expert guidance for Kotlin development across the full ecosystem: coroutines and structured concurrency, reactive Flows, Jetpack Compose UI, Kotlin Multiplatform shared code, Spring Boot idiomatic Kotlin, testing strategies, and seamless Java interop. Use this skill whenever you are writing, reviewing, or migrating Kotlin code on any platform.

## Table of Contents

- [1. Coroutines](#1-coroutines)
- [2. Flows](#2-flows)
- [3. Jetpack Compose](#3-jetpack-compose)
- [4. Kotlin Multiplatform](#4-kotlin-multiplatform)
- [5. Spring Boot with Kotlin](#5-spring-boot-with-kotlin)
- [6. Kotlin Idioms](#6-kotlin-idioms)
- [7. Testing](#7-testing)
- [8. Java Interop](#8-java-interop)

---

## 1. Coroutines

Kotlin coroutines provide lightweight concurrency without callback hell. The key concepts are **structured concurrency**, **dispatchers**, and **cooperative cancellation**.

### Launch and Async

`launch` is fire-and-forget (returns `Job`); `async` returns a `Deferred<T>` you can `await()`.

```kotlin
fun main() = runBlocking {
    val job = launch { delay(1000); println("World") }
    val deferred = async { delay(500); 42 }

    println("Hello")
    println("The answer is ${deferred.await()}")
    job.join()
}
```

### Structured Concurrency

Every coroutine belongs to a `CoroutineScope`. When a scope is cancelled, all children are cancelled.

```kotlin
class UserRepository(private val api: UserApi) {
    suspend fun fetchUser(id: String): User = coroutineScope {
        val profile = async { api.getProfile(id) }
        val settings = async { api.getSettings(id) }
        User(profile.await(), settings.await())
    }
}
```

### supervisorScope

Use `supervisorScope` when child failures should not cancel siblings.

```kotlin
suspend fun loadDashboard(): Dashboard = supervisorScope {
    val news = async { fetchNews() }
    val weather = async { fetchWeather() }

    Dashboard(
        news = runCatching { news.await() }.getOrDefault(emptyList()),
        weather = runCatching { weather.await() }.getOrNull(),
    )
}
```

### Exception Handling

`CoroutineExceptionHandler` catches uncaught exceptions from `launch`. For `async`, exceptions propagate through `await()`.

```kotlin
val handler = CoroutineExceptionHandler { _, ex ->
    logger.error("Unhandled: ${ex.message}", ex)
}
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default + handler)

scope.launch { riskyOperation() }   // handler logs if this throws
```

### Cancellation

Coroutines are cooperatively cancellable. Use `ensureActive()` or `yield()` in tight loops.

```kotlin
suspend fun processLargeFile(file: File) = coroutineScope {
    file.bufferedReader().useLines { lines ->
        lines.forEach { line ->
            ensureActive()   // throws CancellationException if cancelled
            processLine(line)
        }
    }
}

suspend fun acquireAndProcess() {
    val resource = acquireResource()
    try { resource.process() }
    finally { withContext(NonCancellable) { resource.release() } }
}
```

### Anti-Patterns

```kotlin
// WRONG: blocking the main thread
fun loadData() = runBlocking { api.fetch() }
// RIGHT: suspend or launch from correct scope
suspend fun loadData() = withContext(Dispatchers.IO) { api.fetch() }

// WRONG: swallowing CancellationException
try { delay(1000) } catch (e: Exception) { /* swallows cancellation */ }
// RIGHT: rethrow CancellationException
try { delay(1000) } catch (e: CancellationException) { throw e } catch (e: Exception) { handleError(e) }
```

---

## 2. Flows

Flows are Kotlin's cold asynchronous streams. They emit values lazily when collected.

### Cold Flow Basics

```kotlin
fun fibonacci(): Flow<Long> = flow {
    var a = 0L; var b = 1L
    while (true) { emit(a); val next = a + b; a = b; b = next }
}

suspend fun main() { fibonacci().take(10).collect { println(it) } }
```

### StateFlow and SharedFlow

`StateFlow` holds a single current value (replaces `LiveData`). `SharedFlow` is a hot broadcast stream.

```kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()
    fun increment() { _count.update { it + 1 } }
}

class EventBus {
    private val _events = MutableSharedFlow<Event>(
        replay = 0, extraBufferCapacity = 64,
        onBufferOverflow = BufferOverflow.DROP_OLDEST,
    )
    val events: SharedFlow<Event> = _events.asSharedFlow()
    suspend fun emit(event: Event) { _events.emit(event) }
}
```

### Operators

```kotlin
fun searchResults(query: Flow<String>, api: SearchApi): Flow<List<Result>> =
    query
        .debounce(300)
        .filter { it.length >= 2 }
        .distinctUntilChanged()
        .flatMapLatest { q -> flow { emit(api.search(q)) } }
        .catch { emit(emptyList()) }
```

### flowOn and Combining

`flowOn` changes the **upstream** dispatcher. `combine` merges multiple flows.

```kotlin
fun readSensor(): Flow<SensorReading> = flow {
    while (currentCoroutineContext().isActive) { emit(sensor.read()); delay(100) }
}.flowOn(Dispatchers.IO).conflate()

val userLabel: Flow<String> = combine(userName, userAge) { name, age -> "$name ($age)" }
```

### Testing Flows with Turbine

```kotlin
@Test
fun `counter increments`() = runTest {
    val vm = CounterViewModel()
    vm.count.test {
        assertEquals(0, awaitItem())
        vm.increment()
        assertEquals(1, awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Anti-Patterns

```kotlin
// WRONG: Channel when a flow suffices — manual lifecycle management
val channel = Channel<Int>()
// RIGHT: cold flow
val numbers: Flow<Int> = flow { emit(1); emit(2) }

// WRONG: collecting in init{} — leaks if scope outlives UI
init { scope.launch { flow.collect { update(it) } } }
// RIGHT: lifecycle-aware collection
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) { flow.collect { update(it) } }
}
```

---

## 3. Jetpack Compose

Compose is Android's declarative UI toolkit. Composable functions re-execute ("recompose") when state changes.

### State Hoisting

Keep state in the caller; pass it down and send events up.

```kotlin
@Composable
fun CounterScreen() {
    var count by remember { mutableIntStateOf(0) }
    Counter(count = count, onIncrement = { count++ })
}

@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text("Count: $count", style = MaterialTheme.typography.displaySmall)
        Spacer(Modifier.height(8.dp))
        Button(onClick = onIncrement) { Text("Add one") }
    }
}
```

### remember and derivedStateOf

```kotlin
@Composable
fun FilteredList(items: List<String>, query: String) {
    val filtered by remember(items, query) {
        derivedStateOf { items.filter { it.contains(query, ignoreCase = true) } }
    }
    LazyColumn { items(filtered) { item -> Text(item) } }
}
```

### LazyColumn with Keys

Always provide stable keys so Compose can efficiently diff the list.

```kotlin
@Composable
fun UserList(users: List<User>) {
    LazyColumn {
        items(items = users, key = { it.id }) { user ->
            Row(Modifier.fillMaxWidth().padding(16.dp, 12.dp)) {
                Text(user.name, style = MaterialTheme.typography.bodyLarge)
            }
        }
    }
}
```

### Navigation

```kotlin
@Composable
fun AppNavGraph(navController: NavHostController = rememberNavController()) {
    NavHost(navController, startDestination = "home") {
        composable("home") {
            HomeScreen(onUserClick = { id -> navController.navigate("user/$id") })
        }
        composable(
            "user/{userId}",
            arguments = listOf(navArgument("userId") { type = NavType.StringType }),
        ) { entry ->
            UserDetailScreen(entry.arguments?.getString("userId") ?: return@composable)
        }
    }
}
```

### Side Effects

```kotlin
@Composable
fun TimerScreen(viewModel: TimerViewModel = viewModel()) {
    val seconds by viewModel.seconds.collectAsStateWithLifecycle()
    LaunchedEffect(Unit) { viewModel.startTimer() }
    DisposableEffect(Unit) {
        val listener = SensorManager.registerListener(...)
        onDispose { listener.unregister() }
    }
    Text("Elapsed: ${seconds}s")
}
```

### Anti-Patterns

```kotlin
// WRONG: objects recreated every recomposition
@Composable fun Bad() {
    val fmt = SimpleDateFormat("yyyy-MM-dd")   // new instance each time
}
// RIGHT: remember it
@Composable fun Good() {
    val fmt = remember { SimpleDateFormat("yyyy-MM-dd", Locale.US) }
}

// WRONG: side effect in composable body — runs on every recomposition
@Composable fun Bad(vm: MyViewModel) { vm.trackScreenView() }
// RIGHT: LaunchedEffect
@Composable fun Good(vm: MyViewModel) { LaunchedEffect(Unit) { vm.trackScreenView() } }
```

---

## 4. Kotlin Multiplatform

KMP lets you share business logic across Android, iOS, Desktop, and Web while keeping platform-specific UI and APIs.

### Project Structure

```
shared/src/
  commonMain/kotlin/    # shared code
  commonTest/kotlin/    # shared tests
  androidMain/kotlin/   # Android-specific
  iosMain/kotlin/       # iOS-specific
androidApp/             # Android UI (Compose)
iosApp/                 # iOS UI (SwiftUI)
```

### expect/actual

Declare `expect` in common code; provide `actual` implementations per platform.

```kotlin
// commonMain
expect class PlatformLogger() { fun log(message: String) }
expect fun currentTimeMillis(): Long

// androidMain
actual class PlatformLogger {
    actual fun log(message: String) { android.util.Log.d("App", message) }
}
actual fun currentTimeMillis(): Long = System.currentTimeMillis()

// iosMain
actual class PlatformLogger {
    actual fun log(message: String) { platform.Foundation.NSLog(message) }
}
actual fun currentTimeMillis(): Long =
    (platform.Foundation.NSDate().timeIntervalSince1970 * 1000).toLong()
```

### Shared Module with Ktor

```kotlin
// commonMain
@Serializable
data class Article(val id: Long, val title: String, val body: String)

class ArticleRepository(private val client: HttpClient) {
    suspend fun getArticles(): List<Article> =
        client.get("https://api.example.com/articles").body()
}
// Engine per platform: OkHttp (Android), Darwin (iOS), CIO (JVM)
```

### Compose Multiplatform

```kotlin
// commonMain — shared UI across Android, iOS, Desktop, Web
@Composable
fun App() {
    MaterialTheme {
        var text by remember { mutableStateOf("Hello, World!") }
        Column(Modifier.fillMaxSize().padding(16.dp), Arrangement.Center, Alignment.CenterHorizontally) {
            Text(text, style = MaterialTheme.typography.headlineMedium)
            Spacer(Modifier.height(16.dp))
            Button(onClick = { text = "Clicked!" }) { Text("Click me") }
        }
    }
}
```

### Anti-Patterns

```kotlin
// WRONG: platform imports in commonMain
import android.content.Context   // won't compile on iOS
// RIGHT: use expect/actual to abstract platform dependencies

// WRONG: duplicating logic per platform (copy-paste across androidMain/iosMain)
// RIGHT: put shared logic in commonMain, only platform glue in *Main
```

---

## 5. Spring Boot with Kotlin

Kotlin is a first-class language for Spring Boot. Use DSL configuration, data classes, extension functions, and coroutines.

### Application and Bean DSL

```kotlin
@SpringBootApplication
class ArticleApplication

fun main(args: Array<String>) {
    runApplication<ArticleApplication>(*args) {
        addInitializers(beans {
            bean<ArticleService>()
            bean { ArticleHandler(ref()) }
        })
    }
}
```

### Router DSL (WebFlux)

```kotlin
@Bean
fun routes(handler: ArticleHandler) = coRouter {
    "/api/v1/articles".nest {
        GET("", handler::listAll)
        GET("/{id}", handler::getById)
        POST("", handler::create)
        PUT("/{id}", handler::update)
        DELETE("/{id}", handler::delete)
    }
}
```

### Coroutines in Spring Controllers

Spring WebFlux supports suspending functions and `Flow` return types natively.

```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserController(private val userService: UserService) {
    @GetMapping
    suspend fun listUsers(): List<UserDto> = userService.findAll()

    @GetMapping("/{id}")
    suspend fun getUser(@PathVariable id: Long): UserDto =
        userService.findById(id) ?: throw ResponseStatusException(HttpStatus.NOT_FOUND)

    @GetMapping("/stream", produces = [MediaType.APPLICATION_NDJSON_VALUE])
    fun streamUsers(): Flow<UserDto> = userService.streamAll()
}
```

### Data Classes, Validation, and Extensions

```kotlin
@Table("articles")
data class Article(
    @Id val id: Long? = null,
    val title: String,
    val body: String,
    val createdAt: Instant = Instant.now(),
)

data class CreateArticleRequest(
    @field:NotBlank val title: String,
    @field:Size(min = 10, max = 10_000) val body: String,
)
fun CreateArticleRequest.toEntity() = Article(title = title, body = body)

// Repository extension
suspend fun ArticleRepository.findByIdOrThrow(id: Long): Article =
    findById(id) ?: throw ResponseStatusException(HttpStatus.NOT_FOUND, "Article $id not found")
```

### Anti-Patterns

```kotlin
// WRONG: nullable injection
class MyService { @Autowired var repo: ArticleRepository? = null }
// RIGHT: constructor injection
class MyService(private val repo: ArticleRepository)

// WRONG: blocking in a coroutine controller
@GetMapping("/slow") suspend fun slow(): String { Thread.sleep(5000); return "done" }
// RIGHT: use delay or withContext(Dispatchers.IO)
@GetMapping("/slow") suspend fun slow(): String { delay(5000); return "done" }
```

---

## 6. Kotlin Idioms

### Scope Functions

| Function | Object | Returns | Use When |
|----------|--------|---------|----------|
| `let`    | `it`   | lambda  | null checks, transforms |
| `run`    | `this` | lambda  | config + compute |
| `apply`  | `this` | object  | object configuration |
| `also`   | `it`   | object  | side effects (logging) |

```kotlin
val length: Int? = name?.let { it.trim().length }

val request = Request.Builder().apply {
    url("https://api.example.com")
    header("Authorization", "Bearer $token")
    post(body)
}.build()

val user = userRepository.findById(id)
    .also { logger.debug("Found user: $it") }
    ?: throw NotFoundException("User $id")
```

### Sealed Classes and When

```kotlin
sealed interface Result<out T> {
    data class Success<T>(val data: T) : Result<T>
    data class Failure(val error: Throwable) : Result<Nothing>
    data object Loading : Result<Nothing>
}

fun <T> Result<T>.fold(
    onSuccess: (T) -> Unit,
    onFailure: (Throwable) -> Unit,
    onLoading: () -> Unit = {},
) = when (this) {
    is Result.Success -> onSuccess(data)
    is Result.Failure -> onFailure(error)
    is Result.Loading -> onLoading()
}
```

### Value Classes

Zero-overhead type safety at compile time.

```kotlin
@JvmInline value class UserId(val value: Long)
@JvmInline value class Email(val value: String) {
    init { require("@" in value) { "Invalid email: $value" } }
}
// findUser(Email("x@y.com")) won't compile — type mismatch with UserId
```

### Delegation

```kotlin
class LoggingList<T>(
    private val inner: MutableList<T> = mutableListOf()
) : MutableList<T> by inner {
    override fun add(element: T): Boolean {
        println("Adding $element")
        return inner.add(element)
    }
}
```

### DSL Builders

Use `@DslMarker` to prevent scope leakage in nested builders.

```kotlin
@DslMarker annotation class HtmlDsl

@HtmlDsl class Body {
    private val children = mutableListOf<String>()
    fun h1(text: String) { children += "<h1>$text</h1>" }
    fun p(text: String) { children += "<p>$text</p>" }
    fun render() = "<body>${children.joinToString("")}</body>"
}

fun html(block: Body.() -> Unit): String = "<html>${Body().apply(block).render()}</html>"
val page = html { h1("Hello"); p("World") }
```

### Anti-Patterns

```kotlin
// WRONG: chained scope functions — unreadable
name?.let { it.trim() }?.let { it.uppercase() }?.let { it.take(10) }
// RIGHT: simple chaining
name?.trim()?.uppercase()?.take(10)

// WRONG: non-exhaustive when on sealed type
when (result) { is Result.Success -> show(result.data); else -> {} }
// RIGHT: handle all branches explicitly
```

---

## 7. Testing

Kotlin testing leverages JUnit 5, MockK, Kotest, coroutine test utilities, and Compose test APIs.

### JUnit 5

```kotlin
class CalculatorTest {
    private val calc = Calculator()

    @Test fun `add two numbers`() { assertEquals(4, calc.add(2, 2)) }

    @ParameterizedTest
    @CsvSource("1,1,2", "0,0,0", "-1,1,0")
    fun `addition variants`(a: Int, b: Int, expected: Int) {
        assertEquals(expected, calc.add(a, b))
    }

    @Nested inner class Division {
        @Test fun `divide by zero throws`() {
            assertThrows<ArithmeticException> { calc.divide(1, 0) }
        }
    }
}
```

### MockK

```kotlin
class UserServiceTest {
    private val repo = mockk<UserRepository>()
    private val notifier = mockk<NotificationService>(relaxed = true)
    private val service = UserService(repo, notifier)

    @Test fun `createUser saves and notifies`() = runTest {
        coEvery { repo.save(any()) } returns User(1, "Alice")
        val result = service.createUser("Alice")
        assertEquals("Alice", result.name)
        coVerify(exactly = 1) { repo.save(match { it.name == "Alice" }) }
        coVerify { notifier.send(any(), eq("Welcome, Alice!")) }
    }
}
```

### Kotest

```kotlin
class StringSpecExample : StringSpec({
    "length returns string length" { "hello".length shouldBe 5 }
    "startsWith tests prefix" { "world" should startWith("wor") }
})

class UserValidatorSpec : BehaviorSpec({
    given("valid email") {
        `when`("validated") {
            then("is valid") { UserValidator.validate(validUser).isValid shouldBe true }
        }
    }
})
```

### Coroutine Testing with runTest

```kotlin
@Test fun `periodic sync respects interval`() = runTest {
    val api = mockk<Api>()
    coEvery { api.fetchData() } returns emptyList()
    val sync = PeriodicSync(api, interval = 1.hours)
    val job = launch { sync.start() }
    advanceTimeBy(3.hours)
    coVerify(exactly = 3) { api.fetchData() }
    job.cancel()
}
```

### Compose UI Testing

```kotlin
class CounterScreenTest {
    @get:Rule val composeRule = createComposeRule()

    @Test fun `counter increments on click`() {
        composeRule.setContent { CounterScreen() }
        composeRule.onNodeWithText("Count: 0").assertIsDisplayed()
        composeRule.onNodeWithText("Add one").performClick()
        composeRule.onNodeWithText("Count: 1").assertIsDisplayed()
    }
}
```

### Testcontainers

```kotlin
@Testcontainers @SpringBootTest
class ArticleRepoIntegrationTest {
    companion object {
        @Container val postgres = PostgreSQLContainer("postgres:16-alpine")
            .apply { withDatabaseName("testdb"); withUsername("test"); withPassword("test") }

        @JvmStatic @DynamicPropertySource
        fun props(reg: DynamicPropertyRegistry) {
            reg.add("spring.r2dbc.url") { "r2dbc:postgresql://${postgres.host}:${postgres.firstMappedPort}/testdb" }
            reg.add("spring.r2dbc.username", postgres::getUsername)
            reg.add("spring.r2dbc.password", postgres::getPassword)
        }
    }

    @Autowired lateinit var repo: ArticleRepository

    @Test fun `save and retrieve`() = runTest {
        val saved = repo.save(Article(title = "Test", body = "Content"))
        assertNotNull(repo.findById(saved.id!!))
    }
}
```

### Anti-Patterns

```kotlin
// WRONG: runBlocking in tests — does NOT advance virtual time
@Test fun bad() = runBlocking { delay(10_000) }   // actually waits 10s
// RIGHT: runTest — virtual time, instant completion
@Test fun good() = runTest { delay(10_000) }       // completes instantly

// WRONG: asserting before recomposition settles
composeRule.onNodeWithText("Loading").assertIsDisplayed()
// RIGHT: waitUntil
composeRule.waitUntil(3000) {
    composeRule.onAllNodesWithText("Loaded").fetchSemanticsNodes().isNotEmpty()
}
```

---

## 8. Java Interop

Kotlin is designed for seamless Java interoperability. Understanding edge cases prevents subtle bugs during migration.

### Calling Java from Kotlin

```kotlin
// Java getters/setters map to Kotlin properties automatically
val user = JavaUser()
user.name = "Alice"       // calls setName()
println(user.name)        // calls getName()
```

### Nullable Platform Types

Java types without nullability annotations arrive as **platform types** (`String!`). Always handle them explicitly.

```kotlin
// Dangerous — crashes if Java returns null
val title: String = javaObj.title            // implicit !!

// Safe
val title: String = javaObj.title ?: "Untitled"

// Best — annotate the Java source with @Nullable / @NotNull
```

### SAM Conversions

Kotlin lambdas substitute Java Single Abstract Method interfaces.

```kotlin
button.setOnClickListener { view -> println("Clicked: ${view.id}") }
executor.execute(Runnable { println("Running") })   // explicit SAM when ambiguous
```

### Annotations for Java Callers

Control generated bytecode when Java code calls your Kotlin.

```kotlin
class ApiClient(@get:JvmName("getBaseUrl") val baseUrl: String) {
    companion object {
        @JvmStatic fun create(url: String) = ApiClient(url)
        @JvmField val DEFAULT_TIMEOUT = 30_000L
    }
    @JvmOverloads
    fun fetch(path: String, headers: Map<String, String> = emptyMap()): Response { ... }
}
// Java: ApiClient.create("..."); ApiClient.DEFAULT_TIMEOUT; client.fetch("/users")
```

### Incremental Migration Strategy

Migrate file-by-file, starting with leaf classes. Order: (1) POJOs/data classes, (2) utility classes, (3) service classes, (4) framework-heavy classes (last).

```kotlin
// POJO -> data class (60 lines Java -> 1 line Kotlin)
data class User(val id: Long, val name: String, val email: String, val createdAt: Instant = Instant.now())

// Utility -> extension function
fun String.capitalizeFirst(): String =
    replaceFirstChar { if (it.isLowerCase()) it.titlecase() else it.toString() }

// Gradual coroutine adoption — keep blocking wrapper for Java callers
class UserService(private val repo: UserRepository) {
    suspend fun findUser(id: Long): User? = repo.findById(id)
    @JvmName("findUserBlocking")
    fun findUserBlocking(id: Long): User? = runBlocking { findUser(id) }
}
```

### Anti-Patterns

```kotlin
// WRONG: ignoring platform types
val name: String = javaObject.name          // NPE if null
// RIGHT: treat as nullable
val name: String = javaObject.name ?: "Unknown"

// WRONG: converting everything at once — massive PR, high risk
// RIGHT: one file per PR, ensure Java callers still compile

// WRONG: !! as a migration crutch
val value = javaMap.get(key)!!
// RIGHT: explicit null handling
val value = javaMap[key] ?: error("Missing key: $key")
```

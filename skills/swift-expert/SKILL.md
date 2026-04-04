---
name: swift-expert
description: Swift/SwiftUI development: protocol-oriented programming, concurrency with async/await, Combine, UIKit interop, App Store submission, and iOS/macOS patterns.
---

# Swift Expert

Expert-level Swift and SwiftUI development skill for iOS, macOS, watchOS, and visionOS. Covers protocol-oriented programming, modern concurrency with async/await and actors, SwiftUI view composition, Combine reactive streams, UIKit interop, data persistence with SwiftData and Core Data, app architecture patterns, and App Store distribution. Use this when writing, reviewing, or architecting Swift code that needs to be idiomatic, performant, and ready for production.

## Table of Contents

- [Protocol-Oriented Programming](#protocol-oriented-programming)
- [Swift Concurrency](#swift-concurrency)
- [SwiftUI](#swiftui)
- [Combine Framework](#combine-framework)
- [UIKit Interop](#uikit-interop)
- [Data Persistence](#data-persistence)
- [App Architecture](#app-architecture)
- [App Store and Distribution](#app-store-and-distribution)

---

## Protocol-Oriented Programming

### Protocols with Default Implementations

Prefer protocols over class inheritance. Provide default behavior through extensions so conforming types get functionality for free.

```swift
protocol Cacheable {
    var cacheKey: String { get }
    var cacheDuration: TimeInterval { get }
    func serialize() -> Data
    static func deserialize(from data: Data) -> Self?
}

extension Cacheable where Self: Codable {
    var cacheDuration: TimeInterval { 300 }
    func serialize() -> Data { (try? JSONEncoder().encode(self)) ?? Data() }
    static func deserialize(from data: Data) -> Self? { try? JSONDecoder().decode(Self.self, from: data) }
}

// UserProfile gets serialize/deserialize for free via the constrained extension
struct UserProfile: Codable, Cacheable {
    let id: UUID; let name: String
    var cacheKey: String { "user_\(id.uuidString)" }
}
```

### Associated Types and Type Erasure

```swift
protocol Repository {
    associatedtype Entity: Identifiable
    func fetch(id: Entity.ID) async throws -> Entity
    func save(_ entity: Entity) async throws
    func delete(id: Entity.ID) async throws
}

struct UserRepository: Repository {
    typealias Entity = User
    func fetch(id: UUID) async throws -> User { try await APIClient.shared.get("/users/\(id)") }
    func save(_ entity: User) async throws { try await APIClient.shared.put("/users/\(entity.id)", body: entity) }
    func delete(id: UUID) async throws { try await APIClient.shared.delete("/users/\(id)") }
}

// Type-erased wrapper for heterogeneous collections
struct AnyRepository<T: Identifiable>: Repository {
    private let _fetch: (T.ID) async throws -> T
    private let _save: (T) async throws -> Void
    private let _delete: (T.ID) async throws -> Void

    init<R: Repository>(_ repo: R) where R.Entity == T {
        _fetch = repo.fetch; _save = repo.save; _delete = repo.delete
    }
    func fetch(id: T.ID) async throws -> T { try await _fetch(id) }
    func save(_ entity: T) async throws { try await _save(entity) }
    func delete(id: T.ID) async throws { try await _delete(id) }
}
```

### Protocol Composition and Existential Types

```swift
// Compose small protocols at the use site
func syncToCloud<T: Identifiable & Timestamped & Codable>(_ item: T) async throws {
    let data = try JSONEncoder().encode(item)
    try await CloudSync.upload(key: item.id.uuidString, data: data, lastModified: item.updatedAt)
}

// Named composition when reused frequently
protocol PersistableEntity: Identifiable, Timestamped, SoftDeletable, Codable {}

// `some` -- opaque type, fixed concrete type, no boxing overhead
func makeDefaultCache() -> some Cacheable { UserProfile(id: UUID(), name: "default") }

// `any` -- existential, allows heterogeneous collections
func allCacheables() -> [any Cacheable] {
    [UserProfile(id: UUID(), name: "Alice"), SessionToken(value: "abc123")]
}
// Prefer `some` when possible; use `any` only for heterogeneous collections.
```

**Anti-patterns to avoid:**
- Protocols with 10+ requirements -- split into composable pieces.
- Class inheritance when protocol composition is more flexible.
- Reaching for `any` when `some` suffices -- existentials have boxing overhead.

---

## Swift Concurrency

### async/await Basics

```swift
func fetchUser(id: UUID) async throws -> User {
    let (data, response) = try await URLSession.shared.data(for: makeRequest(id))
    guard let http = response as? HTTPURLResponse, http.statusCode == 200 else { throw APIError.badStatus }
    return try JSONDecoder().decode(User.self, from: data)
}
```

### Structured Concurrency with TaskGroup

```swift
// async let for a fixed number of concurrent tasks
func fetchDashboard(userId: UUID) async throws -> Dashboard {
    async let profile = fetchUser(id: userId)
    async let posts = fetchPosts(userId: userId)
    async let notifications = fetchNotifications(userId: userId)
    return try await Dashboard(profile: profile, posts: posts, notifications: notifications)
}

// TaskGroup for dynamic concurrency
func fetchAllThumbnails(ids: [UUID]) async throws -> [UUID: UIImage] {
    try await withThrowingTaskGroup(of: (UUID, UIImage).self) { group in
        for id in ids { group.addTask { (id, try await ImageLoader.load(id: id)) } }
        var results: [UUID: UIImage] = [:]
        for try await (id, image) in group { results[id] = image }
        return results
    }
}
```

### Actors and Sendable

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]
    private var inFlight: [URL: Task<UIImage, Error>] = [:]

    func image(for url: URL) async throws -> UIImage {
        if let cached = cache[url] { return cached }
        if let existing = inFlight[url] { return try await existing.value }

        let task = Task {
            let (data, _) = try await URLSession.shared.data(from: url)
            guard let image = UIImage(data: data) else { throw ImageError.decodeFailed }
            return image
        }
        inFlight[url] = task
        let image = try await task.value
        cache[url] = image
        inFlight[url] = nil
        return image
    }
}

// Sendable: value types conform by default; reference types need explicit conformance
struct APIConfig: Sendable { let baseURL: URL; let apiKey: String }

// @unchecked Sendable only when you manually guarantee thread safety (e.g., locks)
final class AtomicCounter: @unchecked Sendable {
    private let lock = NSLock(); private var _value = 0
    func increment() { lock.withLock { _value += 1 } }
}
```

### MainActor and AsyncSequence

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published var user: User?
    @Published var isLoading = false

    func load(userId: UUID) async {
        isLoading = true
        defer { isLoading = false }
        user = try? await fetchUser(id: userId)
    }
}

// AsyncStream bridges callback-based APIs into structured concurrency
func monitorLocationUpdates() -> AsyncStream<CLLocation> {
    AsyncStream { continuation in
        let delegate = LocationDelegate(
            onUpdate: { continuation.yield($0) },
            onFinish: { continuation.finish() }
        )
        continuation.onTermination = { _ in delegate.stop() }
    }
}

// Consuming
func trackLocation() async {
    for await location in monitorLocationUpdates() {
        await updateMap(with: location)
    }
}
```

### Continuations for Bridging Legacy Code

```swift
func fetchUserBridged(id: UUID) async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetchUser(id: id) { result in continuation.resume(with: result) }
    }
}
// Every path must resume exactly once. Missing = hang. Double = crash.
```

**Anti-patterns to avoid:**
- Creating unstructured `Task {}` when `async let` or `TaskGroup` provides automatic cancellation.
- Resuming a continuation zero or more-than-once (deadlock or crash).
- Marking classes `@unchecked Sendable` without ensuring thread safety.
- Blocking an actor's executor with synchronous I/O -- starves other callers.

---

## SwiftUI

### View Composition and Modifiers

Build small, reusable views. Extract repeated modifier chains into custom modifiers.

```swift
struct ProfileCard: View {
    let user: User

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            AsyncImage(url: user.avatarURL) { $0.resizable().aspectRatio(contentMode: .fill) }
                placeholder: { ProgressView() }
                .frame(width: 80, height: 80).clipShape(Circle())
            Text(user.name).font(.headline)
            Text(user.bio).font(.subheadline).foregroundStyle(.secondary).lineLimit(2)
        }
        .padding().cardStyle()
    }
}

// Custom ViewModifier for reusable styling
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content.background(.regularMaterial)
            .clipShape(RoundedRectangle(cornerRadius: 16))
            .shadow(color: .black.opacity(0.1), radius: 8, y: 4)
    }
}
extension View { func cardStyle() -> some View { modifier(CardModifier()) } }
```

### State Management

```swift
// @State -- local state owned by the view
struct CounterView: View {
    @State private var count = 0
    var body: some View { Button("Count: \(count)") { count += 1 } }
}

// @Binding -- two-way reference to parent state
struct ToggleRow: View {
    let title: String
    @Binding var isOn: Bool
    var body: some View { Toggle(title, isOn: $isOn) }
}

// @Observable macro (iOS 17+) -- replaces ObservableObject
@Observable
final class SettingsModel {
    var theme: Theme = .system
    var notificationsEnabled = true
    var fontSize: CGFloat = 16
}

struct SettingsView: View {
    @State private var settings = SettingsModel()

    var body: some View {
        Form {
            Picker("Theme", selection: $settings.theme) {
                ForEach(Theme.allCases, id: \.self) { Text($0.rawValue) }
            }
            Toggle("Notifications", isOn: $settings.notificationsEnabled)
        }
    }
}

// @EnvironmentObject for shared state (pre-iOS 17, still common)
final class AuthManager: ObservableObject {
    @Published var isAuthenticated = false
    @Published var currentUser: User?
    func signOut() { isAuthenticated = false; currentUser = nil }
}

struct AccountView: View {
    @EnvironmentObject var auth: AuthManager
    var body: some View {
        if let user = auth.currentUser {
            Text("Hello, \(user.name)")
            Button("Sign Out") { auth.signOut() }
        }
    }
}
```

### Navigation and Animations

```swift
// Type-safe NavigationStack (iOS 16+)
enum AppRoute: Hashable { case userDetail(UUID), settings, postDetail(Post) }

struct RootView: View {
    @State private var path = NavigationPath()
    var body: some View {
        NavigationStack(path: $path) {
            UserListView()
                .navigationDestination(for: AppRoute.self) { route in
                    switch route {
                    case .userDetail(let id): UserDetailView(userId: id)
                    case .settings: SettingsView()
                    case .postDetail(let post): PostDetailView(post: post)
                    }
                }
        }
    }
}

// Matched geometry for hero transitions
struct AnimatedCard: View {
    @State private var isExpanded = false
    @Namespace private var ns
    var body: some View {
        Group { if isExpanded { ExpandedView() } else { CompactView() } }
            .matchedGeometryEffect(id: "card", in: ns)
            .onTapGesture { withAnimation(.spring(response: 0.4, dampingFraction: 0.8)) { isExpanded.toggle() } }
    }
}
```

**Anti-patterns to avoid:**
- Putting heavy logic in `body` -- extract into methods or view models.
- Using `@ObservedObject` to own state (use `@StateObject` or `@State` with `@Observable`).
- Overusing `AnyView` -- it defeats SwiftUI diffing. Prefer `@ViewBuilder`.

---

## Combine Framework

### Publishers and Subscribers

```swift
import Combine

class SearchViewModel: ObservableObject {
    @Published var query = ""
    @Published var results: [SearchResult] = []
    private var cancellables = Set<AnyCancellable>()

    init() {
        $query
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
            .removeDuplicates()
            .filter { !$0.isEmpty }
            .flatMap { query in
                SearchAPI.search(query: query).catch { _ in Just([]) }
            }
            .receive(on: DispatchQueue.main)
            .assign(to: &$results)
    }
}
```

### Combining Publishers

```swift
// Zip runs publishers in parallel and combines results
func loadProfile(userId: UUID) -> AnyPublisher<ProfileData, Error> {
    Publishers.Zip3(
        fetchUserPublisher(id: userId),
        fetchPostsPublisher(userId: userId),
        fetchStatsPublisher(userId: userId)
    )
    .map { ProfileData(user: $0, posts: $1, stats: $2) }
    .eraseToAnyPublisher()
}
```

### Error Handling in Combine

```swift
enum NetworkError: Error { case badURL, serverError(Int), decodingFailed, noConnection }

func fetchData<T: Decodable>(from endpoint: String) -> AnyPublisher<T, NetworkError> {
    guard let url = URL(string: endpoint) else { return Fail(error: .badURL).eraseToAnyPublisher() }
    return URLSession.shared.dataTaskPublisher(for: url)
        .mapError { _ in NetworkError.noConnection }
        .flatMap { data, response -> AnyPublisher<T, NetworkError> in
            guard let http = response as? HTTPURLResponse, (200...299).contains(http.statusCode)
            else { return Fail(error: .serverError(0)).eraseToAnyPublisher() }
            return Just(data).decode(type: T.self, decoder: JSONDecoder())
                .mapError { _ in .decodingFailed }.eraseToAnyPublisher()
        }.eraseToAnyPublisher()
}
```

**Anti-patterns to avoid:**
- Forgetting to store `AnyCancellable` -- subscription cancels immediately.
- Using `sink` without `[weak self]` -- retain cycles. Prefer `assign(to: &$property)`.
- Mixing Combine and async/await in the same layer -- pick one per boundary.

---

## UIKit Interop

### UIViewRepresentable

Wrap UIKit views for SwiftUI. The Coordinator handles delegate callbacks.

```swift
struct MapView: UIViewRepresentable {
    @Binding var region: MKCoordinateRegion
    var annotations: [MKPointAnnotation]

    func makeUIView(context: Context) -> MKMapView {
        let map = MKMapView(); map.delegate = context.coordinator; return map
    }
    func updateUIView(_ map: MKMapView, context: Context) {
        map.setRegion(region, animated: true)
        map.removeAnnotations(map.annotations); map.addAnnotations(annotations)
    }
    func makeCoordinator() -> Coordinator { Coordinator(self) }

    class Coordinator: NSObject, MKMapViewDelegate {
        let parent: MapView
        init(_ parent: MapView) { self.parent = parent }
        func mapView(_ mapView: MKMapView, regionDidChangeAnimated: Bool) {
            parent.region = mapView.region
        }
    }
}
```

### UIViewControllerRepresentable and Hosting

```swift
struct DocumentPicker: UIViewControllerRepresentable {
    @Binding var selectedURL: URL?
    func makeUIViewController(context: Context) -> UIDocumentPickerViewController {
        let picker = UIDocumentPickerViewController(forOpeningContentTypes: [.pdf, .plainText])
        picker.delegate = context.coordinator; return picker
    }
    func updateUIViewController(_ vc: UIDocumentPickerViewController, context: Context) {}
    func makeCoordinator() -> Coordinator { Coordinator(self) }

    class Coordinator: NSObject, UIDocumentPickerDelegate {
        let parent: DocumentPicker
        init(_ parent: DocumentPicker) { self.parent = parent }
        func documentPicker(_ c: UIDocumentPickerViewController, didPickDocumentsAt urls: [URL]) {
            parent.selectedURL = urls.first
        }
    }
}

// Hosting SwiftUI in UIKit
let hosting = UIHostingController(rootView: MySwiftUIView())
navigationController?.pushViewController(hosting, animated: true)
```

**Anti-patterns to avoid:**
- Updating `@Binding` inside `updateUIView` -- causes infinite update loops.
- Creating new delegates in `updateUIView` instead of reusing the coordinator.

---

## Data Persistence

### SwiftData (iOS 17+)

```swift
import SwiftData

@Model
final class Expense {
    var title: String
    var amount: Decimal
    var date: Date
    @Relationship(deleteRule: .cascade, inverse: \Receipt.expense)
    var receipts: [Receipt]

    init(title: String, amount: Decimal, date: Date = .now) {
        self.title = title; self.amount = amount; self.date = date; self.receipts = []
    }
}

@Model
final class Receipt {
    var imageData: Data
    var expense: Expense?
    init(imageData: Data) { self.imageData = imageData }
}

// App-level container
@main
struct ExpenseApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
            .modelContainer(for: [Expense.self, Receipt.self])
    }
}

// Query in views
struct ExpenseListView: View {
    @Query(sort: \Expense.date, order: .reverse) private var expenses: [Expense]
    @Environment(\.modelContext) private var context

    var body: some View { List(expenses) { ExpenseRow(expense: $0) } }

    func add(title: String, amount: Decimal) { context.insert(Expense(title: title, amount: amount)) }
    func delete(_ expense: Expense) { context.delete(expense) }
}
```

### Core Data (pre-iOS 17)

```swift
class CoreDataStack {
    static let shared = CoreDataStack()
    let container: NSPersistentContainer
    private init() {
        container = NSPersistentContainer(name: "AppModel")
        container.loadPersistentStores { _, error in if let error { fatalError("Core Data: \(error)") } }
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
}
```

### Keychain and UserDefaults

```swift
// Type-safe UserDefaults property wrapper
@propertyWrapper
struct UserDefault<T> {
    let key: String; let defaultValue: T
    var wrappedValue: T {
        get { UserDefaults.standard.object(forKey: key) as? T ?? defaultValue }
        set { UserDefaults.standard.set(newValue, forKey: key) }
    }
}

// Keychain for sensitive data (tokens, passwords)
struct KeychainHelper {
    static func save(_ data: Data, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key, kSecValueData as String: data]
        SecItemDelete(query as CFDictionary)
        guard SecItemAdd(query as CFDictionary, nil) == errSecSuccess else { throw KeychainError.saveFailed }
    }

    static func load(for key: String) throws -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword, kSecAttrAccount as String: key,
            kSecReturnData as String: true, kSecMatchLimit as String: kSecMatchLimitOne]
        var result: AnyObject?
        let status = SecCopyMatching(query as CFDictionary, &result)
        if status == errSecItemNotFound { return nil }
        guard status == errSecSuccess else { throw KeychainError.loadFailed }
        return result as? Data
    }
}
```

**Anti-patterns to avoid:**
- Storing tokens/passwords in UserDefaults -- use Keychain.
- Core Data fetches on main thread with large results -- use background contexts.
- Missing `automaticallyMergesChangesFromParent` -- UI context misses background saves.

---

## App Architecture

### MVVM with SwiftUI

```swift
// ViewModel
@Observable
final class ArticleListViewModel {
    private(set) var articles: [Article] = []
    private(set) var isLoading = false
    var errorMessage: String?
    private let repository: ArticleRepository

    init(repository: ArticleRepository = .live) { self.repository = repository }

    func loadArticles() async {
        isLoading = true; defer { isLoading = false }
        do { articles = try await repository.fetchAll() }
        catch { errorMessage = error.localizedDescription }
    }
}

// View
struct ArticleListView: View {
    @State private var viewModel = ArticleListViewModel()

    var body: some View {
        List(viewModel.articles) { article in
            ArticleRow(article: article)
        }
        .overlay { if viewModel.isLoading { ProgressView() } }
        .task { await viewModel.loadArticles() }
        .refreshable { await viewModel.loadArticles() }
    }
}
```

### Dependency Injection

```swift
protocol ArticleRepository {
    func fetchAll() async throws -> [Article]
    func delete(id: UUID) async throws
}

struct LiveArticleRepository: ArticleRepository {
    func fetchAll() async throws -> [Article] { try await APIClient.shared.get("/articles") }
    func delete(id: UUID) async throws { try await APIClient.shared.delete("/articles/\(id)") }
}
struct MockArticleRepository: ArticleRepository {
    func fetchAll() async throws -> [Article] { Article.samples }
    func delete(id: UUID) async throws {}
}

// Static convenience for clean call sites
extension ArticleRepository where Self == LiveArticleRepository { static var live: Self { .init() } }
extension ArticleRepository where Self == MockArticleRepository { static var mock: Self { .init() } }

// SwiftUI Environment injection
private struct ArticleRepoKey: EnvironmentKey {
    static let defaultValue: any ArticleRepository = LiveArticleRepository()
}
extension EnvironmentValues {
    var articleRepository: any ArticleRepository {
        get { self[ArticleRepoKey.self] } set { self[ArticleRepoKey.self] = newValue }
    }
}
```

### The Composable Architecture (TCA)

```swift
import ComposableArchitecture

@Reducer
struct CounterFeature {
    @ObservableState struct State: Equatable { var count = 0 }
    enum Action { case increment, decrement }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .increment: state.count += 1; return .none
            case .decrement: state.count -= 1; return .none
            }
        }
    }
}
// TCA gives you: testable reducers, dependency injection, exhaustive action testing
```

### Navigation Router and Deep Linking

```swift
@Observable
final class AppRouter {
    var path = NavigationPath()
    func navigate(to route: AppRoute) { path.append(route) }
    func popToRoot() { path = NavigationPath() }

    func handleDeepLink(_ url: URL) {
        guard let host = URLComponents(url: url, resolvingAgainstBaseURL: false)?.host else { return }
        switch host {
        case "user": if let id = url.queryParam("id").flatMap(UUID.init) { navigate(to: .userDetail(id)) }
        case "settings": navigate(to: .settings)
        default: break
        }
    }
}
```

**Anti-patterns to avoid:**
- Putting networking logic directly in views -- go through a view model or reducer.
- Singletons everywhere instead of dependency injection -- makes testing impossible.
- Force-unwrapping optionals in view models -- surface errors gracefully.

---

## App Store and Distribution

### Signing, TestFlight, and Distribution

```bash
# Archive, export, and upload in CI
xcodebuild archive -workspace App.xcworkspace -scheme App \
  -archivePath build/App.xcarchive -destination "generic/platform=iOS" \
  CODE_SIGN_IDENTITY="Apple Distribution" PROVISIONING_PROFILE_SPECIFIER="App Store Profile"

xcodebuild -exportArchive -archivePath build/App.xcarchive \
  -exportPath build/ -exportOptionsPlist ExportOptions.plist

xcrun altool --upload-app -f build/App.ipa -t ios \
  --apiKey "$API_KEY_ID" --apiIssuer "$ISSUER_ID"
```

```swift
// App Store Connect API -- JWT auth with ES256
struct AppStoreConnectClient {
    let keyId: String; let issuerId: String; let privateKey: P256.Signing.PrivateKey

    func listBuilds(appId: String) async throws -> [Build] {
        let token = try generateJWT(iss: issuerId, kid: keyId, key: privateKey)
        var req = URLRequest(url: URL(string:
            "https://api.appstoreconnect.apple.com/v1/apps/\(appId)/builds")!)
        req.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        let (data, _) = try await URLSession.shared.data(for: req)
        return try JSONDecoder().decode(BuildsResponse.self, from: data).data
    }
}
```

### Xcode Cloud CI/CD

```bash
# ci_scripts/ci_post_clone.sh -- resolve deps and set build number
swift package resolve && agvtool new-version -all "$CI_BUILD_NUMBER"

# ci_scripts/ci_pre_xcodebuild.sh -- inject secrets from environment
cat > "$CI_PRIMARY_REPOSITORY_PATH/App/Secrets.swift" <<EOF
enum Secrets { static let apiKey = "$API_KEY" }
EOF
```

### App Review Guidelines Checklist

```
- [ ] Privacy manifest (PrivacyInfo.xcprivacy) declares all required-reason API usage
- [ ] Sign in with Apple offered when third-party login exists
- [ ] In-app purchases use StoreKit only -- no external payment links
- [ ] Data collection disclosed in App Store Connect privacy labels
- [ ] No private API usage (automatic rejection)
- [ ] App Tracking Transparency prompt if IDFA is used
- [ ] Background modes justified in review notes
```

### StoreKit 2 In-App Purchases

```swift
import StoreKit

actor StoreManager {
    private var purchasedIDs = Set<String>()

    func purchase(_ product: Product) async throws -> Transaction? {
        let result = try await product.purchase()
        guard case .success(let verification) = result else { return nil }
        let txn = try checkVerified(verification)
        await txn.finish(); purchasedIDs.insert(product.id)
        return txn
    }

    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        guard case .verified(let item) = result else { throw StoreError.verificationFailed }
        return item
    }

    // MUST listen for transaction updates to catch restores and deferred purchases
    func listenForTransactions() async {
        for await result in Transaction.updates {
            if let txn = try? checkVerified(result) { purchasedIDs.insert(txn.productID); await txn.finish() }
        }
    }
}
```

**Anti-patterns to avoid:**
- Skipping the privacy manifest -- Apple rejects apps using required-reason APIs without it.
- Not listening for `Transaction.updates` -- misses restored or deferred purchases.
- Uploading builds without incrementing build number -- App Store Connect rejects duplicates.
- Using private APIs or UIWebView -- automatic rejection.

# Local Data Persistence & Caching in iOS  
### FileManager, SQLite/SwiftData, and Cache Invalidation

Local persistence is the backbone of **offline behavior, fast startup, and resilient UX**. This article focuses on:

- When and why to cache
- File-based caching with `FileManager`
- Structured persistence with SQLite / SwiftData
- Cache invalidation strategies
- Boilerplate code patterns

---

## 1. Caching Goals & Constraints

Before writing any code, clarify:

- **What** are we caching? (events, images, tokens?)
- **How fresh** must it be?
- **How big** can it get?
- **What happens** when data is missing, stale, or corrupted?

Common caching goals:

- Faster startup (load cache, then refresh)
- Reduced network usage
- Offline fallback

---

## 2. File-Based Caching with FileManager

Ideal for:

- JSON blobs
- Simple key → data mappings
- Images and binary blobs

---

### 2.1 Boilerplate: Cache Directory

```swift
enum CacheDirectory {
    static var url: URL {
        let paths = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)
        return paths[0].appendingPathComponent("EventCache", isDirectory: true)
    }

    static func ensureExists() throws {
        try FileManager.default.createDirectory(at: url, withIntermediateDirectories: true)
    }
}
```

---

### 2.2 Saving a JSON Blob

```swift
struct EventCache: Codable {
    let version: Int
    let timestamp: Date
    let events: [CachedEvent]
}

struct CachedEvent: Codable {
    let id: String
    let title: String
    let date: Date
}

func saveEventsToDisk(_ events: [CachedEvent]) throws {
    try CacheDirectory.ensureExists()
    let cache = EventCache(version: 1, timestamp: Date(), events: events)
    let data = try JSONEncoder().encode(cache)

    let fileURL = CacheDirectory.url.appendingPathComponent("events.json")
    let tmpURL = fileURL.appendingPathExtension("tmp")

    try data.write(to: tmpURL, options: .atomic)
    try FileManager.default.replaceItemAt(fileURL, withItemAt: tmpURL)
}
```

Notes:

- Write to a temporary file, then atomically replace.
- Store version + timestamp for invalidation.

---

### 2.3 Loading and Validating the Cache

```swift
enum EventCacheError: Error {
    case notFound
    case corrupted
    case expired
}

func loadEventsFromDisk(maxAge: TimeInterval = 60 * 60) throws -> [CachedEvent] {
    let fileURL = CacheDirectory.url.appendingPathComponent("events.json")
    guard FileManager.default.fileExists(atPath: fileURL.path) else {
        throw EventCacheError.notFound
    }

    let data = try Data(contentsOf: fileURL)
    let cache = try JSONDecoder().decode(EventCache.self, from: data)

    guard Date().timeIntervalSince(cache.timestamp) <= maxAge else {
        throw EventCacheError.expired
    }

    return cache.events
}
```

Corruption or decoding failure should be treated as **non-fatal**—delete the file and refetch.

---

## 3. SQLite / SwiftData for Structured Caching

For more complex apps, you may want:

- Querying
- Relationships
- Incremental updates

Two common approaches:

- **Raw SQLite** / GRDB
- **SwiftData** (Apple’s modern persistence)

---

## 3.1 SwiftData Boilerplate (iOS 17+)

### Model

```swift
import SwiftData

@Model
final class EventEntity {
    @Attribute(.unique) var id: String
    var title: String
    var date: Date
    var updatedAt: Date

    init(id: String, title: String, date: Date, updatedAt: Date = .now) {
        self.id = id
        self.title = title
        self.date = date
        self.updatedAt = updatedAt
    }
}
```

### Container Setup

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            EventListView()
        }
        .modelContainer(for: EventEntity.self)
    }
}
```

SwiftData handles SQLite setup behind the scenes.

---

### 3.2 Inserting / Updating Cached Events

```swift
@MainActor
final class EventStore {
    @Environment(\.modelContext) private var context

    func upsert(events: [EventDTO]) throws {
        for dto in events {
            let descriptor = FetchDescriptor<EventEntity>(
                predicate: #Predicate { $0.id == dto.id }
            )
            if let existing = try? context.fetch(descriptor).first {
                existing.title = dto.title
                existing.date = dto.date
                existing.updatedAt = .now
            } else {
                let entity = EventEntity(id: dto.id, title: dto.title, date: dto.date)
                context.insert(entity)
            }
        }
        try context.save()
    }

    func fetchAll() throws -> [EventEntity] {
        try context.fetch(FetchDescriptor<EventEntity>())
    }
}
```

---

## 4. Cache Invalidation Strategies

The hardest part of caching is knowing **when not to trust it**.

Common strategies:

1. **Time-based (TTL)**
2. **Version-based**
3. **Event-based**
4. **Size-based (LRU, quotas)**

You can mix strategies.

---

### 4.1 Time-Based (TTL)

Useful for data that becomes stale quickly (e.g., events, news).

```swift
func isCacheFresh(timestamp: Date, maxAge: TimeInterval) -> Bool {
    Date().timeIntervalSince(timestamp) <= maxAge
}
```

Use TTL to decide:

- Use cache only
- Use cache then refresh in background
- Ignore cache, fetch fresh

---

### 4.2 Version-Based

When backend schema or semantics change:

- Increment a `cacheVersion`.
- If on-disk version != app version → delete or migrate.

```swift
let currentCacheVersion = 2

func validateVersion(_ storedVersion: Int) throws {
    guard storedVersion == currentCacheVersion else {
        throw EventCacheError.corrupted
    }
}
```

---

### 4.3 Event-Based

Invalidate cache when:

- User logs out
- Feature flags change
- User switches organizations or environments

Example:

```swift
func handleLogout() {
    try? FileManager.default.removeItem(at: CacheDirectory.url)
}
```

---

### 4.4 Size-Based (LRU-ish)

For image caches or large data:

- Track cache directory size.
- Evict oldest files when exceeding threshold.

```swift
func enforceSizeLimit(limitInBytes: Int) {
    let fm = FileManager.default
    let urls = (try? fm.contentsOfDirectory(at: CacheDirectory.url, includingPropertiesForKeys: [.contentModificationDateKey, .fileSizeKey])) ?? []

    let filesWithInfo = urls.compactMap { url -> (URL, Date, Int)? in
        guard
            let attrs = try? url.resourceValues(forKeys: [.contentModificationDateKey, .fileSizeKey]),
            let date = attrs.contentModificationDate,
            let size = attrs.fileSize
        else { return nil }
        return (url, date, size)
    }

    let totalSize = filesWithInfo.reduce(0) { $0 + $1.2 }
    guard totalSize > limitInBytes else { return }

    var bytesToFree = totalSize - limitInBytes
    for (url, _, size) in filesWithInfo.sorted(by: { $0.1 < $1.1 }) {
        try? fm.removeItem(at: url)
        bytesToFree -= size
        if bytesToFree <= 0 { break }
    }
}
```

---

## 5. Combining Cache + Network (Repository Pattern)

A typical flow:

```swift
enum DataSource {
    case cache
    case network
    case cacheThenNetwork
}

protocol EventRepository {
    func loadEvents() async throws -> [Event]
}

struct DefaultEventRepository: EventRepository {
    let api: EventAPI
    let cache: EventCacheStore

    func loadEvents() async throws -> [Event] {
        // 1. Try cache
        if let cached = try? cache.loadIfFresh() {
            // 2. Kick off background refresh but do not block UI
            Task {
                if let fresh = try? await api.fetchEvents() {
                    try? cache.save(fresh)
                }
            }
            return cached
        }

        // 3. No cache or stale → go to network
        let fresh = try await api.fetchEvents()
        try? cache.save(fresh)
        return fresh
    }
}
```

This pattern:

- Keeps UI fast
- Keeps cache reasonably fresh
- Avoids blocking UX on the network

---

## 6. Best Practices Checklist

- ✅ Pick the **simplest persistence** approach that meets requirements.
- ✅ For simple blobs: use **FileManager** with atomic writes.
- ✅ For complex relational data: use **SwiftData / SQLite**.
- ✅ Always store **timestamps** and **version info**.
- ✅ Design **cache invalidation** up front (TTL, events, version).
- ✅ Handle corruption by deleting and refetching, not crashing.
- ✅ Keep caches in the **Caches** directory, not Documents (OS may clear it).
- ✅ Separate **domain models** from cache representations (DTOs).
- ✅ Expose caching via a **repository** so UI doesn’t know where data comes from.
- ✅ Log cache hits/misses to validate that caching is actually helping.

Smart local persistence makes apps feel instant and trustworthy.  
Done well, it disappears into the background—users simply experience an app that “just works,” regardless of network conditions.


# Error Handling & Reliability in iOS (Swift Concurrency Edition)

Reliable apps treat failures as **first-class citizens**, not afterthoughts. Thoughtful error handling leads to:

- More predictable behavior  
- Easier debugging  
- Better user trust  
- Cleaner API boundaries  

This article focuses on modern Swift (async/await, actors) and covers:

- Error taxonomies
- Retry vs. fail-fast
- Handling partial results
- Alerting vs. silent recovery
- Graceful degradation on slow networks
- Data corruption & version migration
- Boilerplate enums and patterns

---

## 1. Designing an Error Taxonomy

### 1.1 Domain-specific error enums

Prefer domain-focused errors over generic ones:

```swift
enum NetworkError: Error {
    case offline
    case timedOut
    case badStatus(code: Int)
    case decodingFailed(underlying: Error)
    case cancelled
}

enum PersistenceError: Error {
    case notFound
    case corrupted
    case migrationNeeded
    case writeFailed(underlying: Error)
}
```

Benefits:

- Easier branching in higher layers
- Faster understanding in logs
- Cleaner mapping to UI responses

---

### 1.2 Convert low-level errors at module boundaries

```swift
func fetchEvents() async throws -> [Event] {
    do {
        let (data, response) = try await urlSession.data(for: request)
        return try decodeEvents(data, response: response)
    } catch is URLError {
        throw NetworkError.offline
    } catch DecodingError.dataCorrupted(let context) {
        throw NetworkError.decodingFailed(underlying: context.underlyingError ?? error)
    } catch {
        throw NetworkError.badStatus(code: (response as? HTTPURLResponse)?.statusCode ?? -1)
    }
}
```

Outside callers never deal with `URLError` or raw `DecodingError`—only domain-level errors.

---

## 2. Retry vs. Fail-Fast

### 2.1 When to retry

Retry **only** for transient conditions:

- Network timeouts
- 5xx server errors
- Rate limit responses (with backoff)
- Certain `URLError` cases (e.g., networkConnectionLost)

Never retry:

- 4xx client errors (400, 401, 403, 404)
- Validation failures
- Permanent configuration problems

---

### 2.2 Retry helper

```swift
struct RetryPolicy {
    let maxAttempts: Int
    let initialDelay: Duration
    let multiplier: Double
}

func retrying<T>(
    policy: RetryPolicy = .init(maxAttempts: 3, initialDelay: .seconds(0.5), multiplier: 2.0),
    operation: @escaping () async throws -> T
) async throws -> T {
    var attempt = 0
    var delay = policy.initialDelay

    while true {
        attempt += 1
        do {
            return try await operation()
        } catch is CancellationError {
            throw error
        } catch {
            guard attempt < policy.maxAttempts else { throw error }
            try await Task.sleep(for: delay)
            delay = .seconds(Double(delay.components.seconds) * policy.multiplier)
        }
    }
}
```

Callers decide *when* to use this; the core APIs stay simple.

---

## 3. Handling Partial Results

Example: fetching multiple pages, or a batch of image downloads.

### 3.1 Returning partial success

```swift
struct PartialResult<Value> {
    let items: [Value]
    let errors: [Error]
    var isComplete: Bool { errors.isEmpty }
}
```

Usage:

```swift
func loadThumbnails(for ids: [ImageID]) async -> PartialResult<UIImage> {
    var images: [UIImage] = []
    var errors: [Error] = []

    await withTaskGroup(of: Result<UIImage, Error>.self) { group in
        for id in ids {
            group.addTask {
                do { return .success(try await fetchImage(id)) }
                catch { return .failure(error) }
            }
        }
        for await result in group {
            switch result {
            case .success(let img): images.append(img)
            case .failure(let err): errors.append(err)
            }
        }
    }
    return PartialResult(items: images, errors: errors)
}
```

The UI can then:

- Show successfully loaded images
- Display a small “Some items failed to load” banner

---

## 4. Alerting vs. Silent Recovery

### 4.1 When to alert

Alert (or otherwise visibly message) the user when:

- User action clearly failed (submit, payment, RSVP)
- Data loss may have occurred
- Security/permission issues arise (401, 403)
- Persistent failure (e.g., cannot sync after multiple attempts)

### 4.2 When to recover silently

Recover silently when:

- Background refresh fails (will try again later)
- Non-essential analytics uploads fail
- An image falls back to a placeholder
- One of many parallel requests fails but others succeed

---

### 4.3 UI pattern: message bar, not modal

Prefer inline banners over modal alerts for non-critical issues:

```swift
enum InfoBanner {
    case offline
    case limitedResults
    case syncDelayed
}
```

- Keep modals for critical, blocking problems
- Everything else → banner or inline message

---

## 5. Graceful Degradation on Slow Networks

### 5.1 Timeouts and loading states

- Set *reasonable* timeouts on requests.
- Show skeletons or placeholder content quickly.
- Allow the user to **cancel** long-running operations.

### 5.2 Progressive enhancement

On slow networks:

- Load text data first.
- Lazy-load images or secondary content.
- Consider reducing concurrent requests.

For example: load event list first, then fetch images in the background.

---

## 6. Data Corruption & Version Migration

### 6.1 Detecting corruption

Corruption may come from:

- Crashed writes
- Disk errors
- Migration bugs

Design your persistence layer to:

- Validate checksums, counts, or schema version
- Treat failures as `PersistenceError.corrupted`

```swift
func loadCache() throws -> EventCache {
    let data = try Data(contentsOf: url)
    guard validateChecksum(data) else {
        throw PersistenceError.corrupted
    }
    return try decoder.decode(EventCache.self, from: data)
}
```

If corrupted:

- Delete the bad cache
- Refetch from the network if possible
- Fall back with a meaningful message

---

### 6.2 Versioned schemas and migrations

Always store a schema version:

```swift
struct EventCacheV2: Codable {
    let version: Int
    let events: [CachedEvent]
}

enum CacheVersion: Int {
    case v1 = 1
    case v2 = 2
}
```

Migration entry point:

```swift
func loadAnyVersion() throws -> [Event] {
    let data = try Data(contentsOf: url)
    let version = try detectVersion(data)

    switch version {
    case .v1:
        let old = try decoder.decode(EventCacheV1.self, from: data)
        return migrateV1ToDomain(old)
    case .v2:
        let current = try decoder.decode(EventCacheV2.self, from: data)
        return current.events.map(toDomain)
    }
}
```

---

## 7. Interfacing with Errors at the ViewModel Layer

ViewModels should **translate raw errors into UI-friendly states**:

```swift
enum EventListViewState {
    case loading
    case loaded([Event])
    case offlineCached([Event])
    case error(message: String)
}

@Observable
@MainActor
final class EventListViewModel {
    var state: EventListViewState = .loading

    private let service: EventServiceProtocol

    init(service: EventServiceProtocol) {
        self.service = service
    }

    func load() async {
        do {
            let events = try await service.fetchEvents()
            state = .loaded(events)
        } catch NetworkError.offline {
            let cached = try? service.loadCachedEvents()
            if let cached {
                state = .offlineCached(cached)
            } else {
                state = .error(message: "You appear to be offline and we have no saved events yet.")
            }
        } catch {
            state = .error(message: "Something went wrong. Please try again.")
        }
    }
}
```

Key points:

- Domain errors mapped to state
- User sees a clear message
- Offline is treated separately from generic error

---

## 8. Industry Best Practices Checklist

- ✅ Create **domain-specific error enums** (network, persistence, domain logic).
- ✅ Convert low-level errors at module boundaries.
- ✅ Retry only for clearly transient errors; otherwise fail-fast.
- ✅ Support **partial results** instead of all-or-nothing when applicable.
- ✅ Use banners or inline messaging for non-critical issues; avoid alert spam.
- ✅ Design for **graceful degradation** under slow or flaky networks.
- ✅ Detect and handle data corruption explicitly.
- ✅ Version your on-disk format and plan migrations up-front.
- ✅ Keep ViewModels responsible for mapping errors → user-facing states.
- ✅ Log errors with enough context to debug, but without leaking sensitive data.

Robust error handling is ultimately about *predictability* and *trust*. Users forgive imperfections when the app explains itself clearly and behaves consistently under stress.


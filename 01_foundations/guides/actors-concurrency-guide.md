# Guide: Actors, Concurrency, Tasks, and Task Groups (Modern Swift)

This guide is a practical overview of Swift Concurrency building blocks you’ll use in real SwiftUI/iOS apps: `actor`, `Task`, `TaskGroup`, and how they play nicely with the **Observation framework** instead of `ObservableObject`.

Applies to Swift 5.9+ / Swift 6.

---

## 1. Structured Concurrency in One Paragraph

Swift Concurrency is built on:

- `async` / `await` for suspending work.  
- `Task` for units of asynchronous work.  
- `TaskGroup` for spawning children with a *structured* lifetime.  
- `actor` for isolating mutable state.  

Prefer **structured concurrency** (`async let`, `TaskGroup`) over ad-hoc threads or callbacks.

---

## 2. Actors: Isolating Mutable State

An `actor` is like a class that guarantees **mutual exclusion** for its mutable state.

```swift
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }

    func current() -> Int {
        value
    }
}
```

Usage:

```swift
let counter = Counter()

await counter.increment()
let value = await counter.current()
```

- Accessing actor-isolated state from outside the actor is `await`-ed.  
- Inside the actor, you access properties synchronously.

### When to use an actor

- Shared mutable state across many tasks (caches, stores, managers).  
- Avoiding explicit locks / mutexes.

> Note: `@Observable` cannot be applied to actor types. Use actors for concurrency isolation and `@Observable` classes for UI-facing observable models.

---

## 3. @MainActor and the Observation Framework

`@MainActor` ensures code runs on the main thread (UI work). With the Observation framework, your view models typically look like this:

```swift
import Observation

@MainActor
@Observable
final class CounterViewModel {
    var value: Int = 0

    func increment() {
        value += 1
    }

    func loadInitialValue() async {
        // Simulate async work
        try? await Task.sleep(for: .seconds(1))
        value = 42
    }
}
```

And in SwiftUI:

```swift
struct CounterView: View {
    @State private var model = CounterViewModel()

    var body: some View {
        VStack {
            Text("Value: \(model.value)")
            Button("Increment") {
                model.increment()
            }
        }
        .task {
            await model.loadInitialValue()
        }
    }
}
```

- No `ObservableObject`.  
- No `@Published`.  
- No `@ObservedObject` / `@StateObject` / `@EnvironmentObject`.  
- Just `@Observable` + `@State` and SwiftUI will track property reads.

Use `@Bindable` when you need two-way bindings to `@Observable` types:

```swift
struct EditView: View {
    @Bindable var model: CounterViewModel

    var body: some View {
        Stepper("Value", value: $model.value)
    }
}
```

---

## 4. Task: Structured vs Fire-and-Forget

### Structured use: inside async context

```swift
func loadDashboard() async throws {
    async let user = fetchUser()
    async let stats = fetchStats()

    let (userResult, statsResult) = try await (user, stats)
    // use results
}
```

This is **structured**: child tasks are tied to the parent’s lifetime.

### Fire-and-forget Task

```swift
Task {
    await analytics.track(event: "AppOpened")
}
```

Use sparingly. For fully separated work:

```swift
Task.detached(priority: .background) {
    await doHeavyBackgroundWork()
}
```

Detached tasks don’t inherit actor context; use them when that’s what you want.

---

## 5. Task Cancellation

Tasks are cooperative: they must check for cancellation.

```swift
func loadData() async throws {
    try Task.checkCancellation()

    let (data, _) = try await URLSession.shared.data(from: url)
    try Task.checkCancellation()

    // decode, etc.
}
```

Cancelling:

```swift
let task = Task {
    try await loadData()
}

// later
task.cancel()
```

Patterns:

- Use `Task.checkCancellation()` in loops and long-running work.  
- Handle `CancellationError` separately when needed.

---

## 6. TaskGroup: Parallel Work with Structured Lifetime

Use `withTaskGroup` to launch multiple child tasks and await them as a group.

```swift
func fetchAllUsers(ids: [Int]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User?.self) { group in
        for id in ids {
            group.addTask {
                try? await fetchUser(id: id)
            }
        }

        var results: [User] = []

        for try await user in group {
            if let user {
                results.append(user)
            }
        }

        return results
    }
}
```

- If any child throws, the group cancels the rest.  
- Use `withTaskGroup` (non-throwing) when children can’t throw.

---

## 7. Using Actors + TaskGroup Together

Example: concurrent prefetch while updating a shared cache actor.

```swift
actor ImageCache {
    private var storage: [URL: Data] = [:]

    func imageData(for url: URL) -> Data? {
        storage[url]
    }

    func setImageData(_ data: Data, for url: URL) {
        storage[url] = data
    }
}

let cache = ImageCache()

func prefetchImages(urls: [URL]) async {
    await withTaskGroup(of: Void.self) { group in
        for url in urls {
            group.addTask {
                if let existing = await cache.imageData(for: url) {
                    // already cached
                    return
                }

                do {
                    let (data, _) = try await URLSession.shared.data(from: url)
                    await cache.setImageData(data, for: url)
                } catch {
                    // handle or log
                }
            }
        }
    }
}
```

---

## 8. Actor Reentrancy Basics

Actors can be **reentrant**: while one `await` is in progress inside an actor method, other calls may interleave.

```swift
actor Counter {
    private var value = 0

    func incrementSlowly() async {
        let old = value
        try? await Task.sleep(for: .seconds(1))
        value = old + 1
    }
}
```

Between reading and writing `value`, other methods might run.  

Rule of thumb:

- Keep actor methods small.  
- Avoid holding complex invariants across `await`s.  
- If needed, capture state into locals before awaiting or split logic into smaller methods.

---

## 9. Global Actors

You can define your own global actor for shared contexts (e.g., a “DatabaseActor”).

```swift
@globalActor
enum DatabaseActor {
    actor Actor {}
    static let shared = Actor()
}

@DatabaseActor
func performDatabaseWork() async {
    // Serialized on DatabaseActor
}
```

Global actors pair well with Swift 6’s stricter concurrency checking and help document intent.

---

## 10. Concurrency and Value Types

Structs and enums are naturally safer *if* you avoid sharing mutable references.

Modern rules of thumb:

- Prefer **value types** for domain data.  
- Use **actors** to protect shared mutable reference state.  
- Use `@Observable` classes for UI-facing state observed by SwiftUI.  
- Mark long-lived data models and value types `Sendable` where appropriate.

---

## 11. Quick Checklist

- [ ] Use `actor` for shared mutable state (stores, caches, managers).  
- [ ] Use `@Observable` + `@State` (and `@Bindable`) for UI-facing models instead of `ObservableObject`.  
- [ ] Prefer structured concurrency (`async let`, `TaskGroup`) over ad-hoc tasks.  
- [ ] Be cancellation-friendly in long-running work.  
- [ ] Keep actor methods small and avoid long awaits with invariants mid-way.  
- [ ] Use global actors (`@MainActor`, custom actors) to document threading guarantees.

---

## 🤖 Tooling Note

Portions of drafting and editorial refinement in this repository were accelerated using large language models (including ChatGPT, Claude, and Gemini) under direct human design, validation, and final approval. All technical decisions, code, and architectural conclusions are authored and verified by the repository maintainer.

# Guide: AsyncStream Basics (with Continuation)

`AsyncStream` and `AsyncThrowingStream` let you bridge callback or delegate APIs into Swift’s `async`/`await` world by creating your own `AsyncSequence`. This integrates nicely with modern Swift Concurrency and the Observation framework.

---

## 1. AsyncStream in One Sentence

`AsyncStream<Element>` is an `AsyncSequence` whose elements you **push** from a producer and **iterate** over as an async consumer.

```swift
let stream = AsyncStream<Int> { continuation in
    continuation.yield(1)
    continuation.yield(2)
    continuation.finish()
}

for await value in stream {
    print(value)
}
```

---

## 2. Anatomy of AsyncStream

Creating a stream:

```swift
let stream = AsyncStream<Int> { continuation in
    // Producer side: hold onto `continuation`,
    // and call `yield` or `finish` later.
}
```

Key APIs on `continuation`:

- `yield(_ element: Element)` → push a value.  
- `finish()` → finish the sequence normally.  
- `onTermination = { @Sendable (Termination) in ... }` → cleanup callback when consumer stops.

---

## 3. Simple Timer Example

```swift
func tickStream(every interval: Duration) -> AsyncStream<Date> {
    AsyncStream { continuation in
        let task = Task {
            while !Task.isCancelled {
                continuation.yield(Date())
                try? await Task.sleep(for: interval)
            }
            continuation.finish()
        }

        continuation.onTermination = { @Sendable _ in
            task.cancel()
        }
    }
}
```

Usage:

```swift
for await date in tickStream(every: .seconds(1)) {
    print("Tick at \(date)")
}
```

---

## 4. AsyncThrowingStream

Use when your sequence can fail with an error.

```swift
enum StreamError: Error {
    case failed
}

func throwingStream() -> AsyncThrowingStream<Int, Error> {
    AsyncThrowingStream { continuation in
        continuation.yield(1)
        continuation.yield(2)
        continuation.finish(throwing: StreamError.failed)
    }
}
```

Consumption:

```swift
do {
    for try await value in throwingStream() {
        print(value)
    }
} catch {
    print("Stream error: \(error)")
}
```

Producer side APIs:

- `yield(_ element: Element)`  
- `finish()`  
- `finish(throwing: Failure)`

---

## 5. Bridging Delegate-Based APIs

Example: bridging a generic delegate-style source into `AsyncStream`.

```swift
final class EventsSource {
    var onEvent: ((String) -> Void)?

    func simulateEvent(_ value: String) {
        onEvent?(value)
    }
}
```

Bridge:

```swift
func events(from source: EventsSource) -> AsyncStream[String] {
    AsyncStream { continuation in
        source.onEvent = { value in
            continuation.yield(value)
        }

        continuation.onTermination = { @Sendable _ in
            source.onEvent = nil
        }
    }
}
```

Usage:

```swift
let source = EventsSource()

Task {
    for await event in events(from: source) {
        print("Received event: \(event)")
    }
}

source.simulateEvent("Hello")
source.simulateEvent("World")
```

---

## 6. Bridging URLSession WebSockets

Given a `URLSessionWebSocketTask`, you can expose an `AsyncThrowingStream` of messages:

```swift
func webSocketMessages(
    for task: URLSessionWebSocketTask
) -> AsyncThrowingStream<URLSessionWebSocketTask.Message, Error> {
    AsyncThrowingStream { continuation in
        func receiveNext() {
            task.receive { result in
                switch result {
                case .failure(let error):
                    continuation.finish(throwing: error)
                case .success(let message):
                    continuation.yield(message)
                    receiveNext()
                }
            }
        }

        receiveNext()

        continuation.onTermination = { @Sendable _ in
            task.cancel(with: .normalClosure, reason: nil)
        }
    }
}
```

Then:

```swift
for try await message in webSocketMessages(for: task) {
    // handle message
}
```

---

## 7. Backpressure and Buffering

`AsyncStream` initializer lets you configure buffering:

```swift
AsyncStream<Int>(
    bufferingPolicy: .bufferingNewest(10)
) { continuation in
    // producer
}
```

Policies:

- `.unbounded` (default) — store all elements until consumed.  
- `.bufferingOldest(Int)` — bounded buffer; oldest elements dropped when full.  
- `.bufferingNewest(Int)` — bounded buffer; newest elements dropped when full.

Use bounded buffers for high-frequency streams (sensor data, logs).

---

## 8. Termination Semantics

Termination happens when:

- Producer calls `finish()` or `finish(throwing:)`.  
- Consumer breaks out of the loop or finishes.  
- The consuming task is cancelled.

Use `onTermination` to clean up resources (cancel tasks, release delegates, close sockets).

```swift
continuation.onTermination = { @Sendable termination in
    switch termination {
    case .finished:
        print("Stream finished")
    case .cancelled:
        print("Stream cancelled")
    }
}
```

---

## 9. AsyncStream vs Observation

- Use **Observation** (`@Observable`, `@State`, `@Bindable`) for UI-related state that SwiftUI should track automatically.  
- Use **AsyncStream** for *event streams* and *asynchronous sequences* (notifications, network events, timers, WebSockets).

You can combine them: an `@Observable` view model can expose an `AsyncStream` for low-level consumers while also updating its own properties for the UI.

---

## 10. Quick Checklist

- [ ] Use `AsyncStream` for custom `AsyncSequence`s.  
- [ ] Use `AsyncThrowingStream` when errors are possible.  
- [ ] Configure buffering for high-frequency streams.  
- [ ] Ensure you call `finish()` / `finish(throwing:)` when done producing.  
- [ ] Use `onTermination` to cancel underlying work and release resources.

---

## 🤖 Tooling Note

Portions of drafting and editorial refinement in this repository were accelerated using large language models (including ChatGPT, Claude, and Gemini) under direct human design, validation, and final approval. All technical decisions, code, and architectural conclusions are authored and verified by the repository maintainer.


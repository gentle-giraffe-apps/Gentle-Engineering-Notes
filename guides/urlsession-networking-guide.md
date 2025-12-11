# Guide: URLSession, HTTPS Requests, and WebSockets (Modern Swift)

This guide covers practical patterns for using `URLSession` for HTTPS requests and WebSockets in modern Swift with async/await (Swift 5.9+ / Swift 6).

---

## 1. Basic HTTPS Request with async/await

```swift
struct UserDTO: Codable, Sendable {
    let id: Int
    let name: String
}

enum APIError: Error {
    case invalidResponse
    case statusCode(Int)
}
```

```swift
struct APIClient: Sendable {
    let baseURL = URL(string: "https://api.example.com")!
    let session: URLSession = .shared

    func fetchUser(id: Int) async throws -> UserDTO {
        let url = baseURL.appendingPathComponent("users/\(id)")
        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.setValue("application/json", forHTTPHeaderField: "Accept")

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }

        guard (200..<300).contains(httpResponse.statusCode) else {
            throw APIError.statusCode(httpResponse.statusCode)
        }

        return try JSONDecoder().decode(UserDTO.self, from: data)
    }
}
```

Key points:

- Use `URLSession.data(for:)` with async/await.  
- Always validate status codes.  
- Decode into DTOs, not domain models.

---

## 2. POST with JSON Body

```swift
struct LoginRequestDTO: Codable, Sendable {
    let email: String
    let password: String
}

struct LoginResponseDTO: Codable, Sendable {
    let token: String
}
```

```swift
extension APIClient {
    func login(email: String, password: String) async throws -> LoginResponseDTO {
        let url = baseURL.appendingPathComponent("auth/login")
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("application/json", forHTTPHeaderField: "Accept")

        let body = LoginRequestDTO(email: email, password: password)
        request.httpBody = try JSONEncoder().encode(body)

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200..<300).contains(httpResponse.statusCode) else {
            throw APIError.invalidResponse
        }

        return try JSONDecoder().decode(LoginResponseDTO.self, from: data)
    }
}
```

---

## 3. Custom URLSession Configuration

```swift
func makeConfiguredSession() -> URLSession {
    let configuration = URLSessionConfiguration.default
    configuration.timeoutIntervalForRequest = 30
    configuration.timeoutIntervalForResource = 60
    configuration.waitsForConnectivity = true
    configuration.requestCachePolicy = .reloadIgnoringLocalCacheData

    return URLSession(configuration: configuration)
}
```

You might create different sessions per concern (API vs images).

---

## 4. Retrying with Exponential Backoff (Modern Pattern)

Networking is unreliable. Wrap operations with retry policies:

```swift
func retrying<T: Sendable>(
    maxAttempts: Int = 3,
    initialDelay: Duration = .seconds(0.5),
    multiplier: Double = 2.0,
    operation: @escaping @Sendable () async throws -> T
) async throws -> T {
    var delay = initialDelay

    for attempt in 1...maxAttempts {
        do {
            return try await operation()
        } catch let cancelError as CancellationError {
            throw error
        } catch {
            // If this was the last allowed attempt, rethrow
            if attempt == maxAttempts {
                throw error
            }

            // Otherwise sleep and prepare next delay
            try await Task.sleep(for: delay)
            delay = .seconds(Double(delay.components.seconds) * multiplier)
        }
    }

    // We should never reach here, but Swift requires a return or throw.
    throw CancellationError()
}
```

Usage:

```swift
let user = try await retrying {
    try await apiClient.fetchUser(id: 1)
}
```

---

## 5. WebSockets with URLSessionWebSocketTask

`URLSession` has first-party WebSocket support.

### Creating a WebSocket Task

```swift
let url = URL(string: "wss://echo.websocket.org")!
let webSocketTask = URLSession.shared.webSocketTask(with: url)
webSocketTask.resume()
```

---

### Sending Messages

```swift
func send(_ text: String, on task: URLSessionWebSocketTask) {
    task.send(.string(text)) { error in
        if let error {
            print("WebSocket send error: \(error)")
        }
    }
}
```

You can also send binary data via `.data(Data)`.

---

### Receiving Messages (Callback-Based)

```swift
func receive(on task: URLSessionWebSocketTask) {
    task.receive { result in
        switch result {
        case .failure(let error):
            print("WebSocket receive error: \(error)")

        case .success(let message):
            switch message {
            case .string(let text):
                print("Received text: \(text)")
            case .data(let data):
                print("Received data (\(data.count) bytes)")
            @unknown default:
                break
            }

            // Continue listening
            receive(on: task)
        }
    }
}
```

In modern code, you’d typically wrap this in `AsyncThrowingStream` (see the AsyncStreams guide).

---

## 6. Async/Await Wrapper for WebSocket Messages

```swift
func messages(
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

Usage:

```swift
for try await message in messages(for: webSocketTask) {
    switch message {
    case .string(let text):
        print("Received: \(text)")
    case .data(let data):
        print("Received \(data.count) bytes")
    @unknown default:
        break
    }
}
```

---

## 7. Security and HTTPS Basics

- iOS uses App Transport Security (ATS) → HTTPS by default.  
- Avoid disabling ATS or customizing TLS unless strictly necessary.  
- Use `Authorization` headers (e.g. `Bearer <token>`) for auth.  
- Scope any ATS exceptions to specific dev/staging domains if needed.

For most modern APIs, you just:

- Use `https` URLs.  
- Pass JSON bodies.  
- Validate status codes.  
- Decode DTOs.

---

## 8. Separation of Concerns (with Observation-Friendly View Models)

Keep a small, focused `APIClient` that:

- Builds `URLRequest`s.
- Calls `URLSession`.
- Decodes DTOs.
- Throws domain-friendly errors.

Your **repositories** combine:

- API client(s).  
- FileManager / local stores.  
- In-memory caches.

Your **view models** (using the Observation framework) depend on repositories, not `URLSession` directly:

```swift
import Observation

@MainActor
@Observable
final class UserViewModel {
    private let repository: UserRepository

    var state: State = .idle

    enum State {
        case idle
        case loading
        case loaded(User)
        case failed(Error)
    }

    init(repository: UserRepository) {
        self.repository = repository
    }

    func loadUser(id: Int) async {
        state = .loading
        do {
            let user = try await repository.user(id: id)
            state = .loaded(user)
        } catch {
            state = .failed(error)
        }
    }
}
```

SwiftUI then observes this view model without `ObservableObject` / `@Published`:

```swift
struct UserScreen: View {
    @State private var model: UserViewModel

    init(repository: UserRepository) {
        _model = State(initialValue: UserViewModel(repository: repository))
    }

    var body: some View {
        content
            .task {
                await model.loadUser(id: 1)
            }
    }

    @ViewBuilder
    private var content: some View {
        switch model.state {
        case .idle, .loading:
            ProgressView()
        case .loaded(let user):
            Text(user.name)
        case .failed(let error):
            Text("Error: \(error.localizedDescription)")
        }
    }
}
```

No `@ObservableObject`, `@ObservedObject`, or `@StateObject` needed — just `@Observable` + `@State`.

---

## 🤖 Tooling Note

Portions of drafting and editorial refinement in this repository were accelerated using large language models (including ChatGPT, Claude, and Gemini) under direct human design, validation, and final approval. All technical decisions, code, and architectural conclusions are authored and verified by the repository maintainer.


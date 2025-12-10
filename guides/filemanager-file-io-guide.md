# Guide: File I/O with FileManager in Modern Swift

This guide is a quick reference for reading and writing files on iOS using `FileManager`, often combined with `Codable` and Swift Concurrency (Swift 5.9+ / Swift 6).

---

## 1. Common Directories

Use `FileManager.default.urls(for:in:)` to find sandboxed locations.

```swift
let fileManager = FileManager.default

let documentsURL = fileManager.urls(for: .documentDirectory, in: .userDomainMask).first!
let cachesURL = fileManager.urls(for: .cachesDirectory, in: .userDomainMask).first!
let appSupportURL = fileManager.urls(for: .applicationSupportDirectory, in: .userDomainMask).first!
```

### Which directory should I use?

- **Documents**  
  User-facing data that should be backed up (notes, exports).

- **Application Support**  
  App-specific data not directly user-facing (local DB, JSON stores).

- **Caches**  
  Re-creatable data, safe to delete (image cache, API cache).

- **Temporary**

```swift
let temporaryURL = URL(fileURLWithPath: NSTemporaryDirectory(), isDirectory: true)
```

Short-lived scratch space. OS may wipe at any time.

---

## 2. Creating a Subdirectory

```swift
let directoryURL = appSupportURL.appendingPathComponent("UserData", isDirectory: true)

try fileManager.createDirectory(
    at: directoryURL,
    withIntermediateDirectories: true,
    attributes: nil
)
```

- `withIntermediateDirectories: true` → avoids errors when parents don’t exist.

---

## 3. Writing Raw Data to Disk

```swift
let fileURL = directoryURL.appendingPathComponent("user.json")

let data: Data = ... // e.g. JSON from JSONEncoder

try data.write(to: fileURL, options: .atomic)
```

- `.atomic` ensures the write either fully succeeds or not at all (no partial files).

---

## 4. Reading Raw Data from Disk

```swift
let fileURL = directoryURL.appendingPathComponent("user.json")

if fileManager.fileExists(atPath: fileURL.path) {
    let data = try Data(contentsOf: fileURL)
    // Decode or process data
} else {
    // File doesn't exist yet
}
```

---

## 5. Codable + FileManager (JSON Store Pattern)

A simple JSON store for a single model:

```swift
struct User: Codable, Sendable {
    let id: Int
    let name: String
}

struct UserStore {
    private let fileURL: URL
    private let fileManager: FileManager

    init(fileURL: URL, fileManager: FileManager = .default) {
        self.fileURL = fileURL
        self.fileManager = fileManager
    }

    func save(_ user: User) throws {
        let encoder = JSONEncoder()
        encoder.outputFormatting = [.prettyPrinted]

        let data = try encoder.encode(user)
        try data.write(to: fileURL, options: .atomic)
    }

    func load() throws -> User? {
        guard fileManager.fileExists(atPath: fileURL.path) else {
            return nil
        }

        let data = try Data(contentsOf: fileURL)
        let decoder = JSONDecoder()
        return try decoder.decode(User.self, from: data)
    }
}
```

Usage:

```swift
let storeURL = appSupportURL.appendingPathComponent("user.json")
let store = UserStore(fileURL: storeURL)

try store.save(user)
let loadedUser = try store.load()
```

---

## 6. Listing Files in a Directory

```swift
let fileURLs = try fileManager.contentsOfDirectory(
    at: directoryURL,
    includingPropertiesForKeys: nil,
    options: [.skipsHiddenFiles]
)

let jsonFiles = fileURLs.filter { $0.pathExtension == "json" }
```

You can pass `includingPropertiesForKeys` to fetch metadata like size, modification date, etc.

---

## 7. Removing Files and Directories

```swift
let fileURL = directoryURL.appendingPathComponent("user.json")
try fileManager.removeItem(at: fileURL)
```

For directories, `removeItem` deletes the directory and its contents.

Be careful when deleting from `Documents` or `Application Support` — this may be user data.

---

## 8. Checking for Existence and Attributes

```swift
if fileManager.fileExists(atPath: fileURL.path) {
    let attrs = try fileManager.attributesOfItem(atPath: fileURL.path)
    let size = attrs[.size] as? NSNumber
    let modified = attrs[.modificationDate] as? Date
}
```

Useful for “last updated” timestamps or file sizes.

---

## 9. Concurrency and Safety (Swift 6 Mindset)

`FileManager` operations are typically short and can be called from concurrent tasks, but you often want a central place to serialize reads/writes for a given directory/file set.

A modern pattern is to hide file access behind an `actor`:

```swift
actor FileStore {
    private let baseURL: URL
    private let fileManager: FileManager

    init(baseURL: URL, fileManager: FileManager = .default) {
        self.baseURL = baseURL
        self.fileManager = fileManager

        try? fileManager.createDirectory(
            at: baseURL,
            withIntermediateDirectories: true,
            attributes: nil
        )
    }

    func save(data: Data, fileName: String) throws {
        let url = baseURL.appendingPathComponent(fileName)
        try data.write(to: url, options: .atomic)
    }

    func load(fileName: String) throws -> Data? {
        let url = baseURL.appendingPathComponent(fileName)
        guard fileManager.fileExists(atPath: url.path) else {
            return nil
        }
        return try Data(contentsOf: url)
    }

    func delete(fileName: String) throws {
        let url = baseURL.appendingPathComponent(fileName)
        if fileManager.fileExists(atPath: url.path) {
            try fileManager.removeItem(at: url)
        }
    }
}
```

This works well with Swift 6’s stricter Sendable checking.

---

## 10. Error Handling Pattern

Wrap file and decoding errors into your own error type:

```swift
enum PersistenceError: Error {
    case fileNotFound(URL)
    case decodingFailed(Error)
    case encodingFailed(Error)
    case ioError(Error)
}
```

Example:

```swift
func loadUser(from url: URL, fileManager: FileManager = .default) throws -> User {
    do {
        guard fileManager.fileExists(atPath: url.path) else {
            throw PersistenceError.fileNotFound(url)
        }

        let data = try Data(contentsOf: url)
        do {
            return try JSONDecoder().decode(User.self, from: data)
        } catch {
            throw PersistenceError.decodingFailed(error)
        }
    } catch let error as PersistenceError {
        throw error
    } catch {
        throw PersistenceError.ioError(error)
    }
}
```

---

## 11. When to Move Beyond FileManager + JSON

`FileManager` + `Codable` is great for:

- Small to medium-sized JSON blobs.
- A handful of files (settings, small caches, user profile).

Consider a database (SQLite, Core Data, GRDB, Realm, etc.) when:

- You have **lots** of records with query needs (filtering, sorting, relationships).
- You want incremental updates instead of rewriting a large JSON file.
- You need more robust transactional semantics.

In many modern iOS apps you’ll mix:

- `UserDefaults` / `@AppStorage` for simple key-value settings.
- `FileManager` + JSON for simple data stores.
- A database layer for complex domain data.

---

## 🤖 Tooling Note

Portions of drafting and editorial refinement in this repository were accelerated using large language models (including ChatGPT, Claude, and Gemini) under direct human design, validation, and final approval. All technical decisions, code, and architectural conclusions are authored and verified by the repository maintainer.


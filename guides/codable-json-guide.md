# Guide: Codable, JSON DTOs, and Domain Models in Modern Swift

This guide is a fast, practical reference for using `Codable` with JSON in real apps, including a clean separation between **network DTOs** and **domain models**. It’s written with modern Swift (Swift 5.9+ / Swift 6) in mind.

---

## 1. Codable in One Sentence

```swift
protocol Codable = Encodable & Decodable
```

If all stored properties in a type are `Codable`, the compiler can usually synthesize encoding and decoding for you automatically.

```swift
struct UserDTO: Codable {
    let id: Int
    let fullName: String
    let email: String?
}
```

---

## 2. DTOs vs Domain Models

### DTO (Data Transfer Object)

- Mirrors the JSON payload from the network or disk.
- Naming and shape are close to the API.
- Lives in a “Networking” / “API” module.
- Is almost always `Codable`.

```swift
struct UserDTO: Codable {
    let id: Int
    let fullName: String
    let email: String?
    let createdAt: Date
}
```

### Domain Model

- Reflects your business logic.
- Designed for how the app wants to reason about data.
- Isolated from API quirks and changes.

```swift
struct User {
    struct ID: Hashable, Sendable {
        let rawValue: Int
    }

    let id: ID
    let name: String
    let email: EmailAddress?
    let createdAt: Date
    let isStaff: Bool
}
```

### Mapping DTO → Domain

```swift
extension User {
    init(dto: UserDTO) {
        self.id = .init(rawValue: dto.id)
        self.name = dto.fullName
        self.email = dto.email.map(EmailAddress.init(rawValue:))
        self.createdAt = dto.createdAt
        self.isStaff = dto.email?.hasSuffix("@company.com") == true
    }
}
```

**Rule of thumb:**  
- **DTOs** are `Codable` and tightly bound to the API.  
- **Domain models** are free to evolve; they may or may not be `Codable`.

---

## 3. Basic JSON Decoding

### From Data → DTO

```swift
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
decoder.dateDecodingStrategy = .iso8601

let user = try decoder.decode(UserDTO.self, from: data)
```

### From Data → Array of DTOs

```swift
let users = try decoder.decode([UserDTO].self, from: data)
```

---

## 4. Encoding JSON

```swift
let dto = UserDTO(id: 1, fullName: "Jonathan", email: nil, createdAt: .now)

let encoder = JSONEncoder()
encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
encoder.dateEncodingStrategy = .iso8601

let data = try encoder.encode(dto)
let jsonString = String(data: data, encoding: .utf8)
```

---

## 5. Custom JSON Keys with CodingKeys

Use when server keys don’t match your Swift names.

```swift
struct UserDTO: Codable {
    let id: Int
    let fullName: String
    let email: String?
    let createdAt: Date

    enum CodingKeys: String, CodingKey {
        case id
        case fullName = "full_name"
        case email
        case createdAt = "created_at"
    }
}
```

---

## 6. Required vs Optional Fields

```swift
struct ProfileDTO: Codable {
    let id: Int                 // required: decoding fails if missing
    let bio: String?            // optional: nil if missing or null
    let avatarURL: URL?         // optional
}
```

- Use **optionals** for fields that may be missing or `null`.  
- Use **non-optionals** for fields that must be present.

---

## 7. Default Values for Missing Keys

Pattern: use `decodeIfPresent` and provide defaults.

```swift
struct SettingsDTO: Codable {
    let theme: String
    let notificationsEnabled: Bool

    enum CodingKeys: String, CodingKey {
        case theme
        case notificationsEnabled = "notifications_enabled"
    }

    init(theme: String = "light", notificationsEnabled: Bool = true) {
        self.theme = theme
        self.notificationsEnabled = notificationsEnabled
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)

        let theme = try container.decodeIfPresent(String.self, forKey: .theme) ?? "light"
        let notificationsEnabled = try container.decodeIfPresent(Bool.self, forKey: .notificationsEnabled) ?? true

        self.init(theme: theme, notificationsEnabled: notificationsEnabled)
    }
}
```

Key APIs:

- `decode(_:forKey:)` → throws if key is missing or value is invalid.  
- `decodeIfPresent(_:forKey:)` → returns `nil` when key is missing or `null`.

---

## 8. Nested Containers

JSON:

```json
{
  "id": 1,
  "profile": {
    "bio": "Hello",
    "avatar_url": "https://example.com/avatar.png"
  }
}
```

DTO:

```swift
struct UserDTO: Codable {
    struct Profile: Codable {
        let bio: String
        let avatarURL: URL

        enum CodingKeys: String, CodingKey {
            case bio
            case avatarURL = "avatar_url"
        }
    }

    let id: Int
    let profile: Profile
}
```

---

## 9. Top-Level Arrays and Dictionaries

Top-level array:

```swift
let users = try decoder.decode([UserDTO].self, from: data)
```

Top-level dictionary:

```swift
let lookup = try decoder.decode([String: UserDTO].self, from: data)
```

---

## 10. Dates and Strategies

Always be **explicit** about date formats.

```swift
let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .iso8601
// Or .secondsSince1970, .millisecondsSince1970, .formatted(DateFormatter)
```

Align with your API team on formats and mirror in `JSONEncoder`.

---

## 11. Debugging Decoding Errors

```swift
do {
    let user = try JSONDecoder().decode(UserDTO.self, from: data)
    print(user)
} catch let error as DecodingError {
    switch error {
    case .keyNotFound(let key, let context):
        print("Missing key: \(key.stringValue) in \(context.codingPath)")
    case .typeMismatch(let type, let context):
        print("Type mismatch for \(type) in \(context.codingPath): \(context.debugDescription)")
    case .valueNotFound(let type, let context):
        print("Value not found for \(type) in \(context.codingPath): \(context.debugDescription)")
    case .dataCorrupted(let context):
        print("Data corrupted: \(context.debugDescription)")
    @unknown default:
        print("Unknown decoding error: \(error)")
    }
} catch {
    print("Other error: \(error)")
}
```

In debug builds, also log the raw JSON when safe.

---

## 12. DTO Layer Patterns (Modern Swift Concurrency)

Typical layering:

- `APIClient`  
  - fetches `Data` with `URLSession` + async/await  
  - decodes `DTO`s
- `Repository`  
  - converts `DTO`s to domain models  
  - applies business rules, caching, fallback
- UI / View Models  
  - talk only to repositories, never directly to DTOs or `URLSession`.

This naturally composes with Swift Concurrency and the Observation framework: your observable view models consume **domain models**, not network DTOs.

---

## 🤖 Tooling Note

Portions of drafting and editorial refinement in this repository were accelerated using large language models (including ChatGPT, Claude, and Gemini) under direct human design, validation, and final approval. All technical decisions, code, and architectural conclusions are authored and verified by the repository maintainer.

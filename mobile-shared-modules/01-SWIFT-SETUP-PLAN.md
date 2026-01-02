# Plan de Setup: edugo-swift-shared

## Resumen Ejecutivo

Este documento define el plan completo para crear el proyecto **edugo-swift-shared**, 
un Swift Package con múltiples módulos independientes equivalente a `edugo-shared` de Go.

**Plataformas objetivo**: iOS 26+, iPadOS 26+, macOS 26+
**Swift Version**: 6.0+ con StrictConcurrency habilitado

---

## 1. Package.swift Completo

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "edugo-swift-shared",
    platforms: [
        .iOS(.v18),       // Mínimo iOS 18 para Mutex del Synchronization framework
        .macOS(.v15),     // macOS 15 Sequoia
        .tvOS(.v18),
        .watchOS(.v11),
        .visionOS(.v2)
    ],
    products: [
        // MARK: - Cross Modules (Reutilizables)
        .library(name: "EduGoCommon", targets: ["EduGoCommon"]),
        .library(name: "EduGoLogger", targets: ["EduGoLogger"]),
        .library(name: "EduGoNetwork", targets: ["EduGoNetwork"]),
        .library(name: "EduGoStorage", targets: ["EduGoStorage"]),
        .library(name: "EduGoAuth", targets: ["EduGoAuth"]),
        .library(name: "EduGoAnalytics", targets: ["EduGoAnalytics"]),
        
        // MARK: - Specific Modules (EduGo Domain)
        .library(name: "EduGoRoles", targets: ["EduGoRoles"]),
        .library(name: "EduGoModels", targets: ["EduGoModels"]),
        .library(name: "EduGoAPI", targets: ["EduGoAPI"]),
    ],
    dependencies: [
        // Sin dependencias externas - todo nativo
    ],
    targets: [
        // MARK: - EduGoCommon (Base)
        .target(
            name: "EduGoCommon",
            dependencies: [],
            path: "Sources/EduGoCommon",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoCommonTests", dependencies: ["EduGoCommon"]),
        
        // MARK: - EduGoLogger
        .target(
            name: "EduGoLogger",
            dependencies: ["EduGoCommon"],
            path: "Sources/EduGoLogger",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoLoggerTests", dependencies: ["EduGoLogger"]),
        
        // MARK: - EduGoNetwork
        .target(
            name: "EduGoNetwork",
            dependencies: ["EduGoCommon", "EduGoLogger"],
            path: "Sources/EduGoNetwork",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoNetworkTests", dependencies: ["EduGoNetwork"]),
        
        // MARK: - EduGoStorage
        .target(
            name: "EduGoStorage",
            dependencies: ["EduGoCommon"],
            path: "Sources/EduGoStorage",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoStorageTests", dependencies: ["EduGoStorage"]),
        
        // MARK: - EduGoAuth
        .target(
            name: "EduGoAuth",
            dependencies: ["EduGoCommon", "EduGoStorage", "EduGoNetwork"],
            path: "Sources/EduGoAuth",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoAuthTests", dependencies: ["EduGoAuth"]),
        
        // MARK: - EduGoAnalytics
        .target(
            name: "EduGoAnalytics",
            dependencies: ["EduGoCommon", "EduGoLogger"],
            path: "Sources/EduGoAnalytics",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoAnalyticsTests", dependencies: ["EduGoAnalytics"]),
        
        // MARK: - EduGoRoles (Specific)
        .target(
            name: "EduGoRoles",
            dependencies: ["EduGoCommon"],
            path: "Sources/EduGoRoles",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoRolesTests", dependencies: ["EduGoRoles"]),
        
        // MARK: - EduGoModels (Specific)
        .target(
            name: "EduGoModels",
            dependencies: ["EduGoCommon", "EduGoRoles"],
            path: "Sources/EduGoModels",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoModelsTests", dependencies: ["EduGoModels"]),
        
        // MARK: - EduGoAPI (Specific)
        .target(
            name: "EduGoAPI",
            dependencies: ["EduGoNetwork", "EduGoAuth", "EduGoModels"],
            path: "Sources/EduGoAPI",
            swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
        ),
        .testTarget(name: "EduGoAPITests", dependencies: ["EduGoAPI"]),
    ]
)
```

---

## 2. Estructura de Carpetas

```
edugo-swift-shared/
├── Package.swift
├── README.md
├── CHANGELOG.md
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── release.yml
│       └── lint.yml
├── Sources/
│   ├── EduGoCommon/
│   │   ├── Errors/
│   │   │   ├── AppError.swift
│   │   │   └── ErrorCode.swift
│   │   ├── Extensions/
│   │   │   ├── String+Extensions.swift
│   │   │   └── Date+Extensions.swift
│   │   └── Types/
│   │       └── Result+Extensions.swift
│   │
│   ├── EduGoLogger/
│   │   ├── Logger.swift           # Protocol
│   │   ├── LogLevel.swift
│   │   └── Providers/
│   │       └── OSLoggerProvider.swift
│   │
│   ├── EduGoNetwork/
│   │   ├── HTTPClient.swift       # Protocol
│   │   ├── HTTPMethod.swift
│   │   ├── HTTPError.swift
│   │   ├── Request/
│   │   │   ├── Endpoint.swift
│   │   │   └── RequestBuilder.swift
│   │   └── Providers/
│   │       └── URLSessionClient.swift
│   │
│   ├── EduGoStorage/
│   │   ├── SecureStorage.swift    # Protocol
│   │   └── Keychain/
│   │       ├── KeychainService.swift
│   │       └── KeychainError.swift
│   │
│   ├── EduGoAuth/
│   │   ├── TokenManager.swift     # Actor
│   │   ├── AuthState.swift
│   │   ├── JWT/
│   │   │   ├── JWTDecoder.swift
│   │   │   └── JWTPayload.swift
│   │   └── Session/
│   │       └── SessionManager.swift
│   │
│   ├── EduGoAnalytics/
│   │   ├── AnalyticsService.swift  # Protocol
│   │   ├── AnalyticsEvent.swift
│   │   └── Providers/
│   │       └── ConsoleAnalyticsProvider.swift
│   │
│   ├── EduGoRoles/
│   │   ├── SystemRole.swift
│   │   └── Permissions.swift
│   │
│   ├── EduGoModels/
│   │   ├── User/
│   │   │   ├── User.swift
│   │   │   └── UserProfile.swift
│   │   ├── School/
│   │   │   ├── School.swift
│   │   │   └── SchoolMembership.swift
│   │   └── Material/
│   │       ├── Material.swift
│   │       └── MaterialProgress.swift
│   │
│   └── EduGoAPI/
│       ├── EduGoAPIClient.swift   # Actor
│       ├── Endpoints/
│       │   ├── AuthEndpoints.swift
│       │   ├── SchoolEndpoints.swift
│       │   └── MaterialEndpoints.swift
│       └── Responses/
│           └── APIResponse.swift
│
└── Tests/
    ├── EduGoCommonTests/
    ├── EduGoLoggerTests/
    ├── EduGoNetworkTests/
    ├── EduGoStorageTests/
    ├── EduGoAuthTests/
    ├── EduGoAnalyticsTests/
    ├── EduGoRolesTests/
    ├── EduGoModelsTests/
    └── EduGoAPITests/
```

---

## 3. Consideraciones Swift 6 (StrictConcurrency)

### 3.1 Sendable Compliance

Todos los tipos públicos deben ser `Sendable`:

```swift
// EduGoCommon/Errors/AppError.swift
public struct AppError: Error, Sendable {
    public let code: ErrorCode
    public let message: String
    public let underlyingError: (any Error & Sendable)?
    
    public init(code: ErrorCode, message: String, underlyingError: (any Error & Sendable)? = nil) {
        self.code = code
        self.message = message
        self.underlyingError = underlyingError
    }
}

public enum ErrorCode: String, Sendable, CaseIterable {
    // Auth
    case unauthorized = "UNAUTHORIZED"
    case tokenExpired = "TOKEN_EXPIRED"
    case invalidCredentials = "INVALID_CREDENTIALS"
    
    // Network
    case networkError = "NETWORK_ERROR"
    case timeout = "TIMEOUT"
    case serverError = "SERVER_ERROR"
    
    // Validation
    case validationFailed = "VALIDATION_FAILED"
    case notFound = "NOT_FOUND"
    
    // Storage
    case storageFailed = "STORAGE_FAILED"
    case keychainError = "KEYCHAIN_ERROR"
    
    // General
    case unknown = "UNKNOWN"
}
```

### 3.2 Actors para Estado Mutable

```swift
// EduGoAuth/TokenManager.swift
public actor TokenManager {
    private let storage: any SecureStorage
    private var cachedToken: TokenInfo?
    
    public init(storage: any SecureStorage) {
        self.storage = storage
    }
    
    public func getAccessToken() async throws -> String {
        if let cached = cachedToken, !cached.isExpired {
            return cached.accessToken
        }
        
        // Refresh logic
        let newToken = try await refreshToken()
        cachedToken = newToken
        return newToken.accessToken
    }
    
    public func storeToken(_ token: TokenInfo) async throws {
        cachedToken = token
        try await storage.save(key: "auth_token", value: token.encoded())
    }
    
    public func clearToken() async throws {
        cachedToken = nil
        try await storage.delete(key: "auth_token")
    }
    
    private func refreshToken() async throws -> TokenInfo {
        // Implementation
    }
}
```

### 3.3 Mutex para Sincronización (iOS 18+)

```swift
import Synchronization

// Para código que necesita sincronización sin ser actor
public final class AtomicCounter: Sendable {
    private let mutex = Mutex(0)
    
    public init() {}
    
    public func increment() -> Int {
        mutex.withLock { value in
            value += 1
            return value
        }
    }
    
    public var current: Int {
        mutex.withLock { $0 }
    }
}
```

### 3.4 nonisolated para Métodos Sin Estado

```swift
public actor AnalyticsManager {
    private var events: [AnalyticsEvent] = []
    
    // No necesita aislamiento - solo retorna constante
    nonisolated public var serviceName: String { "EduGoAnalytics" }
    
    public func track(_ event: AnalyticsEvent) {
        events.append(event)
    }
}
```

---

## 4. APIs Recomendadas vs Deprecadas

| Área | Deprecado | Recomendado (Swift 6) |
|------|-----------|----------------------|
| Logging | `print()`, `NSLog()` | `os.Logger` |
| Networking | Completion handlers | `async/await` con URLSession |
| Concurrency | `DispatchQueue` | Structured concurrency, Actors |
| Keychain | Wrappers externos | Security framework nativo |
| JSON | JSONSerialization manual | `Codable` |
| Sync | Locks manuales | `Mutex` (Synchronization) |

---

## 5. Ejemplos de Código por Módulo

### 5.1 EduGoLogger

```swift
// Logger.swift
public protocol Logger: Sendable {
    func debug(_ message: String, file: String, function: String, line: Int)
    func info(_ message: String, file: String, function: String, line: Int)
    func warning(_ message: String, file: String, function: String, line: Int)
    func error(_ message: String, error: Error?, file: String, function: String, line: Int)
}

public extension Logger {
    func debug(_ message: String, file: String = #file, function: String = #function, line: Int = #line) {
        debug(message, file: file, function: function, line: line)
    }
    // ... similar para info, warning, error
}

// Providers/OSLoggerProvider.swift
import os

public struct OSLoggerProvider: Logger {
    private let logger: os.Logger
    
    public init(subsystem: String, category: String) {
        self.logger = os.Logger(subsystem: subsystem, category: category)
    }
    
    public func debug(_ message: String, file: String, function: String, line: Int) {
        logger.debug("\(message, privacy: .public) [\(function):\(line)]")
    }
    
    public func info(_ message: String, file: String, function: String, line: Int) {
        logger.info("\(message, privacy: .public)")
    }
    
    public func warning(_ message: String, file: String, function: String, line: Int) {
        logger.warning("\(message, privacy: .public)")
    }
    
    public func error(_ message: String, error: Error?, file: String, function: String, line: Int) {
        if let error {
            logger.error("\(message, privacy: .public): \(error.localizedDescription, privacy: .public)")
        } else {
            logger.error("\(message, privacy: .public)")
        }
    }
}
```

### 5.2 EduGoNetwork

```swift
// HTTPClient.swift
public protocol HTTPClient: Sendable {
    func execute<T: Decodable & Sendable>(_ endpoint: Endpoint) async throws -> T
    func execute(_ endpoint: Endpoint) async throws -> Data
}

// Providers/URLSessionClient.swift
public actor URLSessionClient: HTTPClient {
    private let session: URLSession
    private let decoder: JSONDecoder
    private let logger: any Logger
    
    public init(
        configuration: URLSessionConfiguration = .default,
        logger: any Logger
    ) {
        self.session = URLSession(configuration: configuration)
        self.decoder = JSONDecoder()
        self.decoder.dateDecodingStrategy = .iso8601
        self.logger = logger
    }
    
    public func execute<T: Decodable & Sendable>(_ endpoint: Endpoint) async throws -> T {
        let data = try await execute(endpoint)
        return try decoder.decode(T.self, from: data)
    }
    
    public func execute(_ endpoint: Endpoint) async throws -> Data {
        let request = try endpoint.urlRequest()
        
        logger.debug("Request: \(endpoint.method.rawValue) \(endpoint.path)")
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw AppError(code: .networkError, message: "Invalid response type")
        }
        
        guard (200...299).contains(httpResponse.statusCode) else {
            throw AppError(
                code: .serverError,
                message: "HTTP \(httpResponse.statusCode)"
            )
        }
        
        return data
    }
}
```

### 5.3 EduGoStorage (Keychain)

```swift
// SecureStorage.swift
public protocol SecureStorage: Sendable {
    func save(key: String, value: Data) async throws
    func load(key: String) async throws -> Data?
    func delete(key: String) async throws
}

// Keychain/KeychainService.swift
import Security

public actor KeychainService: SecureStorage {
    private let service: String
    private let accessGroup: String?
    
    public init(service: String, accessGroup: String? = nil) {
        self.service = service
        self.accessGroup = accessGroup
    }
    
    public func save(key: String, value: Data) async throws {
        // Delete existing first
        try? await delete(key: key)
        
        var query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecValueData as String: value,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
        ]
        
        if let accessGroup {
            query[kSecAttrAccessGroup as String] = accessGroup
        }
        
        let status = SecItemAdd(query as CFDictionary, nil)
        
        guard status == errSecSuccess else {
            throw AppError(code: .keychainError, message: "Failed to save: \(status)")
        }
    }
    
    public func load(key: String) async throws -> Data? {
        var query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]
        
        if let accessGroup {
            query[kSecAttrAccessGroup as String] = accessGroup
        }
        
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        
        if status == errSecItemNotFound {
            return nil
        }
        
        guard status == errSecSuccess else {
            throw AppError(code: .keychainError, message: "Failed to load: \(status)")
        }
        
        return result as? Data
    }
    
    public func delete(key: String) async throws {
        var query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key
        ]
        
        if let accessGroup {
            query[kSecAttrAccessGroup as String] = accessGroup
        }
        
        let status = SecItemDelete(query as CFDictionary)
        
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw AppError(code: .keychainError, message: "Failed to delete: \(status)")
        }
    }
}
```

### 5.4 EduGoAuth (JWT Decoder Nativo)

```swift
// JWT/JWTDecoder.swift
public struct JWTDecoder: Sendable {
    public init() {}
    
    public func decode(_ token: String) throws -> JWTPayload {
        let parts = token.split(separator: ".")
        guard parts.count == 3 else {
            throw AppError(code: .invalidCredentials, message: "Invalid JWT format")
        }
        
        let payloadBase64 = String(parts[1])
            .replacingOccurrences(of: "-", with: "+")
            .replacingOccurrences(of: "_", with: "/")
        
        // Pad if necessary
        let padded = payloadBase64.padding(
            toLength: ((payloadBase64.count + 3) / 4) * 4,
            withPad: "=",
            startingAt: 0
        )
        
        guard let data = Data(base64Encoded: padded) else {
            throw AppError(code: .invalidCredentials, message: "Invalid base64 payload")
        }
        
        let decoder = JSONDecoder()
        return try decoder.decode(JWTPayload.self, from: data)
    }
}

// JWT/JWTPayload.swift
public struct JWTPayload: Codable, Sendable {
    public let sub: String           // Subject (user ID)
    public let exp: Int              // Expiration timestamp
    public let iat: Int              // Issued at
    public let roles: [String]?
    
    public var isExpired: Bool {
        Date().timeIntervalSince1970 > Double(exp)
    }
    
    public var expirationDate: Date {
        Date(timeIntervalSince1970: Double(exp))
    }
}
```

### 5.5 EduGoRoles

```swift
// SystemRole.swift
public enum SystemRole: String, Codable, Sendable, CaseIterable {
    case student = "student"
    case teacher = "teacher"
    case guardian = "guardian"
    case schoolAdmin = "school_admin"
    case systemAdmin = "system_admin"
    
    public var displayName: String {
        switch self {
        case .student: return "Estudiante"
        case .teacher: return "Profesor"
        case .guardian: return "Acudiente"
        case .schoolAdmin: return "Administrador de Escuela"
        case .systemAdmin: return "Administrador del Sistema"
        }
    }
    
    public var permissions: Set<Permission> {
        switch self {
        case .student:
            return [.viewMaterials, .takeQuizzes, .viewProgress]
        case .teacher:
            return [.viewMaterials, .uploadMaterials, .createQuizzes, .viewStudentProgress]
        case .guardian:
            return [.viewProgress, .viewStudentInfo]
        case .schoolAdmin:
            return [.manageSchool, .manageUsers, .viewReports]
        case .systemAdmin:
            return Permission.all
        }
    }
}

// Permissions.swift
public enum Permission: String, Sendable, CaseIterable {
    case viewMaterials
    case uploadMaterials
    case takeQuizzes
    case createQuizzes
    case viewProgress
    case viewStudentProgress
    case viewStudentInfo
    case manageSchool
    case manageUsers
    case viewReports
    
    public static var all: Set<Permission> {
        Set(allCases)
    }
}
```

---

## 6. CI/CD con GitHub Actions

### 6.1 ci.yml

```yaml
name: CI

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: macos-14
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Select Xcode
      run: sudo xcode-select -s /Applications/Xcode_16.0.app
    
    - name: Build
      run: swift build -v
    
    - name: Run Tests
      run: swift test -v
    
    - name: Build for iOS
      run: |
        xcodebuild build \
          -scheme edugo-swift-shared \
          -destination 'platform=iOS Simulator,name=iPhone 16'
    
    - name: Build for macOS
      run: |
        xcodebuild build \
          -scheme edugo-swift-shared \
          -destination 'platform=macOS'
```

### 6.2 lint.yml

```yaml
name: Lint

on:
  pull_request:
    branches: [main]

jobs:
  swiftlint:
    runs-on: macos-14
    
    steps:
    - uses: actions/checkout@v4
    
    - name: SwiftLint
      run: |
        brew install swiftlint
        swiftlint --strict
```

### 6.3 release.yml

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: macos-14
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Build Release
      run: swift build -c release
    
    - name: Create GitHub Release
      uses: softprops/action-gh-release@v1
      with:
        generate_release_notes: true
```

---

## 7. Versionado

### Estrategia: Monorepo con Changelogs por Módulo

```
edugo-swift-shared/
├── CHANGELOG.md           # Changelog global
└── Sources/
    ├── EduGoAuth/
    │   └── CHANGELOG.md   # Changelog del módulo
    ├── EduGoLogger/
    │   └── CHANGELOG.md
    └── ...
```

### Formato de Tags

- Global: `v1.0.0`
- Por módulo (opcional): `auth/v1.0.0`, `logger/v1.0.0`

---

## 8. Dependencias del Grafo de Módulos

```
EduGoCommon (base)
    ↑
    ├── EduGoLogger
    │       ↑
    │       └── EduGoNetwork ← EduGoAnalytics
    │               ↑
    ├── EduGoStorage
    │       ↑
    │       └── EduGoAuth
    │               ↑
    ├── EduGoRoles
    │       ↑
    │       └── EduGoModels
    │               ↑
    └───────────────└── EduGoAPI (integra todo)
```

---

## 9. Fuentes Consultadas

- [Swift.org - Enabling Complete Concurrency Checking](https://www.swift.org/documentation/concurrency/)
- [Apple Developer - Adopting strict concurrency in Swift 6](https://developer.apple.com/documentation/swift/adoptingswift6)
- [SwiftLee - Modern Swift Lock: Mutex & Synchronization Framework](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/)
- [SwiftLee - URLSession with Async/Await](https://www.avanderlee.com/concurrency/urlsession-async-await-network-requests-in-swift/)
- [SwiftLee - OSLog and Unified Logging](https://www.avanderlee.com/debugging/oslog-unified-logging/)
- [Antoine van der Lee - Approachable Concurrency in Swift 6.2](https://www.avanderlee.com/concurrency/approachable-concurrency-in-swift-6-2-a-clear-guide/)

---

## 10. Archivos de Referencia en EduUI

Los siguientes archivos existentes en EduUI sirvieron como referencia de patrones:

- `EduUI/apple-app/Packages/EduGoObservability/Sources/EduGoObservability/Logging/Core/Logger.swift`
- `EduUI/apple-app/Packages/EduGoSecureStorage/Sources/EduGoSecureStorage/Keychain/KeychainService.swift`
- `EduUI/apple-app/Packages/EduGoDomainCore/Sources/EduGoDomainCore/Errors/AppError.swift`

---

**Estado**: Plan completo listo para implementación
**Última actualización**: 2026-01-02

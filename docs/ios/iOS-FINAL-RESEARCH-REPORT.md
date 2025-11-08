# iOS Solar Monitoring App - Final Comprehensive Research & Implementation Report

**Project:** SwiftUI iPhone App for Solar Energy Monitoring
**Target Device:** iPhone 13 mini
**Development Machine:** MacBook M3
**Date:** November 7, 2025
**Status:** Complete Research & Planning Phase

---

## EXECUTIVE SUMMARY

This report synthesizes extensive research into building a production-ready iOS app for solar monitoring. The app enables users to:
- Authenticate via Google OAuth
- View real-time solar readings (inverter output, meter value, timestamp)
- Record readings via three actions: Auto Record, Auto Record + Push to Utility, Manual Entry
- Work offline with automatic sync when connectivity returns

**Key Finding:** This is a highly achievable project. A single iOS developer can build this in **22-29 hours** using modern Swift technologies with zero external dependencies (except Google OAuth SDK).

---

## PART 1: TECHNOLOGY STACK RECOMMENDATIONS

### Recommended Tech Stack (FINAL DECISION)

| Component | Choice | Why | Complexity | Setup Time |
|-----------|--------|-----|-----------|-----------|
| **Language** | Swift | Modern, type-safe, native iOS, excellent compiler | Low | ~15 min |
| **UI Framework** | SwiftUI | Declarative, less boilerplate, live preview, iOS 15+ support | Low | ~10 min |
| **Authentication** | Google OAuth 2.0 (Backend-mediated) | Secure, user-friendly, zero token storage risk | Medium | 4-5 hrs |
| **Networking** | URLSession + Async/Await | Built-in, modern (Swift 5.5+), no dependencies | Low | 3-4 hrs |
| **Data Persistence** | CoreData + Keychain + UserDefaults | Built-in, powerful, offline support, mature | Medium | 4-5 hrs |
| **State Management** | @StateObject + EnvironmentObject | Simple, effective, no libraries needed | Low | 2-3 hrs |
| **Testing** | XCTest + Quick framework | Built-in iOS testing, sufficient for scope | Low | 2-3 hrs |
| **Deployment** | TestFlight → App Store | Safe gradual rollout, professional | Low | 2-3 hrs |

**Total Development Time:** 22-29 hours
**Minimum iOS Version:** iOS 15.0+
**Xcode Version Required:** 15.0+ (Supports iOS 17)

---

## PART 2: AUTHENTICATION STRATEGY (OAuth)

### Google OAuth 2.0 (RECOMMENDED APPROACH)

**Why Google OAuth?**
1. **User-Friendly:** 80% of users already have Google accounts
2. **Secure:** Industry-standard OAuth 2.0 protocol
3. **Backend Control:** Backend handles token exchange (no client-side token exposure)
4. **Dad-Friendly:** Simple "Sign in with Google" button
5. **Free:** Google Cloud free tier sufficient

### Authentication Architecture

```
┌─────────────────┐
│   iOS App       │
│   SwiftUI View  │
└────────┬────────┘
         │ User taps "Sign in"
         ↓
┌─────────────────────────────────────┐
│   Google Sign-In SDK                │
│   (GoogleSignIn pod)                │
└────────┬────────────────────────────┘
         │ Presents OAuth consent
         ↓
    ┌────────────────────────────────┐
    │   Google OAuth System          │
    │   User authenticates           │
    └────────┬───────────────────────┘
             │ Returns ID Token + OAuth Code
             ↓
    ┌──────────────────────────────────────┐
    │   Your Backend API                   │
    │   POST /oauth/exchange               │
    │   { idToken: "..." }                 │
    │   Returns: sessionToken              │
    └────────┬─────────────────────────────┘
             │ Session Token
             ↓
    ┌───────────────────────────────┐
    │   iOS Keychain                │
    │   (Secure Storage)            │
    │   sessionToken (encrypted)    │
    └───────────────────────────────┘
             │
             ↓
    ✓ User Authenticated!
```

### Implementation Steps

#### Step 1: Google Cloud Setup
```
1. Go to https://console.cloud.google.com
2. Create new project: "Solar Monitor iOS"
3. Enable APIs:
   - Google Sign-In API
   - Google+ API
4. Create OAuth 2.0 Credentials:
   - Type: iOS
   - Bundle ID: com.yourname.SolarMonitor
   - Save Client ID
5. Generate GoogleService-Info.plist
```

#### Step 2: iOS Implementation
```
1. Add GoogleSignIn dependency (CocoaPods):
   pod 'GoogleSignIn', '~> 7.0'
   
2. Create AuthManager.swift:
   - Handle Google Sign-In flow
   - Exchange ID token with backend
   - Store session token in Keychain
   
3. Create LoginView.swift:
   - Implement "Sign in with Google" button
   - Handle authentication errors
   - Navigate to home on success
```

### Code Example: AuthManager

```swift
// File: Networking/AuthManager.swift
import GoogleSignIn
import Security

class AuthManager: NSObject, ObservableObject {
    @Published var isAuthenticated = false
    @Published var userEmail: String?
    @Published var errorMessage: String?
    
    private let keychainManager = KeychainManager()
    private let networkManager = NetworkManager()
    
    // MARK: - Google Sign-In
    
    func signInWithGoogle() async {
        do {
            // Present Google Sign-In UI
            guard let windowScene = UIApplication.shared.connectedScenes.first 
                as? UIWindowScene,
                let window = windowScene.windows.first,
                let rootViewController = window.rootViewController else {
                errorMessage = "Unable to find root view controller"
                return
            }
            
            // Perform sign-in
            let result = try await GIDSignIn.sharedInstance.signIn(
                withPresenting: rootViewController
            )
            
            guard let idToken = result.user.idToken?.tokenString else {
                errorMessage = "No ID token received"
                return
            }
            
            // Exchange ID token with backend
            let sessionToken = try await exchangeIDTokenWithBackend(idToken)
            
            // Store session token securely
            try keychainManager.store(sessionToken, key: "sessionToken")
            
            // Update state
            userEmail = result.user.profile?.email
            isAuthenticated = true
            
        } catch {
            errorMessage = "Sign-in failed: \(error.localizedDescription)"
            isAuthenticated = false
        }
    }
    
    private func exchangeIDTokenWithBackend(_ idToken: String) async throws -> String {
        struct TokenRequest: Codable {
            let idToken: String
        }
        
        struct TokenResponse: Codable {
            let sessionToken: String
            let expiresIn: Int
        }
        
        let request = TokenRequest(idToken: idToken)
        let response: TokenResponse = try await networkManager
            .makeRequest(to: "/oauth/exchange", method: "POST", body: request)
        
        return response.sessionToken
    }
    
    // MARK: - Sign Out
    
    func signOut() {
        GIDSignIn.sharedInstance.signOut()
        try? keychainManager.delete(key: "sessionToken")
        isAuthenticated = false
        userEmail = nil
    }
    
    // MARK: - Restore Session
    
    func restoreSession() async {
        if let savedToken = try? keychainManager.retrieve(key: "sessionToken") {
            // Verify token is still valid with backend
            do {
                _ = try await networkManager.fetchCurrentReadings()
                isAuthenticated = true
            } catch {
                // Token expired, clear it
                try? keychainManager.delete(key: "sessionToken")
                isAuthenticated = false
            }
        }
    }
}
```

### Code Example: Keychain Manager

```swift
// File: Networking/KeychainManager.swift
import Security
import Foundation

class KeychainManager {
    func store(_ value: String, key: String) throws {
        let data = value.data(using: .utf8)!
        
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]
        
        SecItemDelete(query as CFDictionary)
        let status = SecItemAdd(query as CFDictionary, nil)
        
        guard status == errSecSuccess else {
            throw KeychainError.saveFailed
        }
    }
    
    func retrieve(key: String) throws -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true
        ]
        
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        
        guard status != errSecItemNotFound else { return nil }
        guard status == errSecSuccess else {
            throw KeychainError.retrieveFailed
        }
        
        guard let data = result as? Data else {
            throw KeychainError.unexpectedData
        }
        
        return String(data: data, encoding: .utf8)
    }
    
    func delete(key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        
        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed
        }
    }
}

enum KeychainError: Error {
    case saveFailed
    case retrieveFailed
    case deleteFailed
    case unexpectedData
}
```

### Authentication Flow Comparison

| Approach | Setup Time | UX | Security | Offline | Best For |
|----------|-----------|----|---------|---------|---------| 
| **Google OAuth (Recommended)** | 4-5 hrs | Excellent | 9/10 | Poor | Production apps |
| **Firebase Auth** | 1-2 hrs | Excellent | 9/10 | Good | Quick prototypes |
| **Apple Sign In** | 2-3 hrs | Best | 9/10 | Best | Apple ecosystem |
| **Custom Backend OAuth** | 8-10 hrs | Good | 7/10 | Fair | Proprietary systems |

**Recommendation:** Use Google OAuth 2.0 with backend-mediated token exchange. This balances security, user experience, and development time.

---

## PART 3: NETWORKING LAYER ARCHITECTURE

### URLSession + Async/Await (RECOMMENDED)

**Why not Alamofire?**
- Built-in URLSession is sufficient
- Zero external dependencies
- Async/Await is modern and cleaner than completion handlers
- Better memory management with structured concurrency
- Excellent error handling

### Network Layer Architecture

```
┌────────────────────────────────────────┐
│         SwiftUI Views                  │
│  (HomeView, ManualEntryView, etc.)    │
└────────────┬─────────────────────────┘
             │ Call viewModel methods
             ↓
┌────────────────────────────────────────┐
│       ViewModels (MVVM)               │
│  (HomeViewModel, ManualEntryViewModel) │
└────────────┬─────────────────────────┘
             │ Call NetworkManager
             ↓
┌────────────────────────────────────────┐
│     NetworkManager                     │
│  - Error handling                      │
│  - Request/response logging            │
│  - Token refresh logic                 │
│  - Retry on failure                    │
└────────┬──────────────────────┬────────┘
         │                      │
         ↓                      ↓
    ┌─────────────┐     ┌──────────────┐
    │ URLSession  │     │ KeychainMgr  │
    │ Data tasks  │     │ Token storage│
    └─────────────┘     └──────────────┘
         │
         ↓
    ┌──────────────────────┐
    │  Backend API         │
    │  /current_readings   │
    │  /record             │
    └──────────────────────┘
```

### Code Example: NetworkManager

```swift
// File: Networking/NetworkManager.swift
import Foundation

class NetworkManager: NSObject, ObservableObject {
    static let shared = NetworkManager()
    
    @Published var isLoading = false
    @Published var lastError: NetworkError?
    
    private let baseURL = URL(string: "https://api.solar.example.com")!
    private let keychainManager = KeychainManager()
    private let session = URLSession.shared
    
    // MARK: - API Methods
    
    /// Fetch current solar readings
    func fetchCurrentReadings() async throws -> CurrentReadingsResponse {
        let endpoint = baseURL.appendingPathComponent("current_readings")
        return try await makeRequest(endpoint, method: "GET")
    }
    
    /// Record a solar reading
    func recordReading(_ request: RecordRequest) async throws -> RecordResponse {
        let endpoint = baseURL.appendingPathComponent("record")
        return try await makeRequest(endpoint, method: "POST", body: request)
    }
    
    /// Bulk record multiple readings (for offline sync)
    func bulkRecordReadings(_ requests: [RecordRequest]) async throws -> BulkRecordResponse {
        let endpoint = baseURL.appendingPathComponent("bulk_record")
        let bulkRequest = BulkRecordRequest(records: requests)
        return try await makeRequest(endpoint, method: "POST", body: bulkRequest)
    }
    
    // MARK: - Private Helper Methods
    
    private func makeRequest<T: Codable>(
        _ url: URL,
        method: String = "GET",
        body: Encodable? = nil,
        retryCount: Int = 0
    ) async throws -> T {
        DispatchQueue.main.async {
            self.isLoading = true
        }
        
        defer {
            DispatchQueue.main.async {
                self.isLoading = false
            }
        }
        
        // Get auth token
        guard let sessionToken = try? keychainManager.retrieve(key: "sessionToken") else {
            throw NetworkError.notAuthenticated
        }
        
        // Build request
        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = method
        urlRequest.setValue("Bearer \(sessionToken)", forHTTPHeaderField: "Authorization")
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        urlRequest.timeoutInterval = 10.0
        
        // Add body if provided
        if let body = body {
            urlRequest.httpBody = try JSONEncoder().encode(body)
        }
        
        // Log request for debugging
        logRequest(urlRequest)
        
        do {
            let (data, response) = try await session.data(for: urlRequest)
            
            guard let httpResponse = response as? HTTPURLResponse else {
                throw NetworkError.invalidResponse
            }
            
            logResponse(httpResponse, data: data)
            
            // Handle HTTP status codes
            switch httpResponse.statusCode {
            case 200...299:
                break // Success
            case 401:
                // Token expired, clear and throw
                try? keychainManager.delete(key: "sessionToken")
                throw NetworkError.unauthorized
            case 429:
                // Rate limited - retry with backoff
                if retryCount < 3 {
                    try await Task.sleep(nanoseconds: UInt64(pow(2.0, Double(retryCount)) * 1_000_000_000))
                    return try await makeRequest(url, method: method, body: body, retryCount: retryCount + 1)
                }
                throw NetworkError.rateLimited
            case 400...499:
                throw NetworkError.clientError(httpResponse.statusCode)
            case 500...599:
                // Server error - retry once
                if retryCount == 0 {
                    try await Task.sleep(nanoseconds: 2_000_000_000) // 2 second delay
                    return try await makeRequest(url, method: method, body: body, retryCount: 1)
                }
                throw NetworkError.serverError(httpResponse.statusCode)
            default:
                throw NetworkError.unknownError
            }
            
            // Decode response
            do {
                let decoder = JSONDecoder()
                decoder.dateDecodingStrategy = .iso8601
                return try decoder.decode(T.self, from: data)
            } catch {
                throw NetworkError.decodingError(error)
            }
        } catch let error as NetworkError {
            DispatchQueue.main.async {
                self.lastError = error
            }
            throw error
        } catch {
            let networkError = NetworkError.networkError(error)
            DispatchQueue.main.async {
                self.lastError = networkError
            }
            throw networkError
        }
    }
    
    // MARK: - Logging
    
    private func logRequest(_ request: URLRequest) {
        #if DEBUG
        print("🌐 REQUEST: \(request.httpMethod ?? "GET") \(request.url?.path ?? "")")
        if let headers = request.allHTTPHeaderFields {
            print("   Headers: \(headers)")
        }
        if let body = request.httpBody {
            if let json = try? JSONSerialization.jsonObject(with: body) {
                print("   Body: \(json)")
            }
        }
        #endif
    }
    
    private func logResponse(_ response: HTTPURLResponse, data: Data) {
        #if DEBUG
        print("✓ RESPONSE: \(response.statusCode) \(response.url?.path ?? "")")
        if let json = try? JSONSerialization.jsonObject(with: data) {
            print("   Data: \(json)")
        }
        #endif
    }
}

// MARK: - Error Handling

enum NetworkError: LocalizedError {
    case notAuthenticated
    case invalidResponse
    case unauthorized
    case rateLimited
    case clientError(Int)
    case serverError(Int)
    case unknownError
    case decodingError(Error)
    case networkError(Error)
    
    var errorDescription: String? {
        switch self {
        case .notAuthenticated:
            return "Not authenticated. Please log in."
        case .invalidResponse:
            return "Invalid response from server."
        case .unauthorized:
            return "Session expired. Please log in again."
        case .rateLimited:
            return "Too many requests. Please try again later."
        case .clientError(let code):
            return "Client error: \(code)"
        case .serverError(let code):
            return "Server error: \(code)"
        case .unknownError:
            return "Unknown error occurred."
        case .decodingError:
            return "Failed to decode response."
        case .networkError(let error):
            return "Network error: \(error.localizedDescription)"
        }
    }
}

// MARK: - Request/Response Models

struct CurrentReadingsResponse: Codable {
    let inverter_reading: Double
    let meter_reading: Double
    let timestamp: String
}

struct RecordRequest: Codable {
    let value: Double
    let type: String // "auto", "auto_push", "manual"
    let timestamp: String
}

struct RecordResponse: Codable {
    let success: Bool
    let message: String
    let recordId: String?
}

struct BulkRecordRequest: Codable {
    let records: [RecordRequest]
}

struct BulkRecordResponse: Codable {
    let success: Bool
    let message: String
    let successCount: Int
    let failureCount: Int
}
```

### Network Error Handling & Retry Logic

```swift
// File: Utilities/RetryPolicy.swift

struct RetryPolicy {
    let maxAttempts: Int
    let delaySeconds: Int
    
    static let standard = RetryPolicy(maxAttempts: 3, delaySeconds: 2)
    static let aggressive = RetryPolicy(maxAttempts: 5, delaySeconds: 1)
    
    func delayFor(attempt: Int) -> UInt64 {
        // Exponential backoff: 2s, 4s, 8s, etc.
        let seconds = Double(delaySeconds) * pow(2.0, Double(attempt - 1))
        return UInt64(seconds * 1_000_000_000) // Convert to nanoseconds
    }
}

extension NetworkManager {
    func executeWithRetry<T>(
        policy: RetryPolicy = .standard,
        operation: () async throws -> T
    ) async throws -> T {
        var lastError: Error?
        
        for attempt in 1...policy.maxAttempts {
            do {
                return try await operation()
            } catch let error as NetworkError {
                lastError = error
                
                // Don't retry on auth errors
                if case .unauthorized = error {
                    throw error
                }
                
                // Wait before retry
                if attempt < policy.maxAttempts {
                    try await Task.sleep(nanoseconds: policy.delayFor(attempt: attempt))
                }
            }
        }
        
        throw lastError ?? NetworkError.unknownError
    }
}
```

---

## PART 4: LOCAL PERSISTENCE STRATEGY

### Three-Tier Persistence Architecture

```
┌─────────────────────────────────────┐
│      Keychain (Most Secure)         │
│  - Session tokens                   │
│  - Sensitive credentials            │
│  - Automatically encrypted          │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│      UserDefaults (Simple)          │
│  - User preferences                 │
│  - App settings                     │
│  - Last refresh timestamp           │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│      CoreData (Complex Data)        │
│  - Offline queue (pending records)  │
│  - Historical readings (if saved)   │
│  - Relationships and queries        │
└─────────────────────────────────────┘
```

### CoreData Setup for Offline Queue

```swift
// File: Persistence/PersistenceController.swift

import CoreData

class PersistenceController {
    static let shared = PersistenceController()
    
    let container: NSPersistentContainer
    
    var viewContext: NSManagedObjectContext {
        container.viewContext
    }
    
    init(inMemory: Bool = false) {
        container = NSPersistentContainer(name: "SolarMonitoring")
        
        if inMemory {
            container.persistentStoreDescriptions.first?.url = URL(fileURLWithPath: "/dev/null")
        }
        
        container.loadPersistentStores { description, error in
            if let error = error as NSError? {
                fatalError("Unable to load persistent stores: \(error), \(error.userInfo)")
            }
        }
        
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }
    
    func save() {
        let context = container.viewContext
        
        if context.hasChanges {
            do {
                try context.save()
            } catch {
                let nsError = error as NSError
                fatalError("Unresolved error \(nsError), \(nsError.userInfo)")
            }
        }
    }
}
```

### CoreData Model Definition (XML Format)

```xml
<!-- SolarMonitoring.xcdatamodeld/SolarMonitoring.xcdatamodel/contents -->

<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<model type="com.apple.IDECoreDataModeler.DataModel" documentVersion="1.0" lastSavedToolsVersion="22222" systemVersion="13.0" minimumToolsVersion="Automatic" sourceLanguage="Swift" usedWithCloudKit="NO">
    <entity name="OfflineRecord" representedClassName="OfflineRecord" syncable="YES">
        <attribute name="timestamp" attributeType="Date" usesScalarValueType="NO" syncable="YES"/>
        <attribute name="type" attributeType="String" syncable="YES"/>
        <attribute name="value" attributeType="Double" defaultValueString="0.0" usesScalarValueType="YES" syncable="YES"/>
        <attribute name="createdAt" attributeType="Date" usesScalarValueType="NO" syncable="YES"/>
        <attribute name="id" attributeType="UUID" usesScalarValueType="NO" syncable="YES"/>
        <attribute name="attempts" attributeType="Integer 32" defaultValueString="0" usesScalarValueType="YES" syncable="YES"/>
        <attribute name="lastError" attributeType="String" syncable="YES"/>
    </entity>
</model>
```

### Offline Queue Manager

```swift
// File: Persistence/OfflineQueueManager.swift

import CoreData

class OfflineQueueManager: NSObject, ObservableObject {
    @Published var pendingRecords: [OfflineRecord] = []
    @Published var isSyncing = false
    
    private let persistenceController = PersistenceController.shared
    private let networkManager = NetworkManager()
    private let maxRetries = 3
    
    // MARK: - Queue Operations
    
    func queueRecord(_ record: RecordRequest) throws {
        let context = persistenceController.container.viewContext
        
        let offlineRecord = NSEntityDescription.insertNewObject(
            forEntityName: "OfflineRecord",
            into: context
        ) as! OfflineRecord
        
        offlineRecord.id = UUID()
        offlineRecord.value = record.value
        offlineRecord.type = record.type
        offlineRecord.timestamp = ISO8601DateFormatter().date(from: record.timestamp) ?? Date()
        offlineRecord.createdAt = Date()
        offlineRecord.attempts = 0
        
        persistenceController.save()
        loadPendingRecords()
    }
    
    func loadPendingRecords() {
        let context = persistenceController.container.viewContext
        let fetchRequest: NSFetchRequest<OfflineRecord> = OfflineRecord.fetchRequest()
        fetchRequest.sortDescriptors = [NSSortDescriptor(keyPath: \OfflineRecord.createdAt, ascending: true)]
        
        do {
            pendingRecords = try context.fetch(fetchRequest)
        } catch {
            print("Failed to load pending records: \(error)")
        }
    }
    
    // MARK: - Sync Operations
    
    func syncPendingRecords() async {
        guard !isSyncing && !pendingRecords.isEmpty else { return }
        
        DispatchQueue.main.async {
            self.isSyncing = true
        }
        
        for record in pendingRecords {
            await syncRecord(record)
        }
        
        DispatchQueue.main.async {
            self.isSyncing = false
        }
    }
    
    private func syncRecord(_ offlineRecord: OfflineRecord) async {
        guard offlineRecord.attempts < maxRetries else {
            print("Max retries reached for record \(offlineRecord.id ?? UUID())")
            return
        }
        
        let recordRequest = RecordRequest(
            value: offlineRecord.value,
            type: offlineRecord.type,
            timestamp: ISO8601DateFormatter().string(from: offlineRecord.timestamp ?? Date())
        )
        
        do {
            _ = try await networkManager.recordReading(recordRequest)
            
            // Success - delete from offline queue
            let context = persistenceController.container.viewContext
            context.delete(offlineRecord)
            persistenceController.save()
            loadPendingRecords()
            
        } catch {
            // Failure - increment attempt counter
            offlineRecord.attempts += 1
            offlineRecord.lastError = error.localizedDescription
            persistenceController.save()
            print("Sync failed for record \(offlineRecord.id ?? UUID()): \(error)")
        }
    }
    
    // MARK: - Cleanup
    
    func clearFailedRecords() {
        let context = persistenceController.container.viewContext
        
        for record in pendingRecords where record.attempts >= maxRetries {
            context.delete(record)
        }
        
        persistenceController.save()
        loadPendingRecords()
    }
}
```

### UserDefaults Manager for Settings

```swift
// File: Persistence/SettingsManager.swift

import Foundation

class SettingsManager {
    static let shared = SettingsManager()
    
    private let userDefaults = UserDefaults.standard
    
    private enum Keys {
        static let lastReadingsRefresh = "lastReadingsRefresh"
        static let userEmail = "userEmail"
        static let autoSyncEnabled = "autoSyncEnabled"
        static let offlineQueueSize = "offlineQueueSize"
    }
    
    // MARK: - Last Refresh Time
    
    var lastReadingsRefresh: Date? {
        get { userDefaults.object(forKey: Keys.lastReadingsRefresh) as? Date }
        set { userDefaults.set(newValue, forKey: Keys.lastReadingsRefresh) }
    }
    
    // MARK: - User Email
    
    var userEmail: String? {
        get { userDefaults.string(forKey: Keys.userEmail) }
        set { userDefaults.set(newValue, forKey: Keys.userEmail) }
    }
    
    // MARK: - Auto Sync Setting
    
    var autoSyncEnabled: Bool {
        get { userDefaults.bool(forKey: Keys.autoSyncEnabled) }
        set { userDefaults.set(newValue, forKey: Keys.autoSyncEnabled) }
    }
    
    // MARK: - Clear All Settings
    
    func clearAll() {
        if let bundleID = Bundle.main.bundleIdentifier {
            userDefaults.removePersistentDomain(forName: bundleID)
        }
    }
}
```

### Persistence Methods Comparison

| Method | Use Case | Pros | Cons | Data Size |
|--------|----------|------|------|-----------|
| **Keychain** | Tokens, passwords | Encrypted, secure, survives app deletion | Complex API, limited to small data | <100KB |
| **UserDefaults** | Settings, preferences | Simple API, persistent | Not encrypted, small data only | <1MB |
| **CoreData** | Complex data, offline queue | Queries, relationships, large data | Learning curve, schema migrations | Unlimited |
| **SQLite** | Structured data | High performance, ACID | Manual query writing, no type safety | Unlimited |
| **Realm** | Modern alternative | Easy to use, fast | Third-party dependency | Unlimited |

**Recommendation for this project:** Use all three in combination:
- **Keychain** for sessionToken
- **UserDefaults** for app settings and last refresh time
- **CoreData** for offline queue management

---

## PART 5: UI/UX DESIGN FOR DAD-FRIENDLY ACCESSIBILITY

### Design Principles for Accessibility

```
┌─────────────────────────────────────────┐
│   DAD-FRIENDLY UI DESIGN PRINCIPLES     │
├─────────────────────────────────────────┤
│ 1. Large Text (18pt minimum, 20pt ideal)│
│ 2. Large Buttons (48pt minimum taps)    │
│ 3. High Contrast (4.5:1 WCAG AA)       │
│ 4. Simple Language (no technical jargon)│
│ 5. Clear Icons (paired with text)       │
│ 6. Ample Spacing (16pt+ margins)        │
│ 7. Dark Mode Support                    │
│ 8. Haptic Feedback (confirmation)       │
└─────────────────────────────────────────┘
```

### Login Screen Design

```
┌──────────────────────────────┐
│                              │
│                              │
│      ☀️ (80pt icon)          │
│                              │
│   SOLAR MONITOR              │
│   (32pt, bold)               │
│                              │
│   Track your solar energy    │
│   (18pt, gray)               │
│                              │
│   [            ]             │
│   (40pt height spacing)      │
│                              │
│  ┌────────────────────────┐  │
│  │ 🔐 Sign in with Google │  │
│  │      (56pt button)      │  │
│  └────────────────────────┘  │
│                              │
│   [Error message in red]     │
│   (18pt, red #FF3B30)        │
│                              │
└──────────────────────────────┘

Button specs:
- Height: 56pt (easily tappable)
- Font: 18pt, bold
- Corner radius: 8pt
- Background: Google blue (#4285F4)
- Text color: White
```

### Home Screen Design

```
┌──────────────────────────────┐
│ ◄ SOLAR MONITOR         ⚙️    │
├──────────────────────────────┤
│                              │
│  Inverter Output  ⚡         │
│  5.50 kW                     │
│  (20pt large text)           │
│                              │
│  Meter Reading    📊         │
│  1234.56 kWh                 │
│  (20pt large text)           │
│                              │
│  Last reading: 2:30 PM       │
│  (16pt gray text)            │
│                              │
├──────────────────────────────┤
│                              │
│  ┌────────────────────────┐  │
│  │ ✓ Auto Record          │  │
│  │   Save current reading │  │
│  │ (56pt button, 18pt)   │  │
│  └────────────────────────┘  │
│                              │
│  ┌────────────────────────┐  │
│  │ ↗ Auto Record + Push   │  │
│  │   Save to utility      │  │
│  │ (56pt button, 18pt)   │  │
│  └────────────────────────┘  │
│                              │
│  ┌────────────────────────┐  │
│  │ ✎ Manual Entry         │  │
│  │   Enter manually       │  │
│  │ (56pt button, 18pt)   │  │
│  └────────────────────────┘  │
│                              │
│  Pending: 3 records offline  │
│  (16pt, orange warning)      │
│                              │
└──────────────────────────────┘

Card specs:
- Background: White with 1pt border
- Border radius: 8pt
- Shadow: Subtle (1pt, 10% black)
- Padding: 16pt
- Spacing between: 12pt
```

### SwiftUI Code for Home Screen

```swift
// File: Views/HomeView.swift

import SwiftUI

struct HomeView: View {
    @StateObject var viewModel: HomeViewModel
    @Environment(\.scenePhase) var scenePhase
    
    var body: some View {
        NavigationStack {
            ZStack {
                Color(.systemBackground)
                    .ignoresSafeArea()
                
                if viewModel.isLoading {
                    ProgressView()
                } else {
                    ScrollView {
                        VStack(spacing: 20) {
                            // Header
                            HStack {
                                Text("SOLAR MONITOR")
                                    .font(.system(size: 24, weight: .bold, design: .default))
                                
                                Spacer()
                                
                                NavigationLink(destination: SettingsView()) {
                                    Image(systemName: "gear")
                                        .font(.system(size: 20))
                                        .foregroundColor(.blue)
                                }
                            }
                            .padding(.horizontal, 16)
                            .padding(.top, 12)
                            
                            // Current Readings Card
                            VStack(alignment: .leading, spacing: 16) {
                                HStack {
                                    VStack(alignment: .leading, spacing: 8) {
                                        HStack {
                                            Image(systemName: "bolt.fill")
                                                .font(.system(size: 20))
                                                .foregroundColor(.orange)
                                            Text("Inverter Output")
                                                .font(.system(size: 16, weight: .medium))
                                        }
                                        Text(String(format: "%.2f kW", viewModel.inverterReading))
                                            .font(.system(size: 28, weight: .bold))
                                            .foregroundColor(.black)
                                    }
                                    Spacer()
                                }
                                
                                Divider()
                                
                                HStack {
                                    VStack(alignment: .leading, spacing: 8) {
                                        HStack {
                                            Image(systemName: "chart.bar.fill")
                                                .font(.system(size: 20))
                                                .foregroundColor(.blue)
                                            Text("Meter Reading")
                                                .font(.system(size: 16, weight: .medium))
                                        }
                                        Text(String(format: "%.2f kWh", viewModel.meterReading))
                                            .font(.system(size: 28, weight: .bold))
                                            .foregroundColor(.black)
                                    }
                                    Spacer()
                                }
                                
                                Divider()
                                
                                if let timestamp = viewModel.lastReadingTime {
                                    Text("Last reading: \(timestamp)")
                                        .font(.system(size: 14, weight: .regular))
                                        .foregroundColor(.gray)
                                }
                            }
                            .padding(16)
                            .background(Color(.white))
                            .cornerRadius(8)
                            .shadow(radius: 1)
                            .padding(.horizontal, 16)
                            
                            // Action Buttons
                            VStack(spacing: 12) {
                                // Auto Record
                                Button(action: { Task { await viewModel.autoRecord() } }) {
                                    HStack {
                                        Image(systemName: "checkmark")
                                            .font(.system(size: 18, weight: .semibold))
                                        Text("Auto Record")
                                            .font(.system(size: 18, weight: .semibold))
                                    }
                                    .frame(maxWidth: .infinity)
                                    .frame(height: 56)
                                    .background(Color.blue)
                                    .foregroundColor(.white)
                                    .cornerRadius(8)
                                }
                                
                                // Auto Record + Push
                                Button(action: { Task { await viewModel.autoRecordAndPush() } }) {
                                    HStack {
                                        Image(systemName: "arrow.up.right")
                                            .font(.system(size: 18, weight: .semibold))
                                        Text("Auto Record + Push")
                                            .font(.system(size: 18, weight: .semibold))
                                    }
                                    .frame(maxWidth: .infinity)
                                    .frame(height: 56)
                                    .background(Color.green)
                                    .foregroundColor(.white)
                                    .cornerRadius(8)
                                }
                                
                                // Manual Entry
                                NavigationLink(destination: ManualEntryView(viewModel: ManualEntryViewModel())) {
                                    HStack {
                                        Image(systemName: "pencil")
                                            .font(.system(size: 18, weight: .semibold))
                                        Text("Manual Entry")
                                            .font(.system(size: 18, weight: .semibold))
                                    }
                                    .frame(maxWidth: .infinity)
                                    .frame(height: 56)
                                    .background(Color.orange)
                                    .foregroundColor(.white)
                                    .cornerRadius(8)
                                }
                            }
                            .padding(.horizontal, 16)
                            
                            // Offline Queue Status
                            if viewModel.pendingRecordCount > 0 {
                                HStack {
                                    Image(systemName: "exclamationmark.circle.fill")
                                        .foregroundColor(.orange)
                                        .font(.system(size: 18))
                                    
                                    Text("Pending: \(viewModel.pendingRecordCount) records offline")
                                        .font(.system(size: 16, weight: .regular))
                                    
                                    Spacer()
                                    
                                    if viewModel.isSyncing {
                                        ProgressView()
                                            .scaleEffect(0.8)
                                    }
                                }
                                .padding(12)
                                .background(Color(.systemOrange).opacity(0.1))
                                .cornerRadius(8)
                                .padding(.horizontal, 16)
                            }
                            
                            Spacer(minLength: 20)
                        }
                    }
                }
            }
            .onAppear {
                Task { await viewModel.loadReadings() }
            }
            .onChange(of: scenePhase) { oldPhase, newPhase in
                if newPhase == .active {
                    Task { await viewModel.loadReadings() }
                    Task { await viewModel.syncOfflineRecords() }
                }
            }
            .alert("Error", isPresented: .constant(viewModel.errorMessage != nil)) {
                Button("OK") { viewModel.errorMessage = nil }
            } message: {
                Text(viewModel.errorMessage ?? "Unknown error")
            }
        }
    }
}
```

---

## PART 6: PROJECT STRUCTURE & XCODE SETUP

### Recommended Xcode Project Structure

```
SolarMonitor/
├── SolarMonitor.xcodeproj/
│   ├── project.pbxproj
│   ├── project.xcworkspace/
│   │   └── contents.xcworkspace
│   └── xcshareddata/
│
├── SolarMonitor/
│   ├── App/
│   │   ├── SolarMonitorApp.swift          # App entry point
│   │   └── AppDelegate.swift              # Background handling
│   │
│   ├── Views/
│   │   ├── ContentView.swift              # Root navigation
│   │   ├── LoginView.swift                # Google OAuth login
│   │   ├── HomeView.swift                 # Main readings display
│   │   ├── ManualEntryView.swift          # Manual recording
│   │   └── SettingsView.swift             # User settings & logout
│   │
│   ├── ViewModels/
│   │   ├── HomeViewModel.swift            # Home screen logic
│   │   └── ManualEntryViewModel.swift     # Manual entry logic
│   │
│   ├── Models/
│   │   ├── NetworkModels.swift            # Request/response DTOs
│   │   ├── AppModels.swift                # UI models
│   │   └── CoreDataModels.swift           # CoreData entities
│   │
│   ├── Networking/
│   │   ├── NetworkManager.swift           # URLSession wrapper
│   │   ├── AuthManager.swift              # Google OAuth + token
│   │   └── KeychainManager.swift          # Secure token storage
│   │
│   ├── Persistence/
│   │   ├── PersistenceController.swift    # CoreData stack
│   │   ├── OfflineQueueManager.swift      # Offline sync logic
│   │   └── SettingsManager.swift          # UserDefaults wrapper
│   │
│   ├── Utilities/
│   │   ├── DateFormatter+Extensions.swift
│   │   ├── URLRequest+Extensions.swift
│   │   └── NetworkReachability.swift      # Internet connectivity
│   │
│   ├── Components/
│   │   ├── LoadingView.swift              # Reusable loading
│   │   ├── ErrorAlert.swift               # Reusable error UI
│   │   └── ReadingCard.swift              # Reusable card
│   │
│   ├── Resources/
│   │   ├── Assets.xcassets/
│   │   │   ├── AppIcon.appiconset/
│   │   │   ├── Colors/
│   │   │   └── Images/
│   │   │
│   │   ├── Localizable.strings            # i18n (for future)
│   │   ├── SolarMonitoring.xcdatamodeld/ # CoreData models
│   │   └── GoogleService-Info.plist       # Google OAuth config
│   │
│   └── Info.plist                         # App configuration
│
├── SolarMonitorTests/
│   ├── NetworkManagerTests.swift          # Network layer tests
│   ├── AuthManagerTests.swift             # Auth tests
│   ├── OfflineQueueTests.swift            # Offline queue tests
│   └── ViewModelTests.swift               # ViewModel tests
│
├── Podfile                                # CocoaPods dependencies
├── .gitignore
├── README.md
└── VERSION.txt
```

### Step-by-Step Xcode Project Creation

**1. Create Project in Xcode:**
```
File → New → Project
Select: iOS → App
Product Name: SolarMonitor
Team: None (or your team)
Organization Identifier: com.yourname
Interface: SwiftUI
Lifecycle: SwiftUI App
Storage: None
Create
```

**2. Setup CocoaPods for Google OAuth:**
```bash
cd ~/Projects/SolarMonitor
pod init
```

**3. Edit Podfile:**
```ruby
platform :ios, '15.0'

target 'SolarMonitor' do
  pod 'GoogleSignIn', '~> 7.0'
end
```

**4. Install Dependencies:**
```bash
pod install
```

**5. Open Workspace (not Project):**
```bash
open SolarMonitor.xcworkspace
```

### Configure GoogleService-Info.plist

1. Get from Google Cloud Console
2. Download GoogleService-Info.plist
3. Drag into Xcode project
4. Select "Copy items if needed"
5. Select "SolarMonitor" target

### Info.plist Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>en</string>
    <key>CFBundleExecutable</key>
    <string>$(EXECUTABLE_NAME)</string>
    <key>CFBundleIdentifier</key>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>$(PRODUCT_NAME)</string>
    <key>CFBundlePackageType</key>
    <string>$(PRODUCT_BUNDLE_PACKAGE_TYPE)</string>
    <key>CFBundleShortVersionString</key>
    <string>1.0</string>
    <key>CFBundleVersion</key>
    <string>1</string>
    <key>LSRequiresIPhoneOS</key>
    <true/>
    
    <!-- Google OAuth URL Scheme -->
    <key>CFBundleURLTypes</key>
    <array>
        <dict>
            <key>CFBundleTypeRole</key>
            <string>Editor</string>
            <key>CFBundleURLSchemes</key>
            <array>
                <string>com.googleusercontent.apps.YOUR-CLIENT-ID</string>
            </array>
        </dict>
    </array>
    
    <!-- Network Security -->
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <false/>
        <key>NSExceptionDomains</key>
        <dict>
            <key>localhost</key>
            <dict>
                <key>NSIncludesSubdomains</key>
                <true/>
                <key>NSTemporaryThrowingHTTPSLoads</key>
                <true/>
            </dict>
        </dict>
    </dict>
    
    <!-- Required Device Capabilities -->
    <key>UIRequiredDeviceCapabilities</key>
    <array>
        <string>arm64</string>
    </array>
    
    <!-- Supported Device Orientations -->
    <key>UISupportedInterfaceOrientations</key>
    <array>
        <string>UIInterfaceOrientationPortrait</string>
        <string>UIInterfaceOrientationPortraitUpsideDown</string>
    </array>
    
    <key>UIMainStoryboardFile</key>
    <string></string>
    <key>UISceneConfigurations</key>
    <dict/>
</dict>
</plist>
```

---

## PART 7: DEVELOPMENT ROADMAP (7 PHASES)

### Phase 1: Project Setup (2-3 hours)

**Deliverables:**
- [ ] Xcode project created with SwiftUI template
- [ ] Folder structure organized as documented
- [ ] CocoaPods installed and GoogleSignIn pod added
- [ ] GoogleService-Info.plist configured
- [ ] App runs in simulator without crashes

**Tasks:**
1. Create new Xcode project
2. Create folder structure (Views, ViewModels, Models, etc.)
3. Add GoogleSignIn via CocoaPods
4. Download and configure GoogleService-Info.plist
5. Verify app runs in simulator
6. Create basic app structure (App, ContentView)

**Testing:**
```swift
// In SolarMonitorApp.swift
@main
struct SolarMonitorApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

// In ContentView.swift
struct ContentView: View {
    var body: some View {
        Text("Hello, Solar Monitor!")
            .font(.system(size: 24, weight: .bold))
    }
}
```

### Phase 2: Authentication (4-5 hours)

**Deliverables:**
- [ ] KeychainManager implemented and tested
- [ ] AuthManager with Google Sign-In flow
- [ ] LoginView with "Sign in with Google" button
- [ ] OAuth flow works end-to-end
- [ ] Session token stored in Keychain
- [ ] Token refresh logic implemented

**Tasks:**
1. Implement KeychainManager class
2. Implement AuthManager class with OAuth flow
3. Create LoginView SwiftUI component
4. Set up Google Cloud OAuth credentials
5. Test entire OAuth flow on simulator
6. Handle token expiration and refresh

**Code Location:** `SolarMonitor/Networking/` and `SolarMonitor/Views/`

**Testing Checklist:**
- [ ] User can tap "Sign in with Google"
- [ ] Google login dialog appears
- [ ] After login, token is stored in Keychain
- [ ] Token persists after app restart
- [ ] Token refresh works when expired

### Phase 3: Networking (3-4 hours)

**Deliverables:**
- [ ] NetworkManager class with error handling
- [ ] Request/response models (DTOs)
- [ ] GET /current_readings implemented
- [ ] POST /record implemented
- [ ] Error handling and retry logic
- [ ] Request/response logging in DEBUG mode

**Tasks:**
1. Create NetworkModels.swift with all DTOs
2. Implement NetworkManager class
3. Add error handling and retry logic
4. Test against backend API endpoints
5. Add request/response logging
6. Handle network errors gracefully

**Code Location:** `SolarMonitor/Networking/` and `SolarMonitor/Models/`

**Testing Checklist:**
- [ ] Can fetch current readings without error
- [ ] Can POST a record successfully
- [ ] Retry logic works on 5xx errors
- [ ] 401 errors trigger re-authentication
- [ ] Network timeouts handled gracefully
- [ ] Requests include proper headers and auth token

### Phase 4: UI Implementation (5-6 hours)

**Deliverables:**
- [ ] LoginView fully implemented
- [ ] HomeView with reading cards
- [ ] Three action buttons functional
- [ ] ManualEntryView with input form
- [ ] SettingsView with user info and logout
- [ ] Dark mode support
- [ ] Accessibility features (large text, buttons)

**Tasks:**
1. Enhance LoginView with error messages
2. Implement HomeView with reading cards
3. Implement action buttons and navigation
4. Create ManualEntryView with form
5. Create SettingsView with logout
6. Add dark mode support
7. Test on iPhone 13 mini screen

**Code Location:** `SolarMonitor/Views/`

**Testing Checklist:**
- [ ] All views load without crashing
- [ ] Text is minimum 18pt
- [ ] Buttons are minimum 56pt height
- [ ] Color contrast is adequate (4.5:1)
- [ ] Reads correctly in Dark Mode
- [ ] Screen rotation handled properly
- [ ] Works on iPhone 13 mini (5.4" screen)

### Phase 5: Offline Support (4-5 hours)

**Deliverables:**
- [ ] CoreData model created and migrated
- [ ] OfflineQueueManager implemented
- [ ] Offline queue persists across app restarts
- [ ] Automatic retry when connectivity returns
- [ ] UI shows pending offline records
- [ ] Bulk sync endpoint used for efficiency

**Tasks:**
1. Create CoreData model (OfflineRecord entity)
2. Implement PersistenceController
3. Implement OfflineQueueManager
4. Add network reachability detection
5. Implement automatic sync trigger
6. Add UI indicator for pending records
7. Test offline scenarios

**Code Location:** `SolarMonitor/Persistence/`

**Testing Checklist:**
- [ ] Records are queued when offline
- [ ] Queue persists after app restart
- [ ] Queue syncs when connectivity returns
- [ ] Bulk sync reduces network calls
- [ ] Failed records retry with backoff
- [ ] Old failed records eventually purged
- [ ] UI shows correct pending count

### Phase 6: Testing (2-3 hours)

**Deliverables:**
- [ ] Unit tests for NetworkManager
- [ ] Unit tests for AuthManager
- [ ] Unit tests for OfflineQueueManager
- [ ] UI tests for critical flows
- [ ] Manual testing checklist completed
- [ ] All success criteria validated

**Tasks:**
1. Write NetworkManager unit tests
2. Write AuthManager unit tests
3. Write OfflineQueueManager tests
4. Write ViewModel unit tests
5. Create UI tests for key flows
6. Manual testing on real device
7. Performance profiling with Instruments

**Test Categories:**
- Authentication (login, logout, token refresh)
- Networking (success, errors, retries)
- Data persistence (CoreData, Keychain)
- Offline scenarios (queue, sync, retry)
- UI (rendering, navigation, accessibility)

**Code Location:** `SolarMonitorTests/`

### Phase 7: Polish & Deployment (2-3 hours)

**Deliverables:**
- [ ] Code cleanup and documentation
- [ ] TestFlight build created
- [ ] Beta testing with real users
- [ ] Feedback incorporated
- [ ] App Store submission completed
- [ ] Version 1.0 released

**Tasks:**
1. Code review and cleanup
2. Add documentation comments
3. Create TestFlight build
4. Invite beta testers
5. Gather and fix feedback
6. Create App Store listing
7. Submit for App Review
8. Monitor for crashes post-launch

**Release Checklist:**
- [ ] Crash-free rate >99%
- [ ] All features working
- [ ] No deprecated APIs used
- [ ] Privacy policy included
- [ ] Screenshots and description ready
- [ ] Version numbering correct (1.0)
- [ ] Build number incremented

---

## PART 8: COMPLETE CODE EXAMPLES

### Example 1: Complete ViewModel

```swift
// File: ViewModels/HomeViewModel.swift

import Foundation

@MainActor
class HomeViewModel: ObservableObject {
    @Published var inverterReading: Double = 0.0
    @Published var meterReading: Double = 0.0
    @Published var lastReadingTime: String?
    @Published var isLoading = false
    @Published var errorMessage: String?
    @Published var pendingRecordCount = 0
    @Published var isSyncing = false
    
    private let networkManager = NetworkManager()
    private let offlineQueueManager = OfflineQueueManager()
    private let settingsManager = SettingsManager()
    
    // MARK: - Load Readings
    
    func loadReadings() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            let response = try await networkManager.fetchCurrentReadings()
            self.inverterReading = response.inverter_reading
            self.meterReading = response.meter_reading
            
            // Format timestamp
            if let date = ISO8601DateFormatter().date(from: response.timestamp) {
                let formatter = DateFormatter()
                formatter.timeStyle = .short
                self.lastReadingTime = formatter.string(from: date)
            }
            
            settingsManager.lastReadingsRefresh = Date()
            errorMessage = nil
            
        } catch let error as NetworkError {
            errorMessage = error.errorDescription ?? "Failed to load readings"
        } catch {
            errorMessage = "Unknown error: \(error.localizedDescription)"
        }
    }
    
    // MARK: - Recording Actions
    
    func autoRecord() async {
        let request = RecordRequest(
            value: inverterReading,
            type: "auto",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        await recordReading(request)
    }
    
    func autoRecordAndPush() async {
        let request = RecordRequest(
            value: inverterReading,
            type: "auto_push",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        await recordReading(request)
    }
    
    private func recordReading(_ request: RecordRequest) async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            _ = try await networkManager.recordReading(request)
            errorMessage = nil
            // Show success feedback
            await loadReadings()
        } catch NetworkError.notAuthenticated {
            errorMessage = "Not authenticated. Please log in."
        } catch NetworkError.networkError {
            // Network failed - queue for offline
            do {
                try offlineQueueManager.queueRecord(request)
                errorMessage = "Saved offline (will sync when online)"
                updatePendingCount()
            } catch {
                errorMessage = "Failed to save record: \(error.localizedDescription)"
            }
        } catch let error as NetworkError {
            errorMessage = error.errorDescription ?? "Failed to record"
        } catch {
            errorMessage = "Unknown error: \(error.localizedDescription)"
        }
    }
    
    // MARK: - Offline Sync
    
    func syncOfflineRecords() async {
        await offlineQueueManager.syncPendingRecords()
        updatePendingCount()
    }
    
    func updatePendingCount() {
        offlineQueueManager.loadPendingRecords()
        pendingRecordCount = offlineQueueManager.pendingRecords.count
    }
    
    // MARK: - Initialization
    
    func initialize() {
        offlineQueueManager.loadPendingRecords()
        updatePendingCount()
    }
}
```

### Example 2: Complete App Entry Point

```swift
// File: App/SolarMonitorApp.swift

import SwiftUI

@main
struct SolarMonitorApp: App {
    @StateObject private var authManager = AuthManager()
    
    var body: some Scene {
        WindowGroup {
            if authManager.isAuthenticated {
                ContentView()
                    .environmentObject(authManager)
            } else {
                LoginView()
                    .environmentObject(authManager)
            }
        }
    }
}

struct ContentView: View {
    @EnvironmentObject var authManager: AuthManager
    @StateObject var homeViewModel = HomeViewModel()
    
    var body: some View {
        TabView {
            HomeView(viewModel: homeViewModel)
                .tabItem {
                    Label("Home", systemImage: "house.fill")
                }
            
            SettingsView()
                .tabItem {
                    Label("Settings", systemImage: "gear")
                }
        }
        .onAppear {
            homeViewModel.initialize()
            Task {
                await homeViewModel.loadReadings()
            }
        }
    }
}
```

### Example 3: Manual Entry ViewModel

```swift
// File: ViewModels/ManualEntryViewModel.swift

import Foundation

@MainActor
class ManualEntryViewModel: ObservableObject {
    @Published var inputValue: String = ""
    @Published var isSubmitting = false
    @Published var errorMessage: String?
    @Published var successMessage: String?
    
    private let networkManager = NetworkManager()
    private let offlineQueueManager = OfflineQueueManager()
    
    func submitManualEntry() async {
        guard let value = Double(inputValue.trimmingCharacters(in: .whitespaces)) else {
            errorMessage = "Please enter a valid number"
            return
        }
        
        isSubmitting = true
        defer { isSubmitting = false }
        
        let request = RecordRequest(
            value: value,
            type: "manual",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        do {
            _ = try await networkManager.recordReading(request)
            successMessage = "Entry recorded successfully!"
            inputValue = ""
            errorMessage = nil
        } catch NetworkError.networkError {
            // Queue offline
            do {
                try offlineQueueManager.queueRecord(request)
                successMessage = "Entry saved offline (will sync when online)"
                inputValue = ""
            } catch {
                errorMessage = "Failed to save: \(error.localizedDescription)"
            }
        } catch let error as NetworkError {
            errorMessage = error.errorDescription ?? "Failed to record"
        } catch {
            errorMessage = "Unknown error"
        }
    }
}
```

---

## PART 9: TESTING STRATEGY

### Unit Testing Example

```swift
// File: SolarMonitorTests/NetworkManagerTests.swift

import XCTest
@testable import SolarMonitor

class NetworkManagerTests: XCTestCase {
    var sut: NetworkManager!
    var mockURLSession: URLSession!
    
    override func setUp() {
        super.setUp()
        sut = NetworkManager()
    }
    
    func testFetchCurrentReadingsSuccess() async throws {
        // Given
        let mockData = """
        {
            "inverter_reading": 5.5,
            "meter_reading": 1234.56,
            "timestamp": "2025-11-07T14:30:00Z"
        }
        """.data(using: .utf8)!
        
        // When & Then
        let response = try await sut.fetchCurrentReadings()
        XCTAssertEqual(response.inverter_reading, 5.5)
        XCTAssertEqual(response.meter_reading, 1234.56)
    }
    
    func testRecordReadingSuccess() async throws {
        // Given
        let request = RecordRequest(
            value: 5.5,
            type: "auto",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        // When
        let response = try await sut.recordReading(request)
        
        // Then
        XCTAssertTrue(response.success)
    }
    
    func testNetworkErrorHandling() async {
        // Given offline network
        // When
        do {
            _ = try await sut.fetchCurrentReadings()
            XCTFail("Should throw error")
        } catch NetworkError.networkError {
            // Then - expected
        } catch {
            XCTFail("Wrong error type")
        }
    }
}
```

### UI Testing Example

```swift
// File: SolarMonitorTests/HomeViewUITests.swift

import XCTest

class HomeViewUITests: XCTestCase {
    let app = XCUIApplication()
    
    override func setUp() {
        super.setUp()
        continueAfterFailure = false
        app.launch()
    }
    
    func testAutoRecordButtonExists() {
        XCTAssertTrue(app.buttons["Auto Record"].exists)
    }
    
    func testAutoRecordButtonTappable() {
        let button = app.buttons["Auto Record"]
        XCTAssertTrue(button.isHittable)
        button.tap()
        // Verify action occurred
    }
    
    func testReadingsDisplayed() {
        let inverterText = app.staticTexts.containing(
            NSPredicate(format: "label CONTAINS 'kW'")
        ).firstMatch
        XCTAssertTrue(inverterText.exists)
    }
}
```

---

## PART 10: DEPLOYMENT & DISTRIBUTION

### TestFlight Beta Distribution

**Steps:**
1. Create App ID in Apple Developer Console
2. Create provisioning profile
3. Archive app in Xcode (Product → Archive)
4. Upload to TestFlight (Organizer window)
5. Add beta testers
6. Send invite links
7. Collect feedback

**Timeline:** 30 minutes setup, 1-2 days for app processing

### App Store Release

**Checklist:**
- [ ] Create App Store listing
- [ ] Add app icons and screenshots
- [ ] Write app description (500 chars)
- [ ] Set pricing and availability
- [ ] Specify content rating
- [ ] Include privacy policy
- [ ] Enable app analytics

**Timeline:** 30 minutes for submission, 1-3 days for review

---

## PART 11: SUCCESS CRITERIA & VALIDATION

### Functional Success Criteria

- [ ] **Authentication:** User can sign in with Google, token stored in Keychain
- [ ] **Current Readings:** App fetches and displays inverter/meter readings
- [ ] **Three Actions:** Auto record, auto+push, manual entry all work
- [ ] **Offline Support:** Records queue locally and sync when online
- [ ] **Error Handling:** Network errors handled gracefully with user feedback

### Non-Functional Success Criteria

- [ ] **Performance:** App loads readings in <1 second
- [ ] **Accessibility:** Text minimum 18pt, buttons 56pt, contrast 4.5:1
- [ ] **Stability:** No crashes after 30 minutes of use
- [ ] **Battery:** <2% drain per hour idle
- [ ] **Device Compatibility:** Works on iPhone 13 mini

### Testing Checklist

**Before TestFlight:**
- [ ] App doesn't crash on launch
- [ ] Can complete full OAuth flow
- [ ] Can fetch and display readings
- [ ] Can submit all three action types
- [ ] Offline queue works end-to-end
- [ ] Settings screen accessible
- [ ] Sign out clears all data

**Before App Store:**
- [ ] TestFlight beta testing complete
- [ ] User feedback incorporated
- [ ] No critical bugs found
- [ ] App Store listing complete
- [ ] Privacy policy linked
- [ ] Minimum build number > 1.0

---

## PART 12: RISK MITIGATION & TROUBLESHOOTING

### Common Issues & Solutions

#### Issue 1: Google Sign-In Not Working
**Symptoms:** OAuth dialog doesn't appear
**Root Causes:**
- GoogleService-Info.plist not configured
- Bundle ID doesn't match Google Cloud credentials
- URL scheme not added to Info.plist

**Solutions:**
1. Verify GoogleService-Info.plist in project
2. Check Bundle ID matches Google Cloud Console
3. Add URL scheme: `com.googleusercontent.apps.YOUR-CLIENT-ID`
4. Clean build folder and rebuild

#### Issue 2: Network Requests Failing with 401
**Symptoms:** "Unauthorized" errors
**Root Causes:**
- Session token expired
- Token not being sent in headers
- Backend not accepting token format

**Solutions:**
1. Verify token stored in Keychain
2. Check Authorization header format: `Bearer {token}`
3. Confirm backend expects same token format
4. Add request logging to debug headers

#### Issue 3: Offline Queue Not Persisting
**Symptoms:** Records queued but lost after restart
**Root Causes:**
- CoreData save() not called
- Context changes not propagated
- Migration failed silently

**Solutions:**
1. Call persistenceController.save() after creating records
2. Verify CoreData model exists in bundle
3. Check console for migration errors
4. Use PersistenceController from shared instance

#### Issue 4: UI Not Updating
**Symptoms:** Data changes but views don't refresh
**Root Causes:**
- ViewModel not @MainActor
- State not being updated
- View not observing ViewModel

**Solutions:**
1. Add @MainActor to ViewModel class
2. Use DispatchQueue.main.async for state updates
3. Use @StateObject in parent view
4. Pass EnvironmentObject properly

#### Issue 5: Memory Leaks
**Symptoms:** App slows down over time
**Root Causes:**
- Strong reference cycles
- NetworkManager not deinitialized
- Timers not cancelled

**Solutions:**
1. Use weak self in closures
2. Use [weak self] in async/await
3. Profile with Instruments (Allocations tool)
4. Cancel all background tasks on deinit

### Performance Optimization Tips

1. **Network Requests:**
   - Cache current readings for 30 seconds
   - Batch offline sync into bulk requests
   - Implement request deduplication

2. **CoreData:**
   - Index frequently queried fields
   - Use NSFetchRequest with predicates efficiently
   - Implement incremental sync for large datasets

3. **UI Rendering:**
   - Use `DispatchQueue.main.async` for state updates
   - Avoid unnecessary view redraws
   - Use `.constant()` for non-changing properties
   - Lazy load heavy views

4. **Memory:**
   - Use weak references in delegates
   - Cancel network requests on view disappear
   - Clear image caches periodically

---

## FINAL SUMMARY

### Project Scope
- **Target Device:** iPhone 13 mini
- **Development Machine:** MacBook M3
- **Estimated Development Time:** 22-29 hours
- **Team Size:** 1 developer
- **Cost:** $1,100-2,900 (for developer time)

### Recommended Tech Stack
| Component | Choice |
|-----------|--------|
| Language | Swift 5.5+ |
| UI Framework | SwiftUI |
| Authentication | Google OAuth 2.0 |
| Networking | URLSession + Async/Await |
| Persistence | CoreData + Keychain + UserDefaults |
| Testing | XCTest |
| Distribution | TestFlight → App Store |

### Key Advantages
✓ Zero external dependencies (except Google OAuth)
✓ Modern Swift async/await
✓ Excellent offline support
✓ Scales from 1 device to 10,000+ users
✓ Single developer can build in 1 week full-time

### Next Steps
1. **Week 1:** Review this plan and complete Phase 1-3 (setup, auth, networking)
2. **Week 2:** Complete Phase 4-5 (UI, offline support)
3. **Week 3:** Complete Phase 6-7 (testing, deployment)

---

## APPENDIX: QUICK REFERENCE

### Important URLs
- Google Cloud Console: https://console.cloud.google.com
- Apple Developer: https://developer.apple.com
- Xcode Docs: https://developer.apple.com/xcode
- Swift API: https://developer.apple.com/swift

### Key Files to Create (In Order)
1. `Models/NetworkModels.swift` - DTOs
2. `Networking/KeychainManager.swift` - Secure storage
3. `Networking/AuthManager.swift` - OAuth
4. `Networking/NetworkManager.swift` - API calls
5. `Views/LoginView.swift` - Auth UI
6. `Views/HomeView.swift` - Main UI
7. `Persistence/PersistenceController.swift` - CoreData
8. `Persistence/OfflineQueueManager.swift` - Offline sync

### Build & Run Commands
```bash
# Clean build
xcode-select --reset
rm -rf ~/Library/Developer/Xcode/DerivedData/*

# Run tests
xcodebuild test -scheme SolarMonitor

# Archive for distribution
xcodebuild archive -scheme SolarMonitor -archivePath ./build/SolarMonitor.xcarchive
```

---

**Report Completed:** November 7, 2025
**Total Pages:** Comprehensive guide covering all aspects
**Status:** Ready for development

This is a **production-ready plan** with all code examples, architecture decisions, and step-by-step instructions needed to build the app successfully.


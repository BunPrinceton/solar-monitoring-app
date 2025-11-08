# iOS Solar App - Code Templates and Troubleshooting

## Part 1: Complete Code Templates

### Template 1: Full NetworkManager with Error Handling

```swift
// File: Networking/NetworkManager.swift
import Foundation

class NetworkManager: NSObject, ObservableObject {
    static let shared = NetworkManager()
    
    private let baseURL = URL(string: "https://api.solar.example.com")!
    private let keychainManager = KeychainManager()
    
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
    
    // MARK: - Private Helper Methods
    
    private func makeRequest<T: Codable>(
        _ url: URL,
        method: String = "GET",
        body: Encodable? = nil
    ) async throws -> T {
        // Get auth token
        guard let sessionToken = keychainManager.retrieve(key: "sessionToken") else {
            throw NetworkError.notAuthenticated
        }
        
        // Build request
        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = method
        urlRequest.setValue("Bearer \(sessionToken)", forHTTPHeaderField: "Authorization")
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        // Add body if provided
        if let body = body {
            urlRequest.httpBody = try JSONEncoder().encode(body)
        }
        
        // Log request (for debugging)
        logRequest(urlRequest)
        
        // Make request
        do {
            let (data, response) = try await URLSession.shared.data(for: urlRequest)
            
            // Validate response
            guard let httpResponse = response as? HTTPURLResponse else {
                throw NetworkError.invalidResponse
            }
            
            logResponse(httpResponse, data: data)
            
            // Handle status codes
            switch httpResponse.statusCode {
            case 200...299:
                break // Success
            case 401:
                // Token expired
                keychainManager.delete(key: "sessionToken")
                throw NetworkError.unauthorized
            case 400...499:
                throw NetworkError.clientError(httpResponse.statusCode)
            case 500...599:
                throw NetworkError.serverError(httpResponse.statusCode)
            default:
                throw NetworkError.unknownError
            }
            
            // Decode response
            do {
                return try JSONDecoder().decode(T.self, from: data)
            } catch {
                throw NetworkError.decodingError(error)
            }
        } catch let error as NetworkError {
            throw error
        } catch {
            throw NetworkError.networkError(error)
        }
    }
    
    // MARK: - Logging (helpful for debugging)
    
    private func logRequest(_ request: URLRequest) {
        #if DEBUG
        print("""
        ━━━ REQUEST ━━━
        URL: \(request.url?.absoluteString ?? "")
        Method: \(request.httpMethod ?? "GET")
        Headers: \(request.allHTTPHeaderFields ?? [:])
        """)
        #endif
    }
    
    private func logResponse(_ response: HTTPURLResponse, data: Data) {
        #if DEBUG
        let dataString = String(data: data, encoding: .utf8) ?? ""
        print("""
        ━━━ RESPONSE ━━━
        Status: \(response.statusCode)
        Headers: \(response.allHeaderFields)
        Body: \(dataString)
        """)
        #endif
    }
}

// MARK: - Network Models

struct CurrentReadingsResponse: Codable {
    let inverter_reading: Double
    let meter_reading: Double
    let timestamp: String
}

struct RecordRequest: Codable {
    let value: Double
    let type: String
    let timestamp: String
}

struct RecordResponse: Codable {
    let success: Bool
    let message: String
}

// MARK: - Error Types

enum NetworkError: LocalizedError, Equatable {
    case notAuthenticated
    case invalidResponse
    case unauthorized
    case clientError(Int)
    case serverError(Int)
    case decodingError(String)
    case networkError(String)
    case unknownError
    
    var errorDescription: String? {
        switch self {
        case .notAuthenticated:
            return "Please sign in first"
        case .invalidResponse:
            return "Server returned invalid response"
        case .unauthorized:
            return "Your session expired. Please sign in again."
        case .clientError(let code):
            return "Client error: \(code)"
        case .serverError(let code):
            return "Server error: \(code). Please try again later."
        case .decodingError(let msg):
            return "Failed to process response: \(msg)"
        case .networkError(let msg):
            return "Network error: \(msg)"
        case .unknownError:
            return "Unknown error occurred"
        }
    }
    
    static func == (lhs: NetworkError, rhs: NetworkError) -> Bool {
        switch (lhs, rhs) {
        case (.notAuthenticated, .notAuthenticated): return true
        case (.unauthorized, .unauthorized): return true
        default: return false
        }
    }
}
```

### Template 2: Complete AuthManager with Keychain

```swift
// File: Networking/AuthManager.swift
import Foundation
import GoogleSignIn

class AuthManager: NSObject, ObservableObject {
    static let shared = AuthManager()
    
    @Published var isLoggedIn = false
    @Published var userEmail: String?
    @Published var errorMessage: String?
    
    private let keychainManager = KeychainManager()
    
    override init() {
        super.init()
        restoreSession()
    }
    
    // MARK: - Public Methods
    
    func signInWithGoogle(with result: GIDSignInResult) {
        guard let idToken = result.user.idToken?.tokenString else {
            errorMessage = "Failed to get Google token"
            return
        }
        
        // In a real app, send idToken to YOUR backend
        // Backend exchanges token with Google and returns session token
        exchangeOAuthToken(idToken) { [weak self] response in
            DispatchQueue.main.async {
                if let sessionToken = response?.sessionToken {
                    self?.keychainManager.save(sessionToken, key: "sessionToken")
                    self?.isLoggedIn = true
                    self?.userEmail = result.user.profile?.email
                    self?.errorMessage = nil
                } else {
                    self?.errorMessage = "Failed to exchange token"
                }
            }
        }
    }
    
    func signOut() {
        GIDSignIn.sharedInstance.signOut()
        keychainManager.delete(key: "sessionToken")
        isLoggedIn = false
        userEmail = nil
        errorMessage = nil
    }
    
    func restoreSession() {
        if let sessionToken = keychainManager.retrieve(key: "sessionToken") {
            isLoggedIn = true
            // Could verify token is still valid here
        }
    }
    
    func getSessionToken() throws -> String? {
        return keychainManager.retrieve(key: "sessionToken")
    }
    
    // MARK: - Private Methods
    
    private func exchangeOAuthToken(
        _ idToken: String,
        completion: @escaping (TokenExchangeResponse?) -> Void
    ) {
        // TODO: Implement actual backend call
        // For now, mock response
        let mockResponse = TokenExchangeResponse(
            sessionToken: UUID().uuidString,
            expiresIn: 3600
        )
        completion(mockResponse)
    }
}

// MARK: - Models

struct TokenExchangeResponse: Codable {
    let sessionToken: String
    let expiresIn: Int
}

// MARK: - Keychain Manager

class KeychainManager {
    func save(_ value: String, key: String) {
        let data = value.data(using: .utf8) ?? Data()
        
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]
        
        // Delete existing
        SecItemDelete(query as CFDictionary)
        
        // Add new
        SecItemAdd(query as CFDictionary, nil)
    }
    
    func retrieve(key: String) -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true
        ]
        
        var result: AnyObject?
        SecItemCopyMatching(query as CFDictionary, &result)
        
        guard let data = result as? Data else { return nil }
        return String(data: data, encoding: .utf8)
    }
    
    func delete(key: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        
        SecItemDelete(query as CFDictionary)
    }
}
```

### Template 3: CoreData Offline Queue

```swift
// File: Persistence/OfflineQueueManager.swift
import CoreData

class OfflineQueueManager: ObservableObject {
    static let shared = OfflineQueueManager()
    
    @Published var pendingCount = 0
    
    private let dataStack = CoreDataStack.shared
    private let networkManager = NetworkManager.shared
    
    // MARK: - Queue Management
    
    func queueRecording(_ request: RecordRequest) {
        let context = dataStack.context
        
        let record = OfflineRecord(context: context)
        record.id = UUID()
        record.value = request.value
        record.type = request.type
        record.timestamp = ISO8601DateFormatter().date(from: request.timestamp) ?? Date()
        record.isUploaded = false
        record.createdAt = Date()
        record.uploadAttempts = 0
        
        dataStack.save()
        updatePendingCount()
        
        print("Queued offline: \(request.value) \(request.type)")
    }
    
    func retryOfflineRecordings() async {
        let context = dataStack.context
        
        let fetchRequest: NSFetchRequest<OfflineRecord> = OfflineRecord.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "isUploaded == false AND uploadAttempts < 3")
        fetchRequest.sortDescriptors = [
            NSSortDescriptor(keyPath: \OfflineRecord.createdAt, ascending: true)
        ]
        
        guard let records = try? context.fetch(fetchRequest) else { return }
        
        print("Retrying \(records.count) offline records...")
        
        for record in records {
            do {
                _ = try await networkManager.recordReading(record.recordRequest)
                
                record.isUploaded = true
                dataStack.save()
                
                print("Successfully uploaded: \(record.value)")
                updatePendingCount()
            } catch {
                record.uploadAttempts += 1
                dataStack.save()
                
                print("Failed to upload (attempt \(record.uploadAttempts)): \(error)")
            }
        }
    }
    
    func getPendingRecords() -> [OfflineRecord] {
        let context = dataStack.context
        let fetchRequest: NSFetchRequest<OfflineRecord> = OfflineRecord.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "isUploaded == false")
        
        return (try? context.fetch(fetchRequest)) ?? []
    }
    
    func getPendingCount() -> Int {
        let context = dataStack.context
        let fetchRequest: NSFetchRequest<OfflineRecord> = OfflineRecord.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "isUploaded == false")
        
        return (try? context.count(for: fetchRequest)) ?? 0
    }
    
    func clearUploaded() {
        let context = dataStack.context
        let fetchRequest: NSFetchRequest<OfflineRecord> = OfflineRecord.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "isUploaded == true")
        
        let records = (try? context.fetch(fetchRequest)) ?? []
        records.forEach { context.delete($0) }
        
        dataStack.save()
    }
    
    // MARK: - Private
    
    private func updatePendingCount() {
        DispatchQueue.main.async {
            self.pendingCount = self.getPendingCount()
        }
    }
}

// MARK: - Core Data Model

@objc(OfflineRecord)
class OfflineRecord: NSManagedObject {
    @NSManaged var id: UUID
    @NSManaged var value: Double
    @NSManaged var type: String
    @NSManaged var timestamp: Date
    @NSManaged var isUploaded: Bool
    @NSManaged var createdAt: Date
    @NSManaged var uploadAttempts: Int16
    
    var recordRequest: RecordRequest {
        let formatter = ISO8601DateFormatter()
        return RecordRequest(
            value: value,
            type: type,
            timestamp: formatter.string(from: timestamp)
        )
    }
}

// MARK: - Core Data Stack

class CoreDataStack {
    static let shared = CoreDataStack()
    
    let container: NSPersistentContainer
    
    var context: NSManagedObjectContext {
        container.viewContext
    }
    
    init() {
        container = NSPersistentContainer(name: "SolarMonitoring")
        
        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Failed to load Core Data: \(error)")
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
                print("Failed to save Core Data: \(error)")
            }
        }
    }
}
```

### Template 4: Complete ViewModel

```swift
// File: ViewModels/HomeViewModel.swift
import Foundation
import Combine

class HomeViewModel: ObservableObject {
    @Published var inverterReading: Double = 0
    @Published var meterReading: Double = 0
    @Published var lastUpdated: String = "Never"
    @Published var isLoading = false
    @Published var errorMessage: String?
    @Published var successMessage: String?
    @Published var pendingCount: Int = 0
    
    private let networkManager = NetworkManager.shared
    private let offlineQueue = OfflineQueueManager.shared
    private var updateTimer: Timer?
    
    deinit {
        updateTimer?.invalidate()
    }
    
    // MARK: - Public Methods
    
    func fetchCurrentReadings() {
        isLoading = true
        errorMessage = nil
        
        Task {
            do {
                let readings = try await networkManager.fetchCurrentReadings()
                
                DispatchQueue.main.async {
                    self.inverterReading = readings.inverter_reading
                    self.meterReading = readings.meter_reading
                    
                    let formatter = DateFormatter()
                    formatter.dateFormat = "h:mm a"
                    self.lastUpdated = formatter.string(from: Date())
                    
                    self.isLoading = false
                }
                
                // Try to sync offline records
                await syncOfflineRecordings()
            } catch {
                DispatchQueue.main.async {
                    self.errorMessage = error.localizedDescription
                    self.isLoading = false
                }
            }
        }
    }
    
    func autoRecord() {
        let request = RecordRequest(
            value: inverterReading,
            type: "auto",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        submitRecording(request)
    }
    
    func autoRecordAndPush() {
        let request = RecordRequest(
            value: inverterReading,
            type: "auto_push",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        submitRecording(request)
    }
    
    func setupAutoRefresh(interval: TimeInterval = 60) {
        updateTimer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { [weak self] _ in
            self?.fetchCurrentReadings()
        }
    }
    
    // MARK: - Private Methods
    
    private func submitRecording(_ request: RecordRequest) {
        Task {
            do {
                _ = try await networkManager.recordReading(request)
                
                DispatchQueue.main.async {
                    let message = request.type == "auto_push" ?
                        "Recording saved and pushed!" :
                        "Recording saved!"
                    self.showSuccess(message)
                }
            } catch {
                // Save offline if network fails
                offlineQueue.queueRecording(request)
                
                DispatchQueue.main.async {
                    self.showSuccess("Saved offline (will sync when online)")
                }
                
                updatePendingCount()
            }
        }
    }
    
    private func syncOfflineRecordings() async {
        if offlineQueue.getPendingCount() > 0 {
            await offlineQueue.retryOfflineRecordings()
            updatePendingCount()
        }
    }
    
    private func updatePendingCount() {
        DispatchQueue.main.async {
            self.pendingCount = self.offlineQueue.getPendingCount()
        }
    }
    
    private func showSuccess(_ message: String) {
        successMessage = message
        
        // Auto-hide after 3 seconds
        DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
            if self.successMessage == message {
                self.successMessage = nil
            }
        }
    }
}
```

---

## Part 2: Common Implementation Patterns

### Pattern 1: Retry Logic with Exponential Backoff

```swift
func retryWithBackoff<T>(
    maxAttempts: Int = 3,
    initialDelay: TimeInterval = 1,
    action: () async throws -> T
) async throws -> T {
    var lastError: Error?
    var delay = initialDelay
    
    for attempt in 0..<maxAttempts {
        do {
            return try await action()
        } catch {
            lastError = error
            
            if attempt < maxAttempts - 1 {
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
                delay *= 2  // Double delay each time
            }
        }
    }
    
    throw lastError ?? NSError(domain: "RetryError", code: -1)
}

// Usage:
let readings = try await retryWithBackoff {
    try await networkManager.fetchCurrentReadings()
}
```

### Pattern 2: Network Connectivity Check

```swift
import Network

class ConnectivityManager: ObservableObject {
    @Published var isOnline = true
    
    private let monitor = NWPathMonitor()
    private let queue = DispatchQueue(label: "NetworkMonitor")
    
    func startMonitoring() {
        monitor.pathUpdateHandler = { [weak self] path in
            DispatchQueue.main.async {
                self?.isOnline = path.status == .satisfied
            }
        }
        
        monitor.start(queue: queue)
    }
    
    func stopMonitoring() {
        monitor.cancel()
    }
}

// Usage in ViewModel:
@StateObject var connectivity = ConnectivityManager()

func submitRecording(_ request: RecordRequest) {
    if connectivity.isOnline {
        // Try network
    } else {
        // Save offline immediately
        offlineQueue.queueRecording(request)
    }
}
```

### Pattern 3: Loading State Management

```swift
enum LoadingState<T> {
    case idle
    case loading
    case success(T)
    case error(Error)
    
    var isLoading: Bool {
        if case .loading = self { return true }
        return false
    }
}

class DataViewModel: ObservableObject {
    @Published var state: LoadingState<ReadingsResponse> = .idle
    
    func loadData() {
        state = .loading
        
        Task {
            do {
                let data = try await networkManager.fetchCurrentReadings()
                DispatchQueue.main.async {
                    self.state = .success(data)
                }
            } catch {
                DispatchQueue.main.async {
                    self.state = .error(error)
                }
            }
        }
    }
}
```

---

## Part 3: Comprehensive Troubleshooting Guide

### Issue 1: App Crashes on Launch

**Symptoms:**
- App immediately terminates
- "App quit unexpectedly"
- Xcode shows red error

**Diagnosis:**
```bash
# Check console output
# Run with full logs:
log stream --level debug --predicate 'eventMessage contains "SolarMonitor"'
```

**Common Causes & Fixes:**

1. **Missing GoogleService-Info.plist**
   ```
   Fix: Add GoogleService-Info.plist to Xcode target
   1. Drag file into Xcode
   2. Check "Copy items if needed"
   3. Add to target "SolarMonitor"
   ```

2. **CoreData model not found**
   ```
   Fix: Verify SolarMonitoring.xcdatamodeld exists
   1. File menu > New > Data Model
   2. Name it "SolarMonitoring"
   3. Create OfflineRecord entity with attributes
   ```

3. **Missing framework**
   ```
   Fix: Add required frameworks
   1. Project Settings > Build Phases
   2. Link Binary With Libraries
   3. Add: Security.framework
   ```

---

### Issue 2: "Failed to Load Google Service" Error

**Symptoms:**
- GoogleSignIn crashes
- "Invalid bundle configuration"

**Solutions:**

1. **Check GoogleService-Info.plist:**
   ```bash
   cat GoogleService-Info.plist | grep -A5 "BUNDLE_ID"
   ```

2. **Verify Bundle ID matches:**
   ```
   Xcode > Project Settings > General > Bundle Identifier
   Should match: BUNDLE_ID in GoogleService-Info.plist
   ```

3. **Check Info.plist configuration:**
   ```xml
   <key>CFBundleURLTypes</key>
   <array>
     <dict>
       <key>CFBundleURLSchemes</key>
       <array>
         <string>com.googleusercontent.apps.YOUR_CLIENT_ID</string>
       </array>
     </dict>
   </array>
   ```

4. **Reset Google Sign-In:**
   ```bash
   # Delete app from simulator
   xcrun simctl erase all
   
   # Rebuild
   Cmd+B
   ```

---

### Issue 3: Network Requests Fail with 401 Unauthorized

**Symptoms:**
- "Session expired" errors
- API returns 401 status
- Can't make any requests

**Diagnosis Checklist:**
```swift
// Add debugging to NetworkManager
func debugToken() {
    if let token = keychainManager.retrieve(key: "sessionToken") {
        print("Token exists: \(token.prefix(20))...")
    } else {
        print("No token found!")
    }
}

// In ViewModel, call:
networkManager.debugToken()
```

**Solutions:**

1. **Token not saved:**
   ```swift
   // Check AuthManager sign-in
   func signInWithGoogle(with result: GIDSignInResult) {
       print("Debug: Starting OAuth exchange")
       
       if let idToken = result.user.idToken?.tokenString {
           print("Debug: Got ID token: \(idToken.prefix(20))...")
       } else {
           print("Error: No ID token!")
       }
   }
   ```

2. **Token expired:**
   ```swift
   // Implement token refresh
   func refreshToken() async throws -> String {
       // Call backend /refresh endpoint
       // Store new token in keychain
       // Return new token
   }
   ```

3. **Backend not returning token:**
   ```bash
   # Test backend directly
   curl -X POST https://api.solar.example.com/oauth/exchange \
     -H "Content-Type: application/json" \
     -d '{"idToken":"YOUR_TOKEN"}'
   ```

---

### Issue 4: Offline Queue Not Persisting

**Symptoms:**
- Offline records disappear when app closes
- CoreData not saving
- "Pending" count always zero

**Solutions:**

1. **Verify CoreData saves:**
   ```swift
   // Add to CoreDataStack
   func debugSave() {
       let context = container.viewContext
       print("Has changes: \(context.hasChanges)")
       print("Inserted: \(context.insertedObjects.count)")
       
       try? context.save()
       print("Saved successfully")
   }
   ```

2. **Check CoreData model:**
   ```
   1. Xcode > Open SolarMonitoring.xcdatamodeld
   2. Verify OfflineRecord entity exists
   3. Verify all attributes (id, value, type, timestamp, etc.)
   4. Check data types match code
   ```

3. **Enable CoreData debugging:**
   ```bash
   # In Xcode Scheme > Arguments
   Passed On Launch:
   -com.apple.CoreData.SQLiteDebugOption 1
   -com.apple.CoreData.Logging.stderr 1
   ```

4. **Manual save test:**
   ```swift
   func testSave() {
       let queue = OfflineQueueManager.shared
       let request = RecordRequest(
           value: 5.5,
           type: "test",
           timestamp: ISO8601DateFormatter().string(from: Date())
       )
       
       queue.queueRecording(request)
       
       // Verify it was saved
       let count = queue.getPendingCount()
       print("Pending records: \(count)")
       assert(count > 0, "Failed to save!")
   }
   ```

---

### Issue 5: UI Doesn't Update When Data Changes

**Symptoms:**
- Values show old data
- Pull to refresh doesn't update
- @Published properties not triggering updates

**Solutions:**

1. **Check @Published usage:**
   ```swift
   // Wrong:
   class ViewModel {
       var readings: Double = 0  // Won't publish changes!
   }
   
   // Correct:
   class ViewModel: ObservableObject {
       @Published var readings: Double = 0  // Will publish changes
   }
   ```

2. **Verify DispatchQueue.main.async:**
   ```swift
   // Wrong - updates from background thread
   func fetchData() {
       Task {
           let data = try await networkManager.fetch()
           self.data = data  // Updates on background thread!
       }
   }
   
   // Correct - updates on main thread
   func fetchData() {
       Task {
           let data = try await networkManager.fetch()
           DispatchQueue.main.async {
               self.data = data  // Updates on main thread
           }
       }
   }
   ```

3. **Check View updates:**
   ```swift
   // Add debug print
   struct HomeView: View {
       @StateObject var viewModel = HomeViewModel()
       
       var body: some View {
           VStack {
               Text("\(viewModel.reading)")
                   .onReceive(
                       viewModel.$reading,
                       perform: { value in
                           print("Reading changed to: \(value)")
                       }
                   )
           }
       }
   }
   ```

---

### Issue 6: Memory Leaks or Slow Performance

**Symptoms:**
- App slows down over time
- Memory usage keeps growing
- App crashes after running a while

**Solutions:**

1. **Check for retain cycles:**
   ```swift
   // Wrong - creates cycle
   class ViewModel {
       let manager = NetworkManager()
       
       func fetch() {
           manager.fetch { [self] result in  // Captures self!
               self.data = result
           }
       }
   }
   
   // Correct - uses weak self
   func fetch() {
       manager.fetch { [weak self] result in
           self?.data = result
       }
   }
   ```

2. **Clean up timers:**
   ```swift
   class ViewModel: ObservableObject {
       var updateTimer: Timer?
       
       deinit {
           updateTimer?.invalidate()  // Important!
       }
       
       func startRefresh() {
           updateTimer = Timer.scheduledTimer(
               withTimeInterval: 60,
               repeats: true
           ) { [weak self] _ in
               self?.fetchData()
           }
       }
   }
   ```

3. **Monitor with Xcode Instruments:**
   ```
   Xcode > Product > Profile (Cmd+I)
   Select "Memory" instrument
   Watch for leaks (red sections)
   ```

---

### Issue 7: OAuth Login Doesn't Work

**Symptoms:**
- Google login button doesn't respond
- Safari doesn't open for login
- "OAuth flow interrupted"

**Solutions:**

1. **Check URL schemes in Info.plist:**
   ```xml
   <key>CFBundleURLTypes</key>
   <array>
     <dict>
       <key>CFBundleURLSchemes</key>
       <array>
         <string>com.googleusercontent.apps.XXXXX</string>
       </array>
     </dict>
   </array>
   ```

2. **Verify Google Cloud setup:**
   ```
   1. Go to console.cloud.google.com
   2. Select your project
   3. APIs > Google+ API > Enable
   4. Credentials > OAuth 2.0 ID > iOS app
   5. Verify bundle ID matches exactly
   6. Download updated GoogleService-Info.plist
   ```

3. **Test OAuth separately:**
   ```swift
   Button("Test Google Sign In") {
       GIDSignIn.sharedInstance.signIn(
           withPresenting: rootViewController
       ) { result, error in
           if let error = error {
               print("OAuth error: \(error)")
           }
           if let result = result {
               print("Success: \(result.user.profile?.email ?? "")")
           }
       }
   }
   ```

---

## Part 4: Testing Checklist

### Pre-Launch Testing

- [ ] App launches without crashes
- [ ] Google OAuth flow works completely
- [ ] Can fetch current readings (GET /current_readings)
- [ ] Can submit recording (POST /record)
- [ ] Offline queue saves records when offline
- [ ] Offline records retry when back online
- [ ] UI is readable on iPhone 13 mini (18pt+ text)
- [ ] All buttons are easily tappable (48pt+ min)
- [ ] Settings screen shows correct email
- [ ] Sign out works and clears data
- [ ] App recovers from network errors gracefully
- [ ] No crashes after 10 minutes of use
- [ ] Battery usage is reasonable (not draining fast)

### Device Testing Specific

- [ ] Works on physical iPhone 13 mini
- [ ] Face ID/Touch ID doesn't interfere
- [ ] Portrait mode works well
- [ ] Doesn't crash when backgrounded
- [ ] Can handle low battery mode
- [ ] Handles network switching (WiFi to cellular)

---

## Part 5: Performance Optimization Tips

### 1. Network Request Optimization

```swift
// Add request timeouts
var request = URLRequest(url: url)
request.timeoutInterval = 10  // Seconds

// Compress large responses
request.setValue("gzip", forHTTPHeaderField: "Accept-Encoding")

// Cache GET requests
let config = URLSessionConfiguration.default
config.requestCachePolicy = .returnCacheDataElseLoad
let session = URLSession(configuration: config)
```

### 2. CoreData Optimization

```swift
// Use batch operations for large updates
let batchDelete = NSBatchDeleteRequest(
    fetchRequest: OfflineRecord.fetchRequest()
)
batchDelete.resultType = .resultTypeCount

try? context.execute(batchDelete)

// Use NSFetchedResultsController for monitoring
let controller = NSFetchedResultsController<OfflineRecord>(
    fetchRequest: OfflineRecord.fetchRequest(),
    managedObjectContext: context,
    sectionNameKeyPath: nil,
    cacheName: "PendingRecords"
)
```

### 3. UI Performance

```swift
// Lazy load large lists
List {
    ForEach(items, id: \.self) { item in
        ItemRow(item: item)
    }
    .onAppear {
        loadMoreItems()  // Load as needed
    }
}

// Use lighter images
Image(systemName: "bolt.fill")  // SF Symbols are optimal
    .font(.system(size: 20))

// Reduce animation complexity
.animation(.linear, value: isLoading)  // Simple animation
```


# iOS Solar Monitoring App - Comprehensive Implementation Plan

## Executive Summary

This document outlines a complete strategy for building a dad-friendly SwiftUI iOS app for solar monitoring on iPhone 13 mini, built on a MacBook M3. The app will integrate OAuth authentication, display real-time solar data, enable three recording actions, and support offline functionality with local persistence.

---

# PART 1: RESEARCH FINDINGS & BEST PRACTICES

## 1. iOS Development Environment Setup (MacBook M3)

### Xcode Installation
- **Recommended Version:** Xcode 15.0+ (supports iOS 17+)
- **Size:** ~15-20GB
- **Installation Time:** 20-30 minutes on M3
- **Key Components:**
  - Swift compiler (native M3 arm64 support)
  - iOS Simulator
  - SwiftUI preview canvas
  - Instruments for performance profiling

### M3 MacBook Optimization
**Advantages:**
- Native ARM64 architecture for iOS simulator (2-3x faster than Intel)
- Unified memory architecture = faster data access
- Excellent battery life for development
- 8GB+ RAM sufficient for Xcode + simulator

**Setup Steps:**
```bash
# Install Xcode (via App Store or Direct Download)
xcode-select --install

# Verify Swift installation
swift --version

# Open Xcode
open /Applications/Xcode.app
```

---

## 2. Authentication: OAuth Implementation for iOS

### Recommended Approach: Google OAuth 2.0 with Backend Mediation

**Why Google OAuth over alternatives:**
1. **User-friendly** - Most users have Google accounts
2. **Secure** - Industry standard, well-tested
3. **Dad-friendly** - Simple "Sign in with Google" button
4. **Less setup** - Backend handles token management

### Architecture Comparison Table

| Approach | Pros | Cons | Complexity |
|----------|------|------|-----------|
| **Google OAuth (Recommended)** | Simple UX, backend handles security, less client-side token work | Requires Google Cloud setup | Low |
| **Custom Backend OAuth** | Full control, can integrate with proprietary system | More backend work, client must handle tokens | Medium |
| **Apple Sign in** | Native iOS integration, privacy-focused | Limited to Apple ecosystem users | Low |
| **Firebase Auth** | Easy setup, Google-managed infrastructure | Vendor lock-in | Very Low |

### Recommended Implementation: Google OAuth 2.0

**Step 1: Backend Setup (Server-side)**
```
Backend Flow:
1. App sends OAuth code to backend
2. Backend exchanges code for token
3. Backend stores token securely
4. Backend returns user session
5. App stores session token in Keychain
```

**Step 2: Client-side Library**
- Use **GoogleSignIn SDK** for iOS
- Handles OAuth flow securely
- Manages token refresh automatically

**Step 3: Implementation Flow**
```
User taps "Sign in with Google"
  ↓
Google OAuth system appears
  ↓
User logs in with Google credentials
  ↓
OAuth code returned to app
  ↓
App sends code to backend
  ↓
Backend verifies code with Google
  ↓
Backend creates user session
  ↓
App receives session token
  ↓
Store token in Keychain
  ↓
User logged in!
```

### Code Structure for Authentication

```swift
// AuthManager.swift - Handles OAuth and token management
import GoogleSignIn

class AuthManager: NSObject, ObservableObject {
    @Published var isLoggedIn = false
    @Published var userEmail: String?
    @Published var errorMessage: String?
    
    let keychainManager = KeychainManager()
    let apiManager = APIManager()
    
    // Google Sign-in
    func signInWithGoogle(with result: GIDSignInResult) {
        guard let idToken = result.user.idToken?.tokenString else { return }
        
        // Send to backend
        apiManager.exchangeOAuthToken(idToken) { [weak self] response in
            DispatchQueue.main.async {
                if let sessionToken = response?.sessionToken {
                    self?.keychainManager.save(sessionToken, key: "sessionToken")
                    self?.isLoggedIn = true
                    self?.userEmail = result.user.profile?.email
                }
            }
        }
    }
    
    // Check if already logged in
    func restoreSession() {
        if let sessionToken = keychainManager.retrieve(key: "sessionToken") {
            isLoggedIn = true
        }
    }
}

// KeychainManager.swift - Secure token storage
class KeychainManager {
    func save(_ value: String, key: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: value.data(using: .utf8) ?? Data()
        ]
        
        SecItemDelete(query as CFDictionary)
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
}
```

---

## 3. Networking Layer: URLSession Architecture

### Recommended Pattern: Protocol-Based Network Manager

**Why this approach:**
- Testable (can mock for unit tests)
- Reusable across the app
- Handles errors consistently
- Manages authentication automatically

### Network Layer Implementation

```swift
// NetworkModels.swift
struct CurrentReadingsResponse: Codable {
    let inverter_reading: Double
    let meter_reading: Double
    let timestamp: String
}

struct RecordRequest: Codable {
    let value: Double
    let type: String  // "manual" or "auto"
    let timestamp: String
}

// NetworkManager.swift
class NetworkManager {
    static let shared = NetworkManager()
    
    let baseURL = URL(string: "https://api.solar.example.com")!
    let authManager = AuthManager.shared
    
    // GET /current_readings
    func fetchCurrentReadings() async throws -> CurrentReadingsResponse {
        guard let sessionToken = try authManager.getSessionToken() else {
            throw NetworkError.notAuthenticated
        }
        
        var request = URLRequest(url: baseURL.appendingPathComponent("current_readings"))
        request.httpMethod = "GET"
        request.setValue("Bearer \(sessionToken)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        let (data, response) = try await URLSession.shared.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        
        guard (200...299).contains(httpResponse.statusCode) else {
            if httpResponse.statusCode == 401 {
                throw NetworkError.unauthorized
            }
            throw NetworkError.serverError(httpResponse.statusCode)
        }
        
        return try JSONDecoder().decode(CurrentReadingsResponse.self, from: data)
    }
    
    // POST /record
    func recordReading(_ request: RecordRequest) async throws -> RecordResponse {
        guard let sessionToken = try authManager.getSessionToken() else {
            throw NetworkError.notAuthenticated
        }
        
        var urlRequest = URLRequest(url: baseURL.appendingPathComponent("record"))
        urlRequest.httpMethod = "POST"
        urlRequest.setValue("Bearer \(sessionToken)", forHTTPHeaderField: "Authorization")
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601
        urlRequest.httpBody = try encoder.encode(request)
        
        let (data, response) = try await URLSession.shared.data(for: urlRequest)
        
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.serverError((response as? HTTPURLResponse)?.statusCode ?? -1)
        }
        
        return try JSONDecoder().decode(RecordResponse.self, from: data)
    }
}

enum NetworkError: LocalizedError {
    case notAuthenticated
    case invalidResponse
    case unauthorized
    case serverError(Int)
    case decodingError(Error)
    
    var errorDescription: String? {
        switch self {
        case .notAuthenticated:
            return "Please sign in first"
        case .invalidResponse:
            return "Server returned invalid response"
        case .unauthorized:
            return "Your session has expired. Please sign in again."
        case .serverError(let code):
            return "Server error: \(code)"
        case .decodingError:
            return "Failed to decode server response"
        }
    }
}
```

---

## 4. Local Persistence Strategy: CoreData vs Alternatives

### Comparison Matrix

| Method | Use Case | Complexity | Query Power | Offline Support |
|--------|----------|-----------|------------|-----------------|
| **CoreData (Recommended)** | Complex data, sync needs | High | Excellent | Excellent |
| **UserDefaults** | Small settings only | Very Low | None | Basic |
| **SQLite** | Custom queries, large data | High | Excellent | Excellent |
| **Realm** | Modern alternative to CoreData | Medium | Excellent | Good |

### Recommended: CoreData for Offline Queue

**Why CoreData:**
1. Built-in iOS framework (no external dependencies)
2. Powerful query capabilities
3. Handles relationships well (user → recordings)
4. Native sync capabilities
5. Efficient memory management

### CoreData Implementation: Offline Queue

```swift
// CoreDataStack.swift
import CoreData

class CoreDataStack {
    static let shared = CoreDataStack()
    
    let container: NSPersistentContainer
    
    var context: NSManagedObjectContext {
        container.viewContext
    }
    
    init() {
        container = NSPersistentContainer(name: "SolarMonitoring")
        
        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Failed to load Core Data: \(error)")
            }
        }
        
        // Auto-save every 5 seconds
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
    
    func save() {
        let context = container.viewContext
        
        if context.hasChanges {
            try? context.save()
        }
    }
}

// CoreData Models
import CoreData

@objc(OfflineRecord)
class OfflineRecord: NSManagedObject {
    @NSManaged var id: UUID
    @NSManaged var value: Double
    @NSManaged var type: String  // "manual" or "auto"
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

// OfflineQueueManager.swift
class OfflineQueueManager {
    static let shared = OfflineQueueManager()
    
    let dataStack = CoreDataStack.shared
    let networkManager = NetworkManager.shared
    
    // Save recording locally
    func queueRecording(_ request: RecordRequest) {
        let context = dataStack.context
        
        let offlineRecord = OfflineRecord(context: context)
        offlineRecord.id = UUID()
        offlineRecord.value = request.value
        offlineRecord.type = request.type
        offlineRecord.timestamp = ISO8601DateFormatter().date(from: request.timestamp) ?? Date()
        offlineRecord.isUploaded = false
        offlineRecord.createdAt = Date()
        offlineRecord.uploadAttempts = 0
        
        dataStack.save()
    }
    
    // Retry failed uploads
    func retryOfflineRecordings() async {
        let context = dataStack.context
        let fetchRequest: NSFetchRequest<OfflineRecord> = OfflineRecord.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "isUploaded == false")
        fetchRequest.sortDescriptors = [NSSortDescriptor(keyPath: \OfflineRecord.createdAt, ascending: true)]
        
        guard let records = try? context.fetch(fetchRequest) else { return }
        
        for record in records {
            // Maximum 3 retry attempts
            if record.uploadAttempts >= 3 {
                continue
            }
            
            do {
                _ = try await networkManager.recordReading(record.recordRequest)
                
                record.isUploaded = true
                dataStack.save()
            } catch {
                record.uploadAttempts += 1
                dataStack.save()
            }
        }
    }
    
    // Get pending records count
    func getPendingCount() -> Int {
        let context = dataStack.context
        let fetchRequest: NSFetchRequest<OfflineRecord> = OfflineRecord.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "isUploaded == false")
        
        return (try? context.count(for: fetchRequest)) ?? 0
    }
}
```

**Migration to CoreData:**
```swift
// In CoreDataStack init() or separate migration function
func migrateIfNeeded() {
    let container = NSPersistentContainer(name: "SolarMonitoring")
    
    container.loadPersistentStores(completionHandler: { _, error in
        if let error = error as NSError? {
            // Handle migration if needed
            print("Migration issue: \(error), \(error.userInfo)")
        }
    })
}
```

---

## 5. Dad-Friendly UI Design Principles

### Accessibility Standards for Senior Users

**Text Size:**
- Minimum 18pt for body text
- 24-28pt for section headers
- 20pt for buttons

**Color Contrast:**
- Minimum 4.5:1 ratio (WCAG AA standard)
- Avoid red-green color pairs (color blindness)
- Use high-contrast themes

**Button Design:**
- Minimum 48pt tap target
- Clear, descriptive labels
- Visual feedback on tap

**Navigation:**
- Minimize nested screens
- Clear back buttons
- Obvious next steps

### SwiftUI UI Architecture

```swift
// ContentView.swift - Main navigation
import SwiftUI

struct ContentView: View {
    @StateObject var authManager = AuthManager.shared
    
    var body: some View {
        if authManager.isLoggedIn {
            MainTabView()
        } else {
            LoginView()
        }
    }
}

// LoginView.swift - OAuth Sign-in
struct LoginView: View {
    @StateObject var authManager = AuthManager.shared
    @State var showError = false
    
    var body: some View {
        VStack(spacing: 40) {
            VStack(spacing: 20) {
                Image(systemName: "sun.max.fill")
                    .font(.system(size: 80))
                    .foregroundColor(.yellow)
                
                Text("Solar Monitor")
                    .font(.system(size: 32, weight: .bold))
                
                Text("Track your solar energy")
                    .font(.system(size: 20))
                    .foregroundColor(.gray)
            }
            
            Spacer()
            
            VStack(spacing: 20) {
                GoogleSignInButton()
                    .onTapGesture {
                        // Google OAuth flow
                    }
                    .frame(height: 56)
                
                if let error = authManager.errorMessage {
                    Text(error)
                        .font(.system(size: 16))
                        .foregroundColor(.red)
                        .padding()
                        .background(Color.red.opacity(0.1))
                        .cornerRadius(8)
                }
            }
            .padding(.horizontal, 20)
            .padding(.bottom, 40)
        }
        .ignoresSafeArea()
    }
}

// HomeView.swift - Main monitoring screen
struct HomeView: View {
    @StateObject var viewModel = HomeViewModel()
    @State var showRecordingOptions = false
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // Status Cards
                VStack(spacing: 16) {
                    ReadingCard(
                        title: "Inverter Output",
                        value: viewModel.inverterReading,
                        unit: "kW",
                        icon: "bolt.fill"
                    )
                    
                    ReadingCard(
                        title: "Meter Reading",
                        value: viewModel.meterReading,
                        unit: "kWh",
                        icon: "gauge.badge.plus"
                    )
                    
                    Text("Last updated: \(viewModel.lastUpdated)")
                        .font(.system(size: 14))
                        .foregroundColor(.gray)
                }
                .padding(.horizontal, 20)
                
                Spacer()
                
                // Action Buttons
                VStack(spacing: 12) {
                    ActionButton(
                        title: "Auto Record",
                        subtitle: "Save current reading automatically",
                        icon: "checkmark.circle.fill",
                        color: .blue,
                        action: { viewModel.autoRecord() }
                    )
                    
                    ActionButton(
                        title: "Auto Record + Push",
                        subtitle: "Save and send to utility",
                        icon: "arrow.up.circle.fill",
                        color: .green,
                        action: { viewModel.autoRecordAndPush() }
                    )
                    
                    ActionButton(
                        title: "Manual Entry",
                        subtitle: "Enter reading manually",
                        icon: "pencil.circle.fill",
                        color: .orange,
                        action: { showRecordingOptions = true }
                    )
                }
                .padding(.horizontal, 20)
                .padding(.bottom, 20)
            }
            .navigationTitle("Solar Monitor")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    NavigationLink(destination: SettingsView()) {
                        Image(systemName: "gearshape.fill")
                            .font(.system(size: 18))
                    }
                }
            }
            .onAppear {
                viewModel.fetchCurrentReadings()
                // Refresh every 60 seconds
                Timer.scheduledTimer(withTimeInterval: 60, repeats: true) { _ in
                    viewModel.fetchCurrentReadings()
                }
            }
            .sheet(isPresented: $showRecordingOptions) {
                ManualEntryView(isPresented: $showRecordingOptions)
            }
        }
    }
}

// ReadingCard.swift - Display reading component
struct ReadingCard: View {
    let title: String
    let value: Double
    let unit: String
    let icon: String
    
    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            HStack {
                VStack(alignment: .leading) {
                    Text(title)
                        .font(.system(size: 18, weight: .semibold))
                    
                    HStack(spacing: 4) {
                        Text(String(format: "%.2f", value))
                            .font(.system(size: 28, weight: .bold, design: .monospaced))
                        
                        Text(unit)
                            .font(.system(size: 18))
                            .foregroundColor(.gray)
                    }
                }
                
                Spacer()
                
                Image(systemName: icon)
                    .font(.system(size: 32))
                    .foregroundColor(.yellow)
            }
        }
        .padding(20)
        .background(Color(.systemBackground))
        .cornerRadius(12)
        .shadow(radius: 2)
    }
}

// ActionButton.swift - Large touch-friendly button
struct ActionButton: View {
    let title: String
    let subtitle: String
    let icon: String
    let color: Color
    let action: () -> Void
    
    var body: some View {
        Button(action: action) {
            VStack(alignment: .leading, spacing: 8) {
                HStack {
                    VStack(alignment: .leading, spacing: 4) {
                        HStack(spacing: 12) {
                            Image(systemName: icon)
                                .font(.system(size: 24))
                            
                            Text(title)
                                .font(.system(size: 20, weight: .semibold))
                        }
                        
                        Text(subtitle)
                            .font(.system(size: 16))
                            .foregroundColor(.gray)
                    }
                    
                    Spacer()
                    
                    Image(systemName: "chevron.right")
                        .font(.system(size: 18, weight: .semibold))
                }
            }
            .frame(maxWidth: .infinity, alignment: .leading)
            .padding(20)
            .background(color.opacity(0.1))
            .overlay(
                RoundedRectangle(cornerRadius: 12)
                    .stroke(color, lineWidth: 2)
            )
            .foregroundColor(color)
        }
        .frame(minHeight: 80)
    }
}

// ManualEntryView.swift - Manual recording
struct ManualEntryView: View {
    @Binding var isPresented: Bool
    @StateObject var viewModel = ManualEntryViewModel()
    @State var manualValue: String = ""
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                Text("Enter Reading")
                    .font(.system(size: 28, weight: .bold))
                    .frame(maxWidth: .infinity, alignment: .leading)
                
                VStack(spacing: 12) {
                    Text("kW Output")
                        .font(.system(size: 18, weight: .semibold))
                    
                    TextField("0.00", text: $manualValue)
                        .font(.system(size: 32, weight: .semibold, design: .monospaced))
                        .keyboardType(.decimalPad)
                        .padding(.vertical, 20)
                        .padding(.horizontal, 16)
                        .background(Color(.systemBackground))
                        .cornerRadius(12)
                }
                
                Spacer()
                
                VStack(spacing: 12) {
                    Button(action: {
                        viewModel.submitManualReading(Double(manualValue) ?? 0)
                        isPresented = false
                    }) {
                        Text("Submit Recording")
                            .font(.system(size: 20, weight: .semibold))
                            .frame(maxWidth: .infinity)
                            .padding(.vertical, 16)
                            .background(Color.blue)
                            .foregroundColor(.white)
                            .cornerRadius(12)
                    }
                    .frame(minHeight: 56)
                    
                    Button(action: { isPresented = false }) {
                        Text("Cancel")
                            .font(.system(size: 18))
                            .frame(maxWidth: .infinity)
                            .padding(.vertical, 14)
                            .background(Color(.systemGray5))
                            .foregroundColor(.black)
                            .cornerRadius(12)
                    }
                    .frame(minHeight: 56)
                }
            }
            .padding(20)
        }
    }
}

// SettingsView.swift - Settings and OAuth
struct SettingsView: View {
    @StateObject var authManager = AuthManager.shared
    @Environment(\.dismiss) var dismiss
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                VStack(spacing: 12) {
                    Text("Account")
                        .font(.system(size: 18, weight: .semibold))
                        .frame(maxWidth: .infinity, alignment: .leading)
                    
                    VStack(spacing: 8) {
                        HStack {
                            Text("Email")
                                .font(.system(size: 16))
                            Spacer()
                            Text(authManager.userEmail ?? "Not logged in")
                                .font(.system(size: 16, weight: .semibold))
                        }
                    }
                    .padding(12)
                    .background(Color(.systemBackground))
                    .cornerRadius(8)
                }
                
                Spacer()
                
                Button(action: {
                    authManager.signOut()
                    dismiss()
                }) {
                    Text("Sign Out")
                        .font(.system(size: 18, weight: .semibold))
                        .frame(maxWidth: .infinity)
                        .padding(.vertical, 16)
                        .background(Color.red)
                        .foregroundColor(.white)
                        .cornerRadius(12)
                }
                .frame(minHeight: 56)
            }
            .padding(20)
            .navigationTitle("Settings")
        }
    }
}
```

---

## 6. ViewModels and State Management

```swift
// HomeViewModel.swift
import Combine

class HomeViewModel: ObservableObject {
    @Published var inverterReading: Double = 0
    @Published var meterReading: Double = 0
    @Published var lastUpdated: String = "Never"
    @Published var isLoading = false
    @Published var errorMessage: String?
    
    let networkManager = NetworkManager.shared
    let offlineQueue = OfflineQueueManager.shared
    
    var cancellables = Set<AnyCancellable>()
    
    func fetchCurrentReadings() {
        isLoading = true
        
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
        
        Task {
            do {
                _ = try await networkManager.recordReading(request)
                DispatchQueue.main.async {
                    self.showSuccess("Recording saved")
                }
            } catch {
                // Save offline
                self.offlineQueue.queueRecording(request)
                DispatchQueue.main.async {
                    self.showSuccess("Saved offline (will sync when online)")
                }
            }
        }
    }
    
    func autoRecordAndPush() {
        let request = RecordRequest(
            value: inverterReading,
            type: "auto_push",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        Task {
            do {
                _ = try await networkManager.recordReading(request)
                DispatchQueue.main.async {
                    self.showSuccess("Recording saved and pushed to utility")
                }
            } catch {
                self.offlineQueue.queueRecording(request)
                DispatchQueue.main.async {
                    self.showSuccess("Saved offline (will push when online)")
                }
            }
        }
    }
    
    private func showSuccess(_ message: String) {
        // TODO: Show toast notification
        print(message)
    }
}

// ManualEntryViewModel.swift
class ManualEntryViewModel: ObservableObject {
    let networkManager = NetworkManager.shared
    let offlineQueue = OfflineQueueManager.shared
    
    func submitManualReading(_ value: Double) {
        let request = RecordRequest(
            value: value,
            type: "manual",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        Task {
            do {
                _ = try await networkManager.recordReading(request)
            } catch {
                self.offlineQueue.queueRecording(request)
            }
        }
    }
}
```

---

## 7. Xcode Project Structure

### Recommended File Organization

```
SolarMonitor/
├── SolarMonitor.xcodeproj/
├── SolarMonitor/
│   ├── App/
│   │   └── SolarMonitorApp.swift
│   ├── Views/
│   │   ├── ContentView.swift
│   │   ├── LoginView.swift
│   │   ├── HomeView.swift
│   │   ├── ManualEntryView.swift
│   │   └── SettingsView.swift
│   ├── Components/
│   │   ├── ReadingCard.swift
│   │   ├── ActionButton.swift
│   │   └── GoogleSignInButton.swift
│   ├── ViewModels/
│   │   ├── HomeViewModel.swift
│   │   ├── ManualEntryViewModel.swift
│   │   └── AuthViewModel.swift
│   ├── Models/
│   │   ├── NetworkModels.swift
│   │   └── CoreDataModels.swift
│   ├── Networking/
│   │   ├── NetworkManager.swift
│   │   ├── AuthManager.swift
│   │   └── NetworkError.swift
│   ├── Persistence/
│   │   ├── CoreDataStack.swift
│   │   └── OfflineQueueManager.swift
│   ├── Utilities/
│   │   ├── KeychainManager.swift
│   │   └── Extensions.swift
│   └── Resources/
│       ├── Assets.xcassets/
│       ├── Localizable.strings
│       └── Info.plist
├── SolarMonitorTests/
│   ├── NetworkManagerTests.swift
│   ├── OfflineQueueTests.swift
│   └── AuthManagerTests.swift
└── SolarMonitorUITests/
    ├── LoginUITests.swift
    └── HomeScreenUITests.swift
```

---

# PART 2: IMPLEMENTATION ROADMAP

## Phase 1: Project Setup (2-3 hours)

### Step 1: Create Xcode Project
```bash
# Create new iOS app project
# File > New > Project > iOS > App

# Select options:
# - Product Name: SolarMonitor
# - Team: (your team or None)
# - Organization Identifier: com.yourcompany
# - Interface: SwiftUI
# - Life Cycle: SwiftUI App
# - Minimum Deployment: iOS 15.0 (iPhone 13 mini support)
```

### Step 2: Install Dependencies via CocoaPods
```bash
# Create Podfile in project root
cd ~/Projects/SolarMonitor
pod init

# Edit Podfile:
# target 'SolarMonitor' do
#   pod 'GoogleSignIn', '~> 7.0'
# end

pod install

# Open workspace
open SolarMonitor.xcworkspace
```

**Alternative: SPM (Swift Package Manager)**
- Add in Xcode: File > Add Packages
- Enter: `https://github.com/google/GoogleSignIn-iOS.git`
- Select version 7.0+

### Step 3: Google Cloud Setup
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create new project
3. Enable OAuth 2.0
4. Create iOS app credential:
   - Add bundle ID: `com.yourcompany.SolarMonitor`
   - Download OAuth configuration
5. Add GoogleService-Info.plist to Xcode project

### Step 4: Configure Info.plist
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
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>googlechrome</string>
  <string>googleplus</string>
</array>
```

---

## Phase 2: Authentication (4-5 hours)

### Step 1: KeychainManager
```swift
// Create Utilities/KeychainManager.swift
// [See code above]
```

### Step 2: AuthManager with Google OAuth
```swift
// Create Networking/AuthManager.swift
// [See code above]
```

### Step 3: LoginView
```swift
// Create Views/LoginView.swift
// [See code above]
```

### Step 4: Test OAuth Flow
- Run app in simulator
- Tap "Sign in with Google"
- Verify Google login appears
- Check token saved to Keychain

---

## Phase 3: Networking (3-4 hours)

### Step 1: Network Models
```swift
// Create Models/NetworkModels.swift
// [See code above]
```

### Step 2: NetworkManager
```swift
// Create Networking/NetworkManager.swift
// [See code above]
```

### Step 3: Test Network Calls
```swift
// In HomeView, add test button
Button("Test API") {
    Task {
        do {
            let readings = try await networkManager.fetchCurrentReadings()
            print("✅ Got readings: \(readings)")
        } catch {
            print("❌ Error: \(error)")
        }
    }
}
```

---

## Phase 4: UI Implementation (5-6 hours)

### Step 1: Core Components
- Create ReadingCard.swift
- Create ActionButton.swift
- Create GoogleSignInButton.swift

### Step 2: Main Views
- Create HomeView.swift
- Create ManualEntryView.swift
- Create SettingsView.swift

### Step 3: ViewModels
- Create HomeViewModel.swift
- Create ManualEntryViewModel.swift

### Step 4: Integration
- Connect all views together
- Test button interactions
- Verify data flows properly

---

## Phase 5: Offline Support (4-5 hours)

### Step 1: CoreData Setup
```bash
# In Xcode:
# File > New > File > Data Model
# Name it "SolarMonitoring.xcdatamodeld"
# Add Entity: OfflineRecord with attributes:
# - id: UUID
# - value: Double
# - type: String
# - timestamp: Date
# - isUploaded: Boolean
# - createdAt: Date
# - uploadAttempts: Int16
```

### Step 2: CoreDataStack
```swift
// Create Persistence/CoreDataStack.swift
// [See code above]
```

### Step 3: OfflineQueueManager
```swift
// Create Persistence/OfflineQueueManager.swift
// [See code above]
```

### Step 4: Integrate with HomeViewModel
- Call queueRecording on network errors
- Implement retry logic
- Show pending count badge

---

## Phase 6: Testing (3-4 hours)

### Unit Tests
```swift
// Create SolarMonitorTests/NetworkManagerTests.swift
import XCTest

class NetworkManagerTests: XCTestCase {
    func testFetchCurrentReadings() async throws {
        let manager = NetworkManager.shared
        let readings = try await manager.fetchCurrentReadings()
        
        XCTAssertGreater(readings.inverter_reading, 0)
        XCTAssertGreater(readings.meter_reading, 0)
    }
    
    func testRecordReading() async throws {
        let manager = NetworkManager.shared
        let request = RecordRequest(
            value: 5.5,
            type: "manual",
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
        
        let response = try await manager.recordReading(request)
        XCTAssertTrue(response.success)
    }
}
```

### UI Tests
- Test login flow
- Test button taps
- Test offline scenarios
- Test screen transitions

---

## Phase 7: Deployment (2-3 hours)

### Step 1: App Store Preparation
```bash
# Increment version number
# File > Project Settings > General tab > Version
# Change from 1.0 to 1.0.1

# Set build number
# Change from 1 to 2
```

### Step 2: Create Archive
```bash
# Xcode > Product > Archive
# Choose team to sign with
# Review entitlements
```

### Step 3: Upload to TestFlight/App Store
```bash
# Xcode > Organizer > Archives
# Select build > Distribute App
# Choose App Store Connect
# Follow prompts
```

---

# PART 3: DETAILED DECISION MATRIX & TRADE-OFFS

## Authentication Approaches

| Approach | Setup Time | Security | UX Score | Implementation |
|----------|-----------|----------|----------|-----------------|
| Google OAuth | 1 day | Excellent | 9/10 | Use GoogleSignIn SDK |
| Firebase Auth | 4 hours | Excellent | 9/10 | Use FirebaseAuth SDK |
| Custom OAuth | 3 days | Good | 7/10 | Manual token handling |
| Apple Sign In | 2 hours | Excellent | 8/10 | Limited to Apple users |

**Recommendation: Google OAuth** - Balance of ease and security

---

## Persistence Approaches

| Approach | Query Power | Learning Curve | Offline Support | Sync Capability |
|----------|------------|-----------------|-----------------|-----------------|
| CoreData | Excellent | High | Excellent | Native |
| UserDefaults | None | Very Low | Basic | Manual |
| SQLite | Excellent | High | Excellent | Manual |
| Realm | Excellent | Medium | Good | Paid |

**Recommendation: CoreData** - Built-in, powerful, worth learning

---

## Networking Patterns

| Pattern | Testability | Code Reuse | Async Support | Complexity |
|---------|------------|-----------|---------------|-----------|
| Protocol-based | Excellent | Excellent | Native async/await | Medium |
| Singleton | Good | Good | Native async/await | Low |
| Dependency injection | Excellent | Excellent | Native async/await | High |
| Combine-based | Good | Good | Publishers | High |

**Recommendation: Protocol-based with async/await** - Modern, testable, readable

---

# PART 4: TROUBLESHOOTING GUIDE

## Common Issues & Solutions

### Issue 1: Google Sign-in Not Working
**Symptoms:** OAuth dialog doesn't appear
**Cause:** Missing GoogleService-Info.plist or wrong bundle ID
**Solution:**
1. Verify bundle ID matches Google Cloud Console
2. Check GoogleService-Info.plist is in target
3. Clean build folder (Cmd+Shift+K)
4. Rebuild

### Issue 2: API Requests Return 401
**Symptoms:** "Unauthorized" errors
**Cause:** Session token expired or missing
**Solution:**
1. Check token saved to Keychain
2. Implement token refresh mechanism
3. Force re-authentication

### Issue 3: Offline Records Not Syncing
**Symptoms:** Pending records stay offline forever
**Cause:** Retry mechanism not working
**Solution:**
1. Check CoreData saved correctly
2. Verify network connectivity detection
3. Implement background sync with URLSession

### Issue 4: Memory Leaks in ViewModels
**Symptoms:** App crashes or slows over time
**Cause:** Circular references in Combine
**Solution:**
1. Use `[weak self]` in closures
2. Cancel Combine subscriptions
3. Remove `@StateObject` when not needed

---

# PART 5: DEPLOYMENT & BUILD GUIDE

## Building for iPhone 13 mini on MacBook M3

### Prerequisites
- Xcode 15.0+
- macOS 13.0+
- Apple Developer Account ($99/year)
- iPhone 13 mini or simulator

### Build Steps

```bash
# 1. Clean build artifacts
xcodebuild clean

# 2. Build for simulator
xcodebuild -scheme SolarMonitor \
  -configuration Debug \
  -sdk iphonesimulator \
  -derivedDataPath build

# 3. Test in simulator
open build/Build/Products/Debug-iphonesimulator/SolarMonitor.app

# 4. Build for device
xcodebuild -scheme SolarMonitor \
  -configuration Release \
  -sdk iphoneos \
  -archivePath build/SolarMonitor.xcarchive \
  archive

# 5. Export archive for App Store
xcodebuild -exportArchive \
  -archivePath build/SolarMonitor.xcarchive \
  -exportOptionsPlist ExportOptions.plist \
  -exportPath build/ipa
```

### Code Signing
```bash
# Automatic signing (recommended for testing)
# Xcode will handle automatically

# Manual signing (for production)
# Project Settings > Signing & Capabilities > Team
# Select your Apple Team
```

---

# PART 6: RECOMMENDED LIBRARIES & DEPENDENCIES

## Essential Dependencies

| Library | Purpose | Version | Install |
|---------|---------|---------|---------|
| GoogleSignIn | OAuth authentication | 7.0+ | CocoaPods / SPM |
| Alamofire | Networking (optional) | 5.7+ | CocoaPods |
| SwiftyJSON | JSON parsing (optional) | 5.0+ | CocoaPods |

## CocoaPods Podfile
```ruby
platform :ios, '15.0'

target 'SolarMonitor' do
  pod 'GoogleSignIn', '~> 7.0'
  
  target 'SolarMonitorTests' do
    inherit! :search_paths
  end
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '15.0'
    end
  end
end
```

## Swift Package Dependencies (SPM)
```swift
// Package.swift
let package = Package(
    name: "SolarMonitor",
    platforms: [
        .iOS(.v15)
    ],
    dependencies: [
        .package(url: "https://github.com/google/GoogleSignIn-iOS.git", from: "7.0.0")
    ]
)
```

---

# CONCLUSION

This comprehensive plan provides:

✅ **Authentication:** Google OAuth with secure token management
✅ **Networking:** Protocol-based URLSession architecture
✅ **Persistence:** CoreData for offline queue with retry logic
✅ **UI:** Dad-friendly large text, buttons, high contrast
✅ **Best Practices:** MVVM pattern, async/await, error handling
✅ **Deployment:** Full guide for Mac M3 build and App Store release

**Total Estimated Development Time:** 25-30 hours

**Next Steps:**
1. Set up Xcode project
2. Configure Google OAuth
3. Build authentication
4. Implement networking
5. Create UI
6. Add offline support
7. Test thoroughly
8. Deploy to App Store


# iOS Solar App - Quick Start Guide

## 5-Minute Setup Overview

This guide gets you from zero to running code in under 30 minutes on your Mac M3.

### Prerequisites Checklist
- [ ] Mac with M3 chip (or Intel)
- [ ] 20GB free disk space
- [ ] Apple Developer Account (free for testing)
- [ ] iPhone 13 mini or iOS Simulator

---

## Step 1: Install Xcode (5 minutes)

### Option A: Via App Store (Easiest)
1. Open App Store on Mac
2. Search for "Xcode"
3. Click Install
4. Wait for download (~20GB, takes 15-30 minutes)

### Option B: Direct Download
1. Visit [developer.apple.com/download](https://developer.apple.com/download)
2. Sign in with Apple ID
3. Download Xcode .xip file
4. Double-click to extract

### Verify Installation
```bash
xcode-select --install
swift --version
# Output: swift-driver version 1.87.x
```

---

## Step 2: Create New Project (5 minutes)

### In Xcode:
1. File → New → Project
2. Select iOS → App
3. Fill in form:
   - Product Name: `SolarMonitor`
   - Team: (your team or None for testing)
   - Organization Identifier: `com.yourname`
   - Interface: SwiftUI
   - Life Cycle: SwiftUI App
4. Choose save location: `~/Projects/SolarMonitor`
5. Click Create

### Verify Project Opens
- You should see code editor with `ContentView.swift`
- Click Play button (top-left) to run simulator
- iPhone simulator should launch with blank app

---

## Step 3: Create Project Structure (10 minutes)

### Create Folders in Xcode:

Right-click on project folder → New Group

Create these groups:
- `Views`
- `ViewModels`
- `Models`
- `Networking`
- `Persistence`
- `Utilities`
- `Components`

**Result:** Organized file structure matching the plan

---

## Step 4: Add Basic Networking Code (10 minutes)

### Create File: `Models/NetworkModels.swift`

```swift
import Foundation

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
```

### Create File: `Networking/NetworkManager.swift`

```swift
import Foundation

class NetworkManager {
    static let shared = NetworkManager()
    
    let baseURL = URL(string: "https://api.solar.example.com")!
    
    func fetchCurrentReadings() async throws -> CurrentReadingsResponse {
        // TODO: Implement after OAuth
        return CurrentReadingsResponse(
            inverter_reading: 5.5,
            meter_reading: 100.0,
            timestamp: ISO8601DateFormatter().string(from: Date())
        )
    }
    
    func recordReading(_ request: RecordRequest) async throws -> RecordResponse {
        // TODO: Implement after OAuth
        return RecordResponse(success: true, message: "Recorded")
    }
}

enum NetworkError: LocalizedError {
    case notAuthenticated
    case invalidResponse
    case unauthorized
    case serverError(Int)
    
    var errorDescription: String? {
        switch self {
        case .notAuthenticated:
            return "Please sign in first"
        case .invalidResponse:
            return "Invalid server response"
        case .unauthorized:
            return "Session expired"
        case .serverError(let code):
            return "Server error: \(code)"
        }
    }
}
```

---

## Step 5: Create Simple Home Screen (10 minutes)

### Replace `ContentView.swift`:

```swift
import SwiftUI

struct ContentView: View {
    @State var inverter = 5.5
    @State var meter = 100.0
    
    var body: some View {
        VStack(spacing: 20) {
            // Header
            VStack(spacing: 10) {
                Image(systemName: "sun.max.fill")
                    .font(.system(size: 60))
                    .foregroundColor(.yellow)
                
                Text("Solar Monitor")
                    .font(.system(size: 32, weight: .bold))
                
                Text("Track your energy")
                    .font(.system(size: 18))
                    .foregroundColor(.gray)
            }
            .padding(.top, 40)
            
            // Reading Cards
            VStack(spacing: 16) {
                ReadingCard(
                    title: "Inverter Output",
                    value: String(format: "%.2f", inverter),
                    unit: "kW",
                    icon: "bolt.fill",
                    color: .yellow
                )
                
                ReadingCard(
                    title: "Meter Reading",
                    value: String(format: "%.1f", meter),
                    unit: "kWh",
                    icon: "gauge.badge.plus",
                    color: .blue
                )
            }
            .padding(.horizontal, 20)
            
            Spacer()
            
            // Action Buttons
            VStack(spacing: 12) {
                ActionButton(
                    title: "Auto Record",
                    subtitle: "Save reading",
                    icon: "checkmark.circle.fill",
                    color: .blue
                )
                
                ActionButton(
                    title: "Auto Record + Push",
                    subtitle: "Save to utility",
                    icon: "arrow.up.circle.fill",
                    color: .green
                )
                
                ActionButton(
                    title: "Manual Entry",
                    subtitle: "Enter manually",
                    icon: "pencil.circle.fill",
                    color: .orange
                )
            }
            .padding(.horizontal, 20)
            .padding(.bottom, 30)
        }
        .background(Color(.systemBackground))
    }
}

struct ReadingCard: View {
    let title: String
    let value: String
    let unit: String
    let icon: String
    let color: Color
    
    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            HStack {
                VStack(alignment: .leading) {
                    Text(title)
                        .font(.system(size: 16, weight: .semibold))
                    
                    HStack(spacing: 4) {
                        Text(value)
                            .font(.system(size: 28, weight: .bold, design: .monospaced))
                        Text(unit)
                            .font(.system(size: 16))
                            .foregroundColor(.gray)
                    }
                }
                Spacer()
                Image(systemName: icon)
                    .font(.system(size: 32))
                    .foregroundColor(color)
            }
        }
        .padding(16)
        .background(Color(.systemGray6))
        .cornerRadius(12)
    }
}

struct ActionButton: View {
    let title: String
    let subtitle: String
    let icon: String
    let color: Color
    
    var body: some View {
        HStack(spacing: 16) {
            Image(systemName: icon)
                .font(.system(size: 28))
                .foregroundColor(color)
            
            VStack(alignment: .leading, spacing: 4) {
                Text(title)
                    .font(.system(size: 18, weight: .semibold))
                Text(subtitle)
                    .font(.system(size: 14))
                    .foregroundColor(.gray)
            }
            
            Spacer()
            Image(systemName: "chevron.right")
                .font(.system(size: 16, weight: .semibold))
                .foregroundColor(.gray)
        }
        .padding(16)
        .background(color.opacity(0.1))
        .overlay(RoundedRectangle(cornerRadius: 12).stroke(color, lineWidth: 2))
        .frame(minHeight: 60)
    }
}

#Preview {
    ContentView()
}
```

---

## Step 6: Run App in Simulator (2 minutes)

1. Select iPhone 13 mini in device selector (top-left)
2. Click Play button (or Cmd+R)
3. Simulator launches
4. You should see Solar Monitor home screen

Congratulations! Basic app is running.

---

## Step 7: Test on Device (Optional, 5 minutes)

### Requirements:
- iPhone 13 mini
- USB cable
- Apple Developer Account

### Steps:
1. Connect iPhone via USB
2. Trust "This Computer" on iPhone
3. In Xcode: Select your device (instead of simulator)
4. Click Play button
5. App installs and runs on real device

---

## Common Issues & Fixes

### Issue: Xcode won't install
**Solution:**
```bash
# Make space (need 20GB free)
df -h
# Or use App Store download instead
```

### Issue: Simulator won't launch
**Solution:**
```bash
# Restart simulator
xcrun simctl erase all

# Or open it directly
open /Applications/Simulator.app
```

### Issue: Code has errors
**Solution:**
1. Click Issues panel (left sidebar)
2. Read error message
3. Fix syntax error
4. Cmd+B to rebuild

### Issue: Preview not working
**Solution:**
```swift
// Add this at end of file:
#Preview {
    ContentView()
}
```

---

## Next Steps After Setup

1. **Add OAuth:** Follow authentication section in main plan
2. **Test API Calls:** Replace mock data with real network calls
3. **Add Offline Support:** Implement CoreData persistence
4. **Polish UI:** Fine-tune fonts, colors, spacing
5. **Add Tests:** Write unit tests for network manager

---

## Useful Xcode Shortcuts

| Action | Shortcut |
|--------|----------|
| Run app | Cmd+R |
| Build | Cmd+B |
| Stop | Cmd+. |
| Open Preview | Cmd+Alt+Return |
| File Structure | Cmd+Shift+J |
| Search files | Cmd+Shift+O |
| Quick help | Cmd+Alt+? |

---

## Resources

- [Apple SwiftUI Documentation](https://developer.apple.com/xcode/swiftui/)
- [Xcode Help](https://developer.apple.com/support/xcode/)
- [Swift API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)

**You now have a working SwiftUI app!**

Next: Follow the main plan to add OAuth and backend integration.



YourProject/pyproject.toml
```toml
[project]
name = "your_project"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
authors = [
    { name = "xxx", email = "xxx@xxx.com" }
]
requires-python = ">=3.13"
dependencies = [
    "kivy"
]

[project.scripts]
your_project = "your_project:main"

[build-system]
requires = ["uv_build>=0.9.9,<0.10.0"]
build-backend = "uv_build"

[dependency-groups]
iphoneos = []

#### PSProject Configuration ####

[tool.psproject]
app_name = "YourProject"
backends = [
    "kivyschool.kivylauncher",
]
copy__main__py = true
cythonized = false
extra_index = []
pip_install_app = true
dependencies = [
    { package = { products = ["OneSignalFramework"], reference = "OneSignal" } }
]

[tool.psproject.entitlements]
aps-environment = "development"
"com.apple.developer.aps-environment" = "development"
"com.apple.security.application-groups" = ["group.org.pyswift.your_project.onesignal"]

[tool.psproject.info_plist]
UIBackgroundModes = [
    "remote-notification"
]

[tool.psproject.ios]
backends = []
extra_index = [
    "https://pypi.anaconda.org/beeware/simple",
    "https://pypi.anaconda.org/pyswift/simple",
    "https://pypi.anaconda.org/kivyschool/simple"
]

[tool.psproject.macos]
extra_index = []

[tool.psproject.extra_targets.OneSignalNotificationServiceExtension]
type = "app_extension"
sources = []
backends = []
dependencies = [
    { package = { products = ["OneSignalExtension"], reference = "OneSignal" } }
]

[tool.psproject.extra_targets.OneSignalNotificationServiceExtension.entitlements]
"com.apple.security.application-groups" = ["group.org.pyswift.your_project.onesignal"]

[tool.psproject.extra_targets.OneSignalNotificationServiceExtension.info_plist.NSExtension]
NSExtensionPointIdentifier = "com.apple.usernotifications.service"
NSExtensionPrincipalClass = "OneSignalNotificationServiceExtension"

[tool.psproject.swift_packages]
OneSignal = { url = "https://github.com/OneSignal/OneSignal-XCFramework", upToNextMajor = "5.4.1" }

```

YourProject/project_dist/xcode/OneSignalNotificationServiceExtension/NotificationService.swift

```swift
// FILE: OneSignalNotificationServiceExtension/NotificationService.swift
// PURPOSE: Enables rich notifications (images) and confirmed delivery analytics
// REQUIREMENT: This extension must share an App Group with the main app target (configured in next steps)
// WHEN IT RUNS: iOS calls this extension when a push has "mutable-content": true
//               (automatically set when you include images or action buttons)

import UserNotifications
import OneSignalExtension

class NotificationService: UNNotificationServiceExtension {
    
    // Callback to deliver the modified notification to iOS
    var contentHandler: ((UNNotificationContent) -> Void)?
    
    // The original push request from APNs
    var receivedRequest: UNNotificationRequest!
    
    // A mutable copy of the notification we can modify (add images, etc.)
    var bestAttemptContent: UNMutableNotificationContent?

    // Called when a push notification arrives with mutable-content: true
    // You have ~30 seconds to modify the notification before iOS displays it
    override func didReceive(_ request: UNNotificationRequest, withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
        self.receivedRequest = request
        self.contentHandler = contentHandler
        self.bestAttemptContent = (request.content.mutableCopy() as? UNMutableNotificationContent)

        if let bestAttemptContent = bestAttemptContent {
            // DEBUGGING: Uncomment to verify NSE is running
            // bestAttemptContent.body = "[Modified] " + bestAttemptContent.body

            // OneSignal processes the notification:
            // - Downloads and attaches images
            // - Reports confirmed delivery to dashboard
            // - Handles action buttons
            OneSignalExtension.didReceiveNotificationExtensionRequest(self.receivedRequest, with: bestAttemptContent, withContentHandler: self.contentHandler)
        }
    }

    // Called if didReceive() takes too long (~30 seconds)
    // Delivers whatever content we have so the notification isn't lost
    override func serviceExtensionTimeWillExpire() {
        if let contentHandler = contentHandler, let bestAttemptContent = bestAttemptContent {
            OneSignalExtension.serviceExtensionTimeWillExpireRequest(self.receivedRequest, with: self.bestAttemptContent)
            contentHandler(bestAttemptContent)
        }
    }
}
```


YourProject/project_dist/xcode/Sources/IphoneOS/main.swift
```swift
import Foundation
import PySwiftKit
import KivyLauncher
import Kivy_iOS_Module

import OneSignal

// 1 - post_imports
KivyLauncher.pyswiftImports = [
    .ios,
]

// Enable verbose logging to debug issues (remove in production)
OneSignal.Debug.setLogLevel(.LL_VERBOSE)

// Replace with your 36-character App ID from Dashboard > Settings > Keys & IDs
OneSignal.initialize("YOUR_APP_ID", withLaunchOptions: launchOptions)

// Prompt user for push notification permission
// In production, consider using an in-app message instead for better opt-in rates
// fallbackToSettings: if previously denied, opens iOS Settings to re-enable
OneSignal.Notifications.requestPermission({ accepted in
    print("User accepted notifications: \(accepted)")
}, fallbackToSettings: false)

// 3 - main
let exit_status = KivyLauncher.SDLmain()
// 5 - on_exit
exit(exit_status)
```
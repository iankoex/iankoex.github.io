---
title: "Building Windows Apps with Swift and WinUI SDK"
image: "/assets/images/logo.png"
author: "iankoex"
date: 2026-05-18 11:12:58 +0600
description: "A practical guide to building modern Windows desktop applications using Swift and the WinUI SDK."
tags: [Swift, Windows, WINUI3]
---

Swift on Windows has come a long way since Apple first open-sourced the language. What started as a compiler toolchain for Windows has evolved into a viable option for building native desktop applications. With Microsoft's WinUI SDK providing a modern, fluent design system for Windows apps, combining it with Swift gives developers a powerful and expressive way to create polished Windows experiences.

In this guide, I'll walk you through setting up a Swift project on Windows, integrating the WinUI SDK, and building a functional desktop application from scratch. Whether you're a Swift developer looking to expand to Windows or a Windows developer curious about Swift's capabilities, this post will give you a solid foundation to start building.

## Installing the Swift Toolchain

Before we begin, you'll need a working Swift toolchain on your Windows machine. As of the time of writing this article, the official Swift toolchain (Swift 6.3.2) has some issues that prevent it from successfully building `.exe` files the way we intend to build them. The specifics of these issues are beyond the scope of this article.

Instead, you'll want to grab a toolchain from The Browser Company. Shout out to them for putting in the work to make Swift on Windows possible, along with other contributors in the community. You can find their releases on GitHub:

[Swift Build Releases](https://github.com/thebrowsercompany/swift-build/releases)

Download the latest release that matches your system architecture and install it. Once installed, you should be able to run `swift --version` from your terminal to verify the installation.

## Required Windows SDKs

Before setting up the project, you need to install two specific Windows SDKs. The bindings and projections we'll be using are built against specific versions, so installing the correct SDKs is critical.

### Windows SDK 10.0.17763

Start by installing the Windows SDK version 10.0.17763. You can install it via `winget`:

```powershell
winget install --id Microsoft.WindowsSDK.10.0.17763
```

> **Note:** There was a typo in the version number on older versions of Windows. If installing version `17763` doesn't work for you, try `17736` instead.

You can also download it manually from the [Windows SDK Downloads Archive](https://learn.microsoft.com/en-us/windows/apps/windows-sdk/downloads-archive).

### Windows App SDK 1.5.1

The Swift projections are built against a specific version of the Windows App SDK. Based on `WindowsAppSDK-VersionInfo.h`, the expected release is:

```c
// Release information
#define WINDOWSAPPSDK_RELEASE_MAJOR                         1
#define WINDOWSAPPSDK_RELEASE_MINOR                         5
#define WINDOWSAPPSDK_RELEASE_PATCH                         1
#define WINDOWSAPPSDK_RELEASE_MAJORMINOR                    0x00010005

#define WINDOWSAPPSDK_RELEASE_CHANNEL                       "stable"
#define WINDOWSAPPSDK_RELEASE_CHANNEL_W                     L"stable"
```

This means you need **Windows App SDK 1.5.1** (specifically `1.5.240311000`). Do not install the latest version — the bindings expect this exact version.

You can download it from the [Windows App SDK Downloads Archive](https://learn.microsoft.com/en-us/windows/apps/windows-app-sdk/downloads-archive).

## Understanding Bindings and Projections

Before diving into WinUI, it's important to understand two key concepts that make Swift and Windows work together: **bindings** and **projections**.

### What Are Bindings?

Bindings are the bridge between Swift and the underlying Windows Runtime (WinRT) APIs. Windows apps are built on COM (Component Object Model) and WinRT, which use a binary interface that Swift doesn't natively understand. Bindings translate these low-level COM interfaces into something Swift can call directly.

Think of bindings as translators at a diplomatic meeting. Windows speaks COM, Swift speaks Swift, and bindings make sure both sides understand each other. They handle memory management, method signatures, and error propagation between the two worlds.

### What Are Projections?

Projections take bindings a step further. A projection is a Swift-friendly layer generated on top of the raw bindings. Instead of dealing with HRESULT return codes, reference counting, and COM-specific patterns, projections expose WinRT APIs using Swift conventions:

- Methods return values directly instead of using output parameters
- Errors are thrown as Swift `Error` types instead of HRESULT codes
- Collections use Swift-native types like `Array` and `Dictionary`
- Async operations map to Swift's `async/await`

For example, without a projection, creating a WinUI window might look like this:

```cpp
// Raw COM-style call
IInspectable* window = nullptr;
HRESULT hr = ActivateInstance(HStringReference(L"Microsoft.UI.Xaml.Window").Get(), &window);
if (FAILED(hr)) { /* handle error */ }
```

With a proper Swift projection, the same operation becomes:

```swift
// Swift projection
let window = Window()
```

### Why This Matters for WinUI

WinUI is built entirely on WinRT. To use it from Swift, you need both the bindings that connect to the runtime and the projections that make the API feel native to Swift. The ecosystem around Swift on Windows has matured to the point where several projects provide these projections, allowing you to write WinUI code that feels like writing Swift on any other platform.

## Project Initialization

For this project, we'll use a pure Swift Package Manager (SPM) setup. No CMake, no Visual Studio project files, no external build tools. Just a clean `Package.swift` and the Swift toolchain. This keeps things simple and portable.

### Creating the Package

Open PowerShell and create a new executable Swift package:

```powershell
swift package init --name App --type executable
```

This will generate a standard SPM project structure with a `Package.swift` manifest, a `Sources` directory, and a `main.swift` entry point.

### Verifying the Build

Before adding any dependencies, let's make sure the basic package builds and runs correctly. Build the package with:

```powershell
swift build --product App
```

Once the build completes, you'll find the executable at `.build\out\Products\Debug-windows\App.exe`. Run it from PowerShell:

```powershell
.build\out\Products\Debug-windows\App.exe
```

You should see `Hello, world!` printed to the console. If you do, your toolchain is working correctly and we're ready to move on.

### Required Packages

To work with WinUI from Swift, we need several packages that provide the bindings and projections for Windows APIs. The original repositories from The Browser Company are:

- [swift-windowsfoundation](https://github.com/thebrowsercompany/swift-windowsfoundation)
- [swift-uwp](https://github.com/thebrowsercompany/swift-uwp)
- [swift-winui](https://github.com/thebrowsercompany/swift-winui)
- [swift-windowsappsdk](https://github.com/thebrowsercompany/swift-windowsappsdk)

### The Archived Repository Problem

Here's where things get tricky. The Browser Company has archived all of these repositories, and the way they originally structured their dependencies is no longer supported by modern SPM.

In their original setup, these packages used **local path dependencies** like this:

```swift
dependencies: [
    .package(path: "../swift-cwinrt"),
    .package(path: "../swift-uwp"),
    .package(path: "../swift-windowsfoundation"),
]
```

If you try to use these packages as-is with a URL-based dependency in your `Package.swift`, you'll run into an error like this:

```
error: package 'swift-windowsfoundation' is required using a revision-based requirement and it depends on local package 'swift-cwinrt', which is not supported
```

This happens because SPM does not allow a package fetched via URL (git revision) to depend on a local path package.

### The Solution: moreSwift

One workaround would be to clone all of The Browser Company's repositories locally and wire them up with path dependencies. But we won't be doing that.

Instead, we'll use repositories maintained by another wonderful community building an open-source, cross-platform UI framework: [moreSwift](https://github.com/moreSwift). They have taken the foundational work from The Browser Company and others, updated the dependency structure to work with modern SPM, and continue to actively maintain these packages.

### Configuring Package.swift

Replace the contents of your `Package.swift` with the following configuration:

```swift
// swift-tools-version: 6.0
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "App",
    dependencies: [
        .package(url: "https://github.com/moreSwift/swift-windowsfoundation.git", exact: "0.1.0"),
        .package(url: "https://github.com/moreSwift/swift-uwp.git", exact: "0.1.0"),
        .package(url: "https://github.com/moreSwift/swift-windowsappsdk.git", exact: "0.1.2"),
        .package(url: "https://github.com/moreSwift/swift-winui.git", exact: "0.1.1"),
    ],
    targets: [
        // Targets are the basic building blocks of a package, defining a module or a test suite.
        // Targets can depend on other targets in this package and products from dependencies.
        .executableTarget(
            name: "App",
            dependencies: [
                .product(name: "WindowsFoundation", package: "swift-windowsfoundation"),
                .product(name: "UWP", package: "swift-uwp"),
                .product(name: "WinAppSDK", package: "swift-windowsappsdk"),
                .product(name: "WinUI", package: "swift-winui"),
            ],
            swiftSettings: [
                .enableUpcomingFeature("ApproachableConcurrency")
            ],
        ),
        .testTarget(
            name: "AppTests",
            dependencies: ["App"],
            swiftSettings: [
                .enableUpcomingFeature("ApproachableConcurrency")
            ],
        ),
    ],
    swiftLanguageModes: [.v6]
)
```

This manifest declares the four moreSwift packages as dependencies and links their products to your `App` target.

Once saved, run `swift package resolve` in PowerShell to fetch the dependencies. SPM will download the packages and resolve the dependency graph.

### Building the App

Now replace the contents of `Sources/App/App.swift` with the following code:

```swift
// The Swift Programming Language
// https://docs.swift.org/swift-book

import Foundation
import WinUI

@main
public class App: SwiftApplication {

    override public func onLaunched(_ args: WinUI.LaunchActivatedEventArgs) {
        let window = Window()
        window.title = "App"

        try! window.activate()

        let button = Button()
        button.content = "Hello World"

        button.click.addHandler { _, _ in
            print("button clicked")
        }

        let panel = StackPanel()
        panel.orientation = .vertical
        panel.spacing = 10
        panel.horizontalAlignment = .center
        panel.verticalAlignment = .center
        panel.children.append(button)
        window.content = panel
    }

}
```

This code defines a `SwiftApplication` subclass that serves as the entry point for your WinUI app. When launched, it creates a window, activates it, and sets up a simple UI with a centered button inside a vertical `StackPanel`. Clicking the button prints a message to the console.

Build and run the app:

```powershell
swift build --product App
.build\out\Products\Debug-windows\App.exe
```

You should see a native Windows window with a "Hello World" button. Click it and watch the console output.

## Troubleshooting

### WindowsAppRuntime Bootstrap DLL Not Found (ARM64)

When running your app, you might encounter a crash that looks like this:

```
💣 Program crashed: Access violation at 0x0000000000000000

Platform: arm64 Windows 11.0 build 26200

Thread 0 crashed:

0 null
1 doInit #1 (showUIOnNoMatch:) in WindowsAppRuntimeInitializer.init(threadingModel:) + 95 in WinAppSDK.dll
  at Sources\WinAppSDK\Initialize.swift:82:17

    80│
    81│         func doInit(showUIOnNoMatch: Bool) throws {
    82│             try CHECKED(Initialize(
      │                 ▲
    83│                 UInt32(WINDOWSAPPSDK_RELEASE_MAJORMINOR),
    84│                 WINDOWSAPPSDK_RELEASE_VERSION_TAG_SWIFT,

2 WindowsAppRuntimeInitializer.init(threadingModel:) + 903 in WinAppSDK.dll
  at Sources\WinAppSDK\Initialize.swift:95:17

3 static SwiftApplication.main() + 67 in WinUI.dll
  at Sources\WinUI\Application\SwiftApplication.swift:46:38

Backtrace took 0.58s
```

This happens because the `WindowsAppRuntimeInitializer` looks for the Bootstrap DLL in the wrong location. Internally, it loads the DLL using:

```swift
private let bootstrapDll = LoadLibraryA("swift-windowsappsdk_CWinAppSDK.resources\\Microsoft.WindowsAppRuntime.Bootstrap.dll")
```

It expects the DLL to be in a `resources` folder, but SPM may not place it there correctly depending on your target architecture. I experienced this issue specifically on ARM64.

#### The Fix

Manually copy the `Microsoft.WindowsAppRuntime.Bootstrap.dll` to the expected resources directory, or edit the source file to point to `swift-windowsappsdk_CWinAppSDK.bundle` instead. Keep in mind that editing the package's source files means you may need to force override or re-apply the change if SPM re-resolves the checkout.

Alternatively, you can copy the DLL from the package's build output to the same directory as your executable before running:

```powershell
Copy-Item ".build\checkouts\swift-windowsappsdk\Sources\CWinAppSDK\resources\Microsoft.WindowsAppRuntime.Bootstrap.dll" -Destination ".build\out\Products\Debug-windows\"
```

This ensures the runtime initializer can find the DLL at launch time.

## Complete Project

You can find the complete project source code on GitHub:

[github.com/iankoex/swift-on-windows-demo-1](https://github.com/iankoex/swift-on-windows-demo-1)

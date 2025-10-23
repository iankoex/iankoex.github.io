---
title: "Improving the Caching and Preloading AVPlayer: Addressing Shortcomings"
image: "/assets/images/logo.png"
author: "iankoex"
date: 2025-10-23 11:12:58 +0600
description: "Building on the previous implementation, this post discusses the shortcomings of the original caching AVPlayer and how we've solved them with a more robust, modular solution."
tags: [Swift, SwiftUI, AVFoundation]
---

In my [previous post](https://blog.iankoex.com/post/building-a-caching-and-preloading-avplayer.html), I showed how to build a basic caching and preloading AVPlayer in Swift. While that implementation worked for simple use cases, it had several shortcomings that became apparent as I used it in more complex applications. In this post, I'll discuss those issues and show how I've addressed them in an improved version.

## Shortcomings of the Previous Implementation

### 1. Excessive Disk Space Usage

The original `VideoCacheManager` would cache entire videos to disk without any size limits. For large videos, this could consume significant storage space, potentially filling up the user's device.

```swift
// Old implementation - caches everything
func appendData(_ data: Data) {
    // ... appends all data without limits
}
```

### 2. Lack of Modularity

Everything was crammed into a single `CachingPlayerItem` class, making it hard to maintain, test, and extend. The caching logic, resource loading, and player management were all intertwined.

### 3. Concurrency Issues

The original code wasn't thread-safe, leading to potential race conditions when multiple requests were made simultaneously.

### 4. No Progress Monitoring or Delegate System

There was no way to monitor caching progress or receive notifications about loading events, making it difficult to provide user feedback.

### 5. Basic Preloader

The preloader was simplistic, with no size limits, no cancellation support, and no way to handle multiple concurrent preloads safely.

### 6. Cache Management Issues

No cache invalidation, cleanup, or retention policies. Cached files could accumulate indefinitely.

### 7. Incorrect Data Appending in Cache Manager

The previous `appendData` method always appended data to the end of the cache file, ignoring the byte offset. This worked for simple sequential downloads but failed when handling HTTP range requests or resuming partial downloads, potentially corrupting the cached file.

```swift
// Old implementation - always appends to end
func appendData(_ data: Data) {
    if let fileHandle = try? FileHandle(forWritingTo: fileURL) {
        fileHandle.seekToEndOfFile()  // Always to the end!
        fileHandle.write(data)
    }
}
```

## Solutions in the Improved Implementation

### Modular Architecture

I've broken down the functionality into focused, single-responsibility classes:

- `CacheManager`: Handles all caching operations
- `ResourceLoader`: Manages AVFoundation resource loading
- `Preloader`: Handles preloading logic
- `AudioVisualService`: Orchestrates the player and caching

```swift
@available(macOS 13, iOS 16, tvOS 14, watchOS 7, *)
public final class CacheManager: Sendable {
    // Focused on caching operations only
    var cacheFileSize: Int { /* ... */ }
    func invalidateCache() throws { /* ... */ }
}
```

### Smart Caching with Size Limits

The new `Preloader` uses an actor-based design with configurable size limits:

```swift
@available(macOS 13, iOS 16, tvOS 14, watchOS 7, *)
public actor Preloader: AudioVisualServiceDelegate {
    let preloadSize: Int

    public init(preloadSize: Int = 5 * 1024 * 1024) { // 5MB default
        self.preloadSize = preloadSize
    }

    public func preload(_ url: URL) {
        // Cancels existing preload and starts new one with limits
    }
}
```

### Async Resource Loading

The `ResourceLoader` now uses Swift's async/await for better concurrency:

```swift
@available(macOS 13, iOS 16, tvOS 14, watchOS 7, *)
extension ResourceLoader: AVAssetResourceLoaderDelegate {
    nonisolated public func resourceLoader(
        _ resourceLoader: AVAssetResourceLoader,
        shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest
    ) -> Bool {
        Task {
            await self.addRequest(loadingRequest)
        }
        return true
    }
}
```

### Delegate System for Progress Monitoring

Added `AudioVisualServiceDelegate` for monitoring caching events:

```swift
public protocol AudioVisualServiceDelegate: Sendable {
    func didCacheData(url: URL, totalBytesCached: Int)
    func didReceiveResponse(url: URL, response: URLResponse)
    func didCompleteCaching(url: URL)
}
```

### Proper Data Appending with Range Support

The new `CacheManager` includes an `appendData(_:offset:)` method that writes data at specific byte offsets, supporting range requests and partial downloads:

```swift
@available(macOS 13, iOS 16, tvOS 14, watchOS 7, *)
extension CacheManager {
    func appendData(_ data: Data, offset: Int) {
        // Writes data at the correct offset, not just appending to end
    }
}
```

### Cache Management and Cleanup

Automatic cache cleanup with retention policies:

```swift
public final class CacheManager: Sendable {
    static let maxCacheRetentionDuration: Double = 60 * 60 * 24 * 7  // 7 days

    public func invalidateCache() throws {
        // Proper cleanup
    }
}
```

## Usage Example

Here's how to use the improved implementation:

```swift
import SwiftUI
import AVKit

struct ContentView: View {
    @State var avService: AudioVisualService = AudioVisualService("https://example.com/video.mp4")

    var body: some View {
        VStack {
            VideoPlayer(player: avService.player)

            HStack {
                Button("Play") { avService.isPlaying = true }
                Button("Pause") { avService.isPlaying = false }
            }
        }
        .padding()
    }
}

// Preloading with size limits
let preloader = Preloader(preloadSize: 5 * 1024 * 1024) // 5MB
preloader.preload(videoURLs)
```

## Key Improvements Summary

- **Modular Design**: Separated concerns for better maintainability
- **Size-Limited Caching**: Prevents excessive disk usage
- **Async/Await**: Modern concurrency with actors
- **Delegate System**: Progress monitoring and event handling
- **Range-Aware Caching**: Proper data appending with offset support
- **Cache Management**: Automatic cleanup and invalidation

The improved implementation addresses all the major shortcomings while maintaining the core caching and preloading functionality. It's more robust, efficient, and suitable for production applications.

Find the full code in the [GitHub repo](https://github.com/iankoex/AudioVisualService). Feel free to contribute and help make it even better!

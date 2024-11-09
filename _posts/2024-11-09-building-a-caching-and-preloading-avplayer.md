---
title: "Building a Caching and Preloading AVPlayer"
image: "/assets/images/logo.png"
author: "iankoex"
date: 2024-11-09 11:12:58 +0600
description: "In this article I will show how I built a caching and preloading AVPlayer in Swift."
tags: [Swift, SwiftUI]
---

I will be building a caching and preloading AVPlayer in Swift. This is a very common use case for many apps that need to play video content. The AVPlayer is a powerful tool that allows you to play video content in your app. However, it does not come with built-in support for caching and preloading.

Like most developers, I went online searching for a solution, but I couldn't find any that wasn't too complicated or didn't descend down to UIKit and AppKit.

Yes, I said it, descend!

#### The Player

First, let's create a simple VideoPlayer that will use AVPlayer to play the video. This is a simple SwiftUI view that takes a URL to the video file and plays it.

```swift
import SwiftUI
import AVKit

struct ContentView: View {
    var avPlayer: AVPlayer = AVPlayer(url: URL(string: "https://apivideo-demo.s3.amazonaws.com/hello.mp4")!)

    var body: some View {
        VStack {
            VideoPlayer(player: avPlayer)

        }
        .padding()
    }
}
```

I hope [api.video](https://api.video) are generous enough to let us use their video.

#### The Cache

`AVFoundation` provides us with a programatic way of caching media files. We can use `AVAssetResourceLoaderDelegate` to intercept the loading of the media file and cache it. `AVAssetResourceLoaderDelegate` has two methods that we can use:

```swift
optional func resourceLoader(_ resourceLoader: AVAssetResourceLoader, shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest) -> Bool

optional func resourceLoader(_ resourceLoader: AVAssetResourceLoader, didCancel loadingRequest: AVAssetResourceLoadingRequest)
```

Our next step would be to move the initialization of the `AVPlayer` to a separate class and implement the `AVAssetResourceLoaderDelegate` protocol.

```swift

import SwiftUI
import AVKit

@Observable
public final class AudioVisualService: Sendable {
    public var player: AVPlayer?
    public let url: String


    public init(_ url: String) {
        self.url = url
        initialisePlayer()
    }

    private func initialisePlayer() {
        guard player == nil else { return }
        guard let url = URL(string: url) else { return }

        let playerItem = AVPlayerItem(url: url)
        playerItem.preferredForwardBufferDuration = TimeInterval(1)
        let player = AVPlayer(playerItem: playerItem)
        player.currentItem?.canUseNetworkResourcesForLiveStreamingWhilePaused = true
        player.automaticallyWaitsToMinimizeStalling = true
        player.allowsExternalPlayback = true

        self.player = player
    }

    private func play() {
        guard player == nil else {
            player?.play()
            return
        }
        initialisePlayer()
        player?.playImmediately(atRate: 0)
    }

    private func pause() {
        player?.pause()
    }
}
```

We wil then create a `AVPlayerItem` subclass that will implement the `AVAssetResourceLoaderDelegate` protocol.

```swift
import AVKit

public final class CachingPlayerItem: AVPlayerItem, Sendable {
    private let url: URL

    nonisolated public init(url: URL) {
        self.url = url
        let asset = AVURLAsset(url: url)

        super.init(asset: asset, automaticallyLoadedAssetKeys: nil)
    }
}
```

We will then implement the `AVAssetResourceLoaderDelegate` protocol in the `CachingPlayerItem` class.

```swift
extension CachingPlayerItem: AVAssetResourceLoaderDelegate {
    public func resourceLoader(_ resourceLoader: AVAssetResourceLoader, shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest) -> Bool {
        return true
    }

    public func resourceLoader(_ resourceLoader: AVAssetResourceLoader, didCancel loadingRequest: AVAssetResourceLoadingRequest) {
        // Cancel the loading request
    }
}
```

<blockquote>
    <p>may the games begin!</p>
</blockquote>

To ensure that our `CachingPlayerItem` calls the delegate methods we need to replace the url scheme with a custom scheme. The updated `CachingPlayerItem` class will look like this:

```swift
import AVKit

public final class CachingPlayerItem: AVPlayerItem, Sendable {
    private let url: URL

    nonisolated public init(url: URL) {
        self.url = url
        let urlWithCustomScheme = Self.replaceScheme(of: url, with: "customcache")
        let asset = AVURLAsset(url: urlWithCustomScheme)

        super.init(asset: asset, automaticallyLoadedAssetKeys: nil)
    }

    nonisolated static func replaceScheme(of url: URL, with scheme: String) -> URL {
        var components = URLComponents(url: url, resolvingAgainstBaseURL: false)
        components?.scheme = scheme
        return components?.url ?? url
    }
}
```

#### The Cache Manager

I will dump the code here and explain it later. Hopefully everything shall become clearer as we go along.

```swift
import Foundation

/// Manages the local caching of video data for smoother playback and offline access.
final class VideoCacheManager: Sendable {
    private let cacheDirectory: URL
    let url: URL
    let identifier: String

    init(for url: URL, identifier: String = "") {
        self.url = url
        self.identifier = identifier + "_"
        self.cacheDirectory = URL.cachesDirectory.appending(path: "VideoCache", directoryHint: .isDirectory)

        let fileManager = FileManager.default
        // Create the cache directory if it doesn't exist
        if !fileManager.fileExists(atPath: cacheDirectory.path) {
            try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true, attributes: nil)
        }
    }

    func fileSize() -> Int {
        let attributes = try? FileManager.default.attributesOfItem(
            atPath: cacheFileURL.path()
        )
        return attributes?[.size] as? Int ?? 0
    }

    /// Returns the URL for a cached file based on the original URL.
    var cacheFileURL: URL {
        cacheDirectory.appending(path: identifier + url.lastPathComponent)
    }

    private var codableURLResponseCachePath: String {
        // for some reason `cacheFileURL.appending(path: ".json").path` doesn't work
        cacheDirectory.appending(path: identifier + url.lastPathComponent + ".json").path
    }

    /// Stores or appends data for the given URL in the cache directory.
    func appendData(_ data: Data) {
        let fileURL = cacheFileURL
        if FileManager.default.fileExists(atPath: fileURL.path) {
            // Append data if the file already exists
            if let fileHandle = try? FileHandle(forWritingTo: fileURL) {
                fileHandle.seekToEndOfFile()
                fileHandle.write(data)
                fileHandle.closeFile()
            }
        } else {
            // Create and write if the file doesn't exist
            try? data.write(to: fileURL, options: .atomic)
        }
    }

    /// Retrieves cached data for the given URL and byte range.
    func cachedData(in range: NSRange) -> Data? {
        let fileURL = cacheFileURL
        guard let fileData = try? Data(contentsOf: fileURL) else { return nil }

        // Check if the requested range is valid
        guard range.location < fileData.count else { return nil }

        // Adjust range length if it goes beyond data bounds
        let adjustedLength = min(range.length, fileData.count - range.location)
        return fileData.subdata(in: range.location..<(range.location + adjustedLength))
    }

    func getCachedResponse() -> URLResponse? {
        var value: CodableURLResponse?
        do {
            if let data = FileManager.default.contents(atPath: codableURLResponseCachePath) {
                value = try JSONDecoder().decode(CodableURLResponse.self, from: data)
            }
        } catch {
            print("Cache: CodableURLResponse from disk could not be decoded")
        }
        return value?.urlResponse
    }

    func cacheURLResponse(_ response: URLResponse) {
        let contentLength = getMaxContentRange(from: response)
        let codableResponse = CodableURLResponse.from(response, with: contentLength)
        if FileManager.default.fileExists(atPath: self.codableURLResponseCachePath) == false {
            try? FileManager.default.removeItem(atPath: codableURLResponseCachePath)
        }
        do {
            let data = try JSONEncoder().encode(codableResponse)
            FileManager.default.createFile(atPath: codableURLResponseCachePath, contents: data, attributes: nil)
        } catch {
            print("Cache: Error while encoding CodableURLResponse")
        }
    }

    var isFullyCached: Bool {
        guard let response = getCachedResponse() else {
            return false
        }
        return response.expectedContentLength == fileSize()
    }

    private func getMaxContentRange(from urlResponse: URLResponse) -> Int? {
        guard let response = urlResponse as? HTTPURLResponse else { return nil }
        guard let contentRange = response.value(forHTTPHeaderField: "Content-Range") else { return nil }
        let components = contentRange.split(separator: "/")
        if components.count == 2, let maxRange = components.last {
            return Int(maxRange)
        }
        return nil
    }
}

fileprivate struct CodableURLResponse: Codable {
    var expectedContentLength: Int
    var suggestedFilename: String?
    var mimeType: String?
    var textEncodingName: String?
    var url: URL?

    var urlResponse: URLResponse {
        URLResponse(
            url: url ?? URL(string: "https://example.com")!,
            mimeType: mimeType,
            expectedContentLength: expectedContentLength,
            textEncodingName: textEncodingName
        )
    }

    static func from(_ urlResponse: URLResponse, with desiredExpectedContentLength: Int?) -> CodableURLResponse {
        return CodableURLResponse(
            expectedContentLength: desiredExpectedContentLength ?? Int(urlResponse.expectedContentLength),
            suggestedFilename: urlResponse.suggestedFilename,
            mimeType: urlResponse.mimeType,
            textEncodingName: urlResponse.textEncodingName,
            url: urlResponse.url
        )
    }
}
```

Back on the `CachingPlayerItem`, we need to add a property to hold the cache manager and begin implementing the caching logic. We will also be using `URLSession` to download the video data and cache it locally.

```swift

public final class CachingPlayerItem: AVPlayerItem, Sendable {
    nonisolated private let cacheManager: VideoCacheManager
    private var urlSession: URLSession?
    private let url: URL
    private var loadingRequests: [AVAssetResourceLoadingRequest] = []

    // MARK: Public init

    nonisolated public init(url: URL, identifier: String = "") {
        self.url = url
        let urlWithCustomScheme = Self.replaceScheme(of: url, with: "customcache")

        let asset = AVURLAsset(url: urlWithCustomScheme)
        self.cacheManager = VideoCacheManager(for: url, identifier: identifier)

        super.init(asset: asset, automaticallyLoadedAssetKeys: nil)
        asset.resourceLoader.setDelegate(self, queue: .global(qos: .userInitiated))
    }

    deinit {
        invalidate()
    }

    func invalidate() {
        self.loadingRequests.forEach { $0.finishLoading() }
        self.invalidateURLSession()
    }

    func invalidateURLSession() {
        self.urlSession?.invalidateAndCancel()
        self.urlSession = nil
    }

    nonisolated static func replaceScheme(of url: URL, with scheme: String) -> URL {
        var components = URLComponents(url: url, resolvingAgainstBaseURL: false)
        components?.scheme = scheme
        return components?.url ?? url
    }
}

extension CachingPlayerItem: AVAssetResourceLoaderDelegate {

    /// Intercepts loading requests to serve cached data or download data as needed.
    nonisolated public func resourceLoader(_ resourceLoader: AVAssetResourceLoader, shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest) -> Bool {

        return true
    }

    nonisolated public func resourceLoader(_ resourceLoader: AVAssetResourceLoader, didCancel loadingRequest: AVAssetResourceLoadingRequest) {
        if let index = self.loadingRequests.firstIndex(of: loadingRequest) {
            self.loadingRequests.remove(at: index)
        }
    }

}

```

#### Full Implementation of CachingPlayerItem

```swift
import Foundation
import AVFoundation

public final class CachingPlayerItem: AVPlayerItem, Sendable {
    nonisolated private let cacheManager: VideoCacheManager
    private var urlSession: URLSession?
    private let url: URL
    private var loadingRequests: [AVAssetResourceLoadingRequest] = []

    // MARK: Public init

    nonisolated public init(url: URL, identifier: String = "") {
        self.url = url
        let urlWithCustomScheme = Self.replaceScheme(of: url, with: "customcache")

        let asset = AVURLAsset(url: urlWithCustomScheme)
        self.cacheManager = VideoCacheManager(for: url, identifier: identifier)

        super.init(asset: asset, automaticallyLoadedAssetKeys: nil)
        asset.resourceLoader.setDelegate(self, queue: .global(qos: .userInitiated))
    }

    deinit {
        invalidate()
    }

    func invalidate() {
        self.loadingRequests.forEach { $0.finishLoading() }
        self.invalidateURLSession()
    }

    func invalidateURLSession() {
        self.urlSession?.invalidateAndCancel()
        self.urlSession = nil
    }

    nonisolated static func replaceScheme(of url: URL, with scheme: String) -> URL {
        var components = URLComponents(url: url, resolvingAgainstBaseURL: false)
        components?.scheme = scheme
        return components?.url ?? url
    }
}

// MARK: AVAssetResourceLoaderDelegate

extension CachingPlayerItem: AVAssetResourceLoaderDelegate {

    /// Intercepts loading requests to serve cached data or download data as needed.
    nonisolated public func resourceLoader(_ resourceLoader: AVAssetResourceLoader, shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest) -> Bool {

        // Try to handle the request from cache
        if handleRequestFromCacheIfPossible(loadingRequest) {
            return true
        }

        createURLSessionThenLoad()
        loadingRequests.append(loadingRequest)
        return true
    }

    nonisolated public func resourceLoader(_ resourceLoader: AVAssetResourceLoader, didCancel loadingRequest: AVAssetResourceLoadingRequest) {
        if let index = self.loadingRequests.firstIndex(of: loadingRequest) {
            self.loadingRequests.remove(at: index)
        }
    }

}

// MARK: AVAssetResourceLoaderDelegate Methods

extension CachingPlayerItem {

    private func createURLSessionThenLoad() {
        guard urlSession == nil else { return }
        let config = URLSessionConfiguration.default
        config.requestCachePolicy = .returnCacheDataElseLoad
        let operationQueue = OperationQueue()
        operationQueue.maxConcurrentOperationCount = 1
        urlSession = URLSession(configuration: config, delegate: self, delegateQueue: operationQueue)
        createDataTaskAndLoad()
    }

    private func createDataTaskAndLoad() {
        var request = URLRequest(url: url)
        // This will cheat the loader to think it is loading some bytes and therefore
        // call resourceLoader with dataRequest which will then be processed from the cache
        let cachedBytes = cacheManager.isFullyCached ? cacheManager.fileSize() - 1 : cacheManager.fileSize()

        // Set range header to resume download from where cache ends
        request.setValue("bytes=\(cachedBytes)-", forHTTPHeaderField: "Range")
        let task = urlSession?.dataTask(with: request)
        task?.resume()
    }

    nonisolated private func handleRequestFromCacheIfPossible(_ loadingRequest: AVAssetResourceLoadingRequest) -> Bool {
        // If the request is for content info, handle from cache if possible
        var request = loadingRequest
        if isContentInformationRequest(request), let cachedResponse = cacheManager.getCachedResponse() {
            fillContentInformationRequest(for: &request, using: cachedResponse)
            return false
        }

        // Check if the data request can be fully served from cache
        guard let dataRequest = request.dataRequest else { return false }
        guard cachedDataIsEnoughToFullfilRequest(dataRequest) else { return false }
        request.finishLoading()
        return true
    }

    nonisolated private func cachedDataIsEnoughToFullfilRequest(_ dataRequest: AVAssetResourceLoadingDataRequest) -> Bool {
        let requestedOffset = Int(dataRequest.requestedOffset)
        let requestedLength = dataRequest.requestedLength
        let cachedBytes = cacheManager.fileSize()
        guard requestedLength > 2 else { return false }

        // Serve the range directly if fully cached
        if cachedBytes >= requestedOffset + requestedLength {
            let range = NSRange(location: requestedOffset, length: requestedLength)
            if let cachedData = cacheManager.cachedData(in: range) {
                dataRequest.respond(with: cachedData)
                return true
            }
        } else if cachedBytes > requestedOffset {
            // Partially cached: Serve what we can
            let availableLength = cachedBytes - requestedOffset
            let range = NSRange(location: requestedOffset, length: availableLength)
            if let cachedData = cacheManager.cachedData(in: range) {
                dataRequest.respond(with: cachedData)
                return true
            }
        }
        return false
    }

    nonisolated private func processRequests() {
        var finishedRequests = Set<AVAssetResourceLoadingRequest>()

        for var request in loadingRequests {
            // Fill information from cache if available
            if isContentInformationRequest(request), let response = cacheManager.getCachedResponse() {
                fillContentInformationRequest(for: &request, using: response)
            }

            // Respond to data requests with cached data
            if let dataRequest = request.dataRequest, checkAndRespond(forRequest: dataRequest) {
                finishedRequests.insert(request)
                request.finishLoading()
            }
        }

        // Remove finished requests
        loadingRequests = loadingRequests.filter { !finishedRequests.contains($0) }
    }

    nonisolated private func isContentInformationRequest(_ request: AVAssetResourceLoadingRequest) -> Bool {
        return request.contentInformationRequest != nil
    }

    nonisolated private func fillContentInformationRequest(for request: inout AVAssetResourceLoadingRequest, using response: URLResponse) {
        request.contentInformationRequest?.isByteRangeAccessSupported = true
        request.contentInformationRequest?.contentType = response.mimeType
        request.contentInformationRequest?.contentLength = response.expectedContentLength
    }

    nonisolated private func checkAndRespond(forRequest dataRequest: AVAssetResourceLoadingDataRequest) -> Bool {
        let downloadedDataLength = cacheManager.fileSize()

        let requestRequestedOffset = Int(dataRequest.requestedOffset)
        let requestRequestedLength = Int(dataRequest.requestedLength)
        let requestCurrentOffset = Int(dataRequest.currentOffset)

        if downloadedDataLength < requestCurrentOffset {
            return false
        }

        let downloadedUnreadDataLength = downloadedDataLength - requestCurrentOffset
        let requestUnreadDataLength = requestRequestedOffset + requestRequestedLength - requestCurrentOffset
        let respondDataLength = min(requestUnreadDataLength, downloadedUnreadDataLength)
        let range = NSRange(location: requestCurrentOffset, length: respondDataLength)
        if let responseData = cacheManager.cachedData(in: range) {
            dataRequest.respond(with: responseData)
        }

        let requestEndOffset = requestRequestedOffset + requestRequestedLength

        return requestCurrentOffset >= requestEndOffset
    }
}

// MARK: URLSessionTaskDelegate

extension CachingPlayerItem: URLSessionTaskDelegate {
    nonisolated public func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        self.invalidateURLSession()
    }
}

// MARK: URLSessionDataDelegate

extension CachingPlayerItem: URLSessionDataDelegate {

    nonisolated public func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {
        // Dont append data if the video is already fully cached
        // This fixes a bug where a byte was added to the end of the file even though the video was fully cached
        // thereby corrupting the video file and making it unplayable
        if cacheManager.isFullyCached == false {
            cacheManager.appendData(data)
        }

        processRequests()
    }

    nonisolated public func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive response: URLResponse, completionHandler: @escaping (URLSession.ResponseDisposition) -> Void) {
        cacheManager.cacheURLResponse(response)
        self.processRequests()
        completionHandler(.allow)
    }
}
```

#### Notes on AVAssetResourceLoaderDelegate

Return true if you can handle the request and of course false when you can't

Check if the request is for content information or data. Content information is metadata about the media file like content type, length etc. Data is the actual media data. Never call `request.finishLoading()` on a content request, it will stall the player without a way of resuming. The content request is usually 2 bytes.

URLSessionDataDelegate didRecieve response will be called and we need to cache the response in the cache manager. This is where we get the content information.
URLResponse is not codable and therefore we created a struct to store the information we need and cache it.

There is a lot of calculations on bytes received, bytes requested and bytes cached to determine if we can serve the request from cache. This is because the player requests data in chunks and we need to make sure we have the data to serve the request.
The first data request is usually the full length of the video and we have it in cache we can return the whole of it.
if not we return the data we have and the player will request the rest of the data in chunks.

When the requests in chunks are made, we return those chunks and cache them to the disk. This ensures that the player can play the video without any interruptions and we are caching the data as it is being played.
We therefore, don't have to download the entire video to play it.

#### The Preloader

Once we have the cache manager and the caching player item, implementing the preloader is actually easy.
We just need to download the specific data requested use the cache manager to cache it.
Later on when the player is initialised and the caching player item is set, the player will play the video from the cache and continue caching from where the preloader left off.

```swift
import Foundation
import AVFoundation

public actor Preloader: Sendable {
    let preloadSize: Int
    var isPreloadingStore: [URL:Bool] = [:]

    public init(preloadSize: Int = 1 * 1024 * 1024) {
        self.preloadSize = preloadSize
    }

    public func preload(_ urls: [URL]) {
        for url in urls {
            preload(url)
        }
    }

    public func preload(_ urls: [(URL, String)]) {
        for url in urls {
            preload(url.0, identifier: url.1)
        }
    }

    public func preload(_ url: URL, identifier: String = "") {
        // Check if already preloading or the cache already meets preloadSize
        let isPreloading = isPreloadingStore[url] ?? false
        guard isPreloading == false else { return }

        let cacheManager = VideoCacheManager(for: url, identifier: identifier)

        // If entire video is cached, skip preload
        if cacheManager.isFullyCached {
            return
        }

        let cachedBytes = cacheManager.fileSize()
        if cachedBytes > preloadSize {
            return
        }

        // Start downloading up to preloadSize
        isPreloadingStore[url] = true
        Task {
            await startPreloading(
                url: url,
                from: cachedBytes,
                upTo: preloadSize,
                using: cacheManager
            )
        }
    }

    private func startPreloading(url: URL, from offset: Int, upTo bytes: Int, using cacheManager: VideoCacheManager) async {
        var request = URLRequest(url: url)
        request.setValue("bytes=\(offset)-\(bytes - 1)", forHTTPHeaderField: "Range")
        if let (data, response) = try? await createURLSession().data(for: request) {
            cacheManager.cacheURLResponse(response)
            cacheManager.appendData(data)
            self.isPreloadingStore[url] = false
        } else {
            self.isPreloadingStore[url] = false
        }
    }

    private func createURLSession() -> URLSession {
        let config = URLSessionConfiguration.default
        config.requestCachePolicy = .returnCacheDataElseLoad
        let operationQueue = OperationQueue()
        operationQueue.maxConcurrentOperationCount = 1
        return URLSession(configuration: config, delegate: nil, delegateQueue: operationQueue)
    }
}
```

#### Disclaimer

This method will cache the entire video to disk. This is not ideal for large videos as it will consume a lot of disk space. You can modify the code to cache only a certain amount of data and delete the rest. This will require you to keep track of the data you have cached and delete the rest. This is left as an exercise for the reader.

This will also not work with HTTP Live Streaming (HLS) videos as they are segmented and the player requests the segments in chunks. You will have to modify the code to handle this. Again an exercise for the reader.
I will also try and implement this in the future to have an all in one solution to all video types.

#### Back to AudioVisualService

we can now have a full example of how to use the caching player item and the preloader.

```swift
import SwiftUI
import AVKit

struct ContentView: View {
    @State var avService: AudioVisualService = AudioVisualService("https://apivideo-demo.s3.amazonaws.com/hello.mp4")

    var body: some View {
        VStack {
            VideoPlayer(player: avService.player)
            HStack {
                Button("Play") {
                    avService.isPlaying = true
                }

                Button("Pause") {
                    avService.isPlaying = false
                }

                Button("preload") {
                    guard let url = URL(string: avService.url) else {
                        return
                    }
                    let preloader = Preloader(preloadSize: 1 * 1024 * 1024) // 1 MB
                    preloader.preload(url)
                }
            }
        }
        .padding()
    }
}

@Observable
public final class AudioVisualService: Sendable {
    public var player: AVPlayer?
    public let url: String

    public var isPlaying: Bool = true {
        didSet {
            isPlaying ? play() : pause()
        }
    }

    public init(_ url: String) {
        self.url = url
        initialisePlayer()
    }

    private func initialisePlayer() {
        guard player == nil else { return }
        guard let url = URL(string: url) else { return }

        let playerItem = CachingPlayerItem(url: url, identifier: "some identifier")
        playerItem.preferredForwardBufferDuration = TimeInterval(1)
        let player = AVPlayer(playerItem: playerItem)
        player.currentItem?.canUseNetworkResourcesForLiveStreamingWhilePaused = true
        player.automaticallyWaitsToMinimizeStalling = true
        player.allowsExternalPlayback = true

        self.player = player
    }

    private func play() {
        guard player == nil else {
            player?.play()
            return
        }
        initialisePlayer()
        player?.playImmediately(atRate: 0)
    }

    private func pause() {
        player?.pause()
    }
}
```

#### Ends

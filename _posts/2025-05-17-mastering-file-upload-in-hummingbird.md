---
title: "Mastering File Uploads in Swift with Hummingbird: A Practical Guide"
image: "/assets/images/logo.png"
author: "iankoex"
date: 2025-05-17 06:12:58 +0600
description: "In this blog post, I'll show how you can handle multipart form data with Hummingbird."
tags: [Swift, SwiftOnServer, HummingBird]
---

Building web applications that manage file uploads is a common requirement. In this blog post, I'll guide you through handling multipart form data using Hummingbird. You'll learn how to effortlessly receive and process files alongside other form fields, ensuring your Swift backend can seamlessly handle user uploads.

## What is Multipart Form Data?

Multipart form data is a specific encoding type used in HTTP requests to facilitate the transmission of files and data in a single request. This format allows for the inclusion of various data types, such as text fields, images, videos, and other binary files. It is essential for web applications that require users to upload files alongside standard form inputs. The multipart format divides the data into distinct parts, each with its own content type and disposition, enabling the server to process each part appropriately.

## The Anatomy of Multipart Form Data

Understanding the structure of multipart form data is crucial for developers working with file uploads and form submissions. Here’s a breakdown of its key components:

1. **Content-Type Header**:
   The request begins with a `Content-Type` header that specifies the type of data being sent. For multipart form data, this header typically looks like this:

    ```
    Content-Type: multipart/form-data; boundary=---BoundaryString
    ```

    The `boundary` parameter is a unique string that acts as a delimiter between different parts of the form data.

2. **Boundary**:
   The boundary string separates each part of the multipart data. It is defined in the `Content-Type` header and indicates where one part ends and another begins. Each part starts with the boundary string, prefixed by two hyphens (`--`), and ends with the boundary string followed by two hyphens to signify the end of the multipart data.

3. **Parts**:
   Each part of the multipart form data consists of several components:
    - **Headers**: Each part can have its own set of headers that provide metadata about the data being sent. Common headers include:
        - `Content-Disposition`: Specifies how the data should be handled, including the name of the form field and, if applicable, the filename for file uploads. For example:
            ```
            Content-Disposition: form-data; name="file"; filename="example.jpg"
            ```
        - `Content-Type`: Indicates the media type of the data being sent (e.g., `image/jpeg` for JPEG images).
    - **Body**: Following the headers, the actual data (the body) of the part is included. This can be text, binary data, or any other type of content.

4. **Final Boundary**:
   The multipart data concludes with a final boundary that indicates the end of the data. This boundary is marked by the same boundary string used to separate the parts, followed by two hyphens:
    ```
    -----BoundaryString--
    ```

## Example Structure

Here’s a simplified example of what a multipart form data request might look like:

```
POST /upload HTTP/1.1
Host: example.com
Content-Type: multipart/form-data; boundary=---BoundaryString

-----BoundaryString
Content-Disposition: form-data; name="username"

john_doe
-----BoundaryString
Content-Disposition: form-data; name="file"; filename="example.jpg"
Content-Type: image/jpeg

(binary data of the image)
-----BoundaryString--
```

## Hummingbird

[Hummingbird](https://hummingbird.codes) is a robust, feature-rich, and performance-oriented web framework for Swift. This guide assumes you have a basic understanding of Swift and Hummingbird. If you're new to Hummingbird, I recommend starting with the [Getting Started guide](https://docs.hummingbird.codes/2.0/documentation/hummingbird/gettingstarted).

## Swift Mustache: A Templating Engine

[Swift Mustache](https://github.com/hummingbird-project/swift-mustache) is a powerful templating engine for Swift that allows you to create dynamic and reusable HTML templates with ease.

## Setting Up Hummingbird and Swift Mustache

Once you have set up Hummingbird and Swift Mustache, you can start building your application. Here’s a simple example of how to set up a route that handles file uploads:

```swift
func buildApplication(_ args: AppArguments) async throws -> some ApplicationProtocol {
    // Verify the working directory is correct
    assert(FileManager.default.fileExists(atPath: "public/images/hummingbird.png"), "Set your working directory to the root folder of this example to get it to work")
    // load mustache template library
    let library = try await MustacheLibrary(directory: Bundle.module.resourcePath!)

    let router = Router(context: MultipartRequestContext.self)
    router.add(middleware: FileMiddleware())
    WebController(mustacheLibrary: library).addRoutes(to: router)
    return Application(router: router, configuration: .init(address: .hostname(args.hostname, port: args.port)))
}
```

Your `WebController` is responsible for handling web requests and rendering views using Mustache templates. Here’s how you can define routes and handle file uploads:

```swift
struct WebController {
    let library: MustacheLibrary
    let enterTemplate: MustacheTemplate
    let enteredTemplate: MustacheTemplate

    init(mustacheLibrary: MustacheLibrary) {
        self.library = mustacheLibrary
        self.enterTemplate = mustacheLibrary.getTemplate(named: "enter-details")!
        self.enteredTemplate = mustacheLibrary.getTemplate(named: "details-entered")!
    }

    func addRoutes(to router: some RouterMethods<some RequestContext>) {
        router.get("/", use: self.input)
        router.post("/", use: self.post)
    }

    @Sendable func input(request: Request, context: some RequestContext) -> HTML {
        let html = self.enterTemplate.render((), library: self.library)
        return HTML(html: html)
    }

    @Sendable func post(request: Request, context: some RequestContext) async throws -> HTML {
        let user = try await request.decode(as: User.self, context: context)
        let html = self.enteredTemplate.render(user, library: self.library)
        return HTML(html: html)
    }
}
```

### Mustache Templates

You will need a few Mustache templates for your application:

- **`page.mustache`**: The default template for pages.
- **`enter-details.mustache`**: A template for entering user details.
- **`details-entered.mustache`**: A template for displaying the entered user details.

Here’s a simple example of what each template might look like:

**page.mustache**:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>HTMLForm</title>
        <meta charset="UTF-8" />
        <script src="https://cdn.tailwindcss.com"></script>
    </head>
    <body class="bg-gray-100">
        <div class="text-center">
            <h2 class="text-3xl p-3">Multipart Form</h2>
        </div>
        <div class="p-6 max-w-md mx-auto bg-white rounded-xl items-centered space-x-4 shadow-lg">
            <div>
                <img src="images/hummingbird.png" class="w-64 mx-auto" />
            </div>
            <!-- Form body content -->
        </div>
    </body>
</html>
```

**enter-details.mustache**:

```html
<h1 class="text-xl">Please enter your details</h1>
<form action="/" method="post" enctype="multipart/form-data">
    <label for="name">Name</label><br />
    <input type="text" id="name" name="name" class="border" /><br />
    <label for="age">Age</label><br />
    <input type="text" id="age" name="age" class="border" /><br />
    <input type="submit" value="Submit" class="p-1 my-2 border shadow-lg" />
</form>
```

**details-entered.mustache**:

```html
<h1 class="text-xl">You entered</h1>
<ul>
    <li>Name: {{name}}</li>
    <li>Age: {{age}}</li>
</ul>
```

## Processing the Uploaded Data

Once you submit the data, you can process it on the server side. You can print the `ByteBuffer` from the request to inspect the content being received. An example of the response you might see is:

```
------WebKitFormBoundaryM54lRf0SWPW07s4q
Content-Disposition: form-data; name="name"

some user
------WebKitFormBoundaryM54lRf0SWPW07s4q
Content-Disposition: form-data; name="age"

24
------WebKitFormBoundaryM54lRf0SWPW07s4q--
```

To decode this response, we use the `FormDataDecoder` provided by `MultipartKit`. First, we need to validate that the Content-Type we are receiving is indeed `multipart/form-data`. The `FormDataDecoder` initializer also requires a `Boundary`, which is the boundary string that separates the different parts of the form data. We can extract this boundary string from the `Content-Type` header of the request. We also need to provide the response decoder with methods to handle this.

### Implementing the Multipart Request Decoder

Here’s how you can implement a `MultipartRequestDecoder` to decode the incoming multipart form data:

```swift
struct MultipartRequestDecoder: RequestDecoder {
    func decode<T>(_ type: T.Type, from request: Request, context: some RequestContext) async throws -> T where T: Decodable {
        let decoder = FormDataDecoder()
        return try await decoder.decode(type, from: request, context: context)
    }
}

extension FormDataDecoder {
    /// Extend JSONDecoder to decode from `HBRequest`.
    /// - Parameters:
    ///   - type: Type to decode
    ///   - request: Request to decode from
    public func decode<T: Decodable>(_ type: T.Type, from request: Request, context: some RequestContext) async throws -> T {
        guard let contentType = request.headers[.contentType],
            let mediaType = MediaType(from: contentType),
            let parameter = mediaType.parameter,
            parameter.name == "boundary"
        else {
            throw HTTPError(.unsupportedMediaType)
        }
        let buffer = try await request.body.collect(upTo: context.maxUploadSize)
        return try self.decode(T.self, from: buffer, boundary: parameter.value)
    }
}
```

### Handling File Uploads

Edit the `enter-details.mustache` to handle file uploads, add the following code:

```html
<input type="file" id="pfp" name="pfp" accept="image/*" class="border" /><br />
```

This adds an input element that will handle file selection, we have specified that only images are accepted. On the server side we can check the response we are getting:

```
------WebKitFormBoundaryDoXFevD2sAF8HAF4
Content-Disposition: form-data; name="name"


------WebKitFormBoundaryDoXFevD2sAF8HAF4
Content-Disposition: form-data; name="age"


------WebKitFormBoundaryDoXFevD2sAF8HAF4
Content-Disposition: form-data; name="pfp"; filename="icon.png"
Content-Type: image/png

�PNG [Binary data truncated]
```

Our model cannot handle the uploaded file because we have not specified `pfp`. Therefore we are getting the error:

```
"error":{"message":"The given data was not valid input."}}
```

To handle file uploads, you need to update the `User` model to include a property for the uploaded file. Here’s how you can modify the model:

```swift
struct User: Decodable {
    let name: String
    let age: Int
    let pfp: File
}
```

The `File` struct is designed to handle the decoding of the multipart data. Here’s its implementation:

```swift
import Hummingbird
import MultipartKit
import StructuredFieldValues

struct File: MultipartPartConvertible, Decodable {
    let data: ByteBuffer
    let filename: String
    let contentType: String

    var multipart: MultipartPart? { nil }

    init?(multipart: MultipartPart) {
        self.data = multipart.body
        guard let contentType = multipart.headers["content-type"].first else { return nil }
        guard let contentDispositionHeader = multipart.headers["content-disposition"].first else { return nil }
        guard let contentDisposition = try? StructuredFieldValueDecoder().decode(MultipartContentDispostion.self, from: contentDispositionHeader) else { return nil }
        guard let filename = contentDisposition.parameters.filename else { return nil }
        self.filename = filename
        self.contentType = contentType
    }
}

struct MultipartContentDispostion: StructuredFieldValue {
    struct Parameters: StructuredFieldValue {
        static let structuredFieldType: StructuredFieldType = .dictionary
        var name: String
        var filename: String?
    }
    static let structuredFieldType: StructuredFieldType = .item
    var item: String
    var parameters: Parameters
}

extension StructuredFieldValueDecoder {
    public func decode<StructuredField: StructuredFieldValue>(
        _ type: StructuredField.Type = StructuredField.self,
        from string: String
    ) throws -> StructuredField {
        let decoded = try string.utf8.withContiguousStorageIfAvailable { bytes in
            try self.decode(type, from: bytes)
        }
        if let decoded {
            return decoded
        }
        var string = string
        string.makeContiguousUTF8()
        return try self.decode(type, from: string)
    }
}
```

### ends

Congratulations! You now have a working setup to handle multipart form data, including file uploads, using Hummingbird and Swift. This guide has covered the essential components of multipart form data, how to set up your application, and how to process user inputs effectively.
Handling multiple file uploads is an exercise left for the reader

You can find an example of handling multiple file uploads in the [Hummingbird Examples Repository](https://github.com/hummingbird-project/hummingbird-examples/tree/main/multipart-form).

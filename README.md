# Graphiti 

Graphiti is a Swift library for building GraphQL schemas/types fast, safely and easily.

[![Swift][swift-badge]][swift-url]
[![License][mit-badge]][mit-url]
[![Slack][slack-badge]][slack-url]
[![GitHub Actions][gh-actions-badge]][gh-actions-url]
[![Maintainability][maintainability-badge]][maintainability-url]
[![Coverage][coverage-badge]][coverage-url]

Looking for help? Find resources [from the community](http://graphql.org/community/).


## Getting Started

An overview of GraphQL in general is available in the
[README](https://github.com/facebook/graphql/blob/master/README.md) for the
[Specification for GraphQL](https://github.com/facebook/graphql). That overview
describes a simple set of GraphQL examples that exist as [tests](Tests/GraphitiTests/StarWarsTests/)
in this repository. A good way to get started with this repository is to walk
through that README and the corresponding tests in parallel.

### Using Graphiti

Add Graphiti to your `Package.swift`

```swift
import PackageDescription

let package = Package(
    dependencies: [
        .Package(url: "https://github.com/GraphQLSwift/Graphiti.git", .upToNextMinor(from: "0.20.1")),
    ]
)
```

Graphiti provides two important capabilities: building a type schema, and
serving queries against that type schema.

#### Defining entities

First, we declare our regular Swift entities.

```swift
struct Message : Codable {
    let content: String
}
```

One of the main design decisions behind Graphiti is **not** to polute your entities declarations. This way you can bring your entities to any other solution with ease.

#### Defining the context

Second step is to create your application's **context**. The context will be passed to all of your field resolver functions. This allows you to apply dependency injection to your API. You will usually use the Context as the state holder of your API. Therefore, this will often be a `class`.

```swift
/**
 * This data is hard coded for the sake of the demo, but you could imagine
 * fetching this data from a database or a backend service instead.
 */
final class MessageContext {
    func message() -> Message {
        Message(content: "Hello, world!")
    }
}
```

Notice again that this step doesn't require Graphiti. It's purely business logic.

#### Defining the API implementation

Now that we have our entities and context we can create the API itself.

```swift
import Graphiti

struct MessageRoot {
    func message(context: MessageContext, arguments: NoArguments) -> Message {
        context.message()
    }
}
```

#### Defining the API

Now we can finally define the Schema using the builder pattern.

```swift
struct MessageAPI : API {
    let root: MessageRoot
    let schema: Schema<MessageRoot, MessageContext>
    
    // Notice that `API` allows dependency injection.
    // You could pass mocked subtypes of `root` and `context` when testing, for example.
    init(root: MessageRoot) throws {
        self.root = root

        self.schema = try Schema<MessageRoot, MessageContext>(
            Type(Message.self,
                Field("content", at: \.content)
            }

            Query(
                Field("message", at: MessageRoot.message)
            )
        )
    }
}
```

#### Querying

To query the schema we need to instantiate the api and pass in an EventLoopGroup to feed the execute function alongside the query itself.

```swift
import NIO

let root = MessageRoot()
let context = MessageContext()
let api = try MessageAPI(root: root)
let group = MultiThreadedEventLoopGroup(numberOfThreads: System.coreCount)
        
defer {
    try? group.syncShutdownGracefully()
}

api.execute(
    request: "{ message { content } }",
    context: context,
    on: group
).whenSuccess { result in
    print(result)
}
```

The output will be:

```json
{"data":{"message":{"content":"Hello, world!"}}}
```

`API.execute` returns a `GraphQLResult` which adopts `Encodable`. You can use it with a `JSONEncoder` to send the response back to the client using JSON.

#### Async resolvers

To use async resolvers, just add one more parameter with type `EventLoopGroup` to the resolver function and change the return type to `EventLoopFuture<YouReturnType>`. Don't forget to import NIO.

```swift
import NIO

struct MessageRoot {
    func message(context: MessageContext, arguments: NoArguments, group: EventLoopGroup) -> EventLoopFuture<Message> {
        group.next().makeSucceededFuture(store.message())
    }
}
```

## License

This project is released under the MIT license. See [LICENSE](LICENSE) for details.

[swift-badge]: https://img.shields.io/badge/Swift-5.2-orange.svg?style=flat
[swift-url]: https://swift.org

[mit-badge]: https://img.shields.io/badge/License-MIT-blue.svg?style=flat
[mit-url]: https://tldrlegal.com/license/mit-license

[slack-badge]: https://zewo-slackin.herokuapp.com/badge.svg
[slack-url]: http://slack.zewo.io

[gh-actions-badge]: https://github.com/GraphQLSwift/Graphiti/workflows/Tests/badge.svg
[gh-actions-url]: https://github.com/GraphQLSwift/Graphiti/actions?query=workflow%3ATests

[maintainability-badge]: https://api.codeclimate.com/v1/badges/25559824033fc2caa94e/maintainability
[maintainability-url]: https://codeclimate.com/github/GraphQLSwift/Graphiti/maintainability

[coverage-badge]: https://api.codeclimate.com/v1/badges/25559824033fc2caa94e/test_coverage
[coverage-url]: https://codeclimate.com/github/GraphQLSwift/Graphiti/test_coverage

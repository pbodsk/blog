---
title: Getting started with Vapor 3
date: 2018-01-02 13:00:00
tags: vapor,swift,linux,swift4,vapor3
authorIds:
- stso
categories:
- Vapor
---

Vapor has been our go-to framework when we develop backend solutions at Nodes since January 2017. A lot has happened during the past year, specially when we saw Vapor 2 got released back in May. Our overall opinion is that Vapor 2 has been a mature and a fairly feature-rich framework we've been enjoying working with. That said, there's still some room for improvement, which has made us follow the development of Vapor 3 with excitement.



As of writing this post, the latest version of the next major version of Vapor is [3.0 Beta 1](https://github.com/vapor/vapor/releases/tag/3.0.0-beta.1). Although the major version hasn't yet hit a stable version, we wanted to take aside some time to have a look at how these changes will affect our daily work. We have developed and help maintain around 25+ customer projects and 30+ open source packages so it's important for us to know the amount of changes needed in order to migrate these projects to Vapor 3.

## Our focus areas

Before diving in and setting up our first Vapor 3 project, let's reflect a bit on what we want to achieve with this research of Vapor 3. Hopefully this will guide our exploration as well as give us some way categorize our findings and compare it with Vapor 2. These are the following areas we will look into when testing out Vapor 3 for this post:

- Configuration: How do we setup providers and middlewares and how do we configure these?
- Models: How do we create a model and how do we query it?
- Routing: How do we setup routes for our project and how does these interface with our controllers?
- Views: How do we setup views for our project?
- Commands: How do we setup and run commands?

Besides looking into the above areas using a new Vapor 3 project, we will also look into what it will require to migrate an existing project to Vapor 3.

## Getting started with Vapor 3

Please note that this post is based on a beta version of the framework and things might change before we see the official release of the next major version of Vapor.

### Setting up a project

First, make sure you have the recent version of the Vapor toolbox:

```bash
brew upgrade vapor
```

And if you don't have the toolbox already installed, then run:

```bash
brew install vapor/tap/vapor
```

Also make sure you're running Swift 4 as this is now required in Vapor 3:

```bash
swift --version
```

With that in place, we can now create our Vapor 3 project. For this project, we're going to use the [`api-template`](https://github.com/vapor/api-template/tree/beta) since that has been updated for Vapor 3 (using the `beta` branch):

```bash
vapor new vapor-3-test --template=api --branch=beta
```

As always, let's make the Vapor toolbox generate our Xcode project for us:

```bash
cd vapor-3-test
vapor xcode
```

When it's done generating the project you should be able to run the project and when visiting `http://localhost:8080/hello` in your browser you should see the `Hello, world!` page.

When looking through the files in our new project, it's already worth noticing some differences. First up, we don't have `Droplet+Setup.swift` and `Config+Setup.swift`. They are instead replaced by:

- `boot.swift`: The `boot` function is called after initialization but before it starts running, meaning that it is an ideal place for running code that should run each time the application starts. Speaking of "application", notice how the signature now calls the `Droplet` an `Application` now. The signature also implies now that there's only going to be one `Application`. 
- `configure.swift`: This is where you would configure and setup your application. You can basically think of this as a replacement for your old `Config+Setup.swift` file (although the way you setup is now different).
- `routes.swift`: This is where your "main" route collection is and this where you would add individual routes or other route collections.

### Configuration

Let's go ahead and try and configure something. For this project, we'll use MySQL as this seems to be very common in Vapor projects. Let's add the MySQL package to our `Package.swift` file:

```swift
.package(url: "https://github.com/vapor/mysql-driver.git", .branch("beta")),
```

Don't forget to add `FluentMySQL` to the list of dependencies for the `App` target:

```Swift
.target(name: "App", dependencies: ["Vapor", "FluentMySQL"]),
```

It's worth noticing that we're not adding a `mysql-provider` package. It's a bit unclear to us at this point, but it seems like the naming conventions (or Vapor concepts) are changing in this area. Looking at the docs, there is also something called ["Services"](https://docs.vapor.codes/3.0/concepts/services/) now. Our applications basically has a list of services which is functionality that can be used throughout our project. It seems that you _register_ providers to your services container and you _use_ configurations on your services container.

Setting up MySQL for our project looks like this:

```Swift
import Vapor
import FluentMySQL

public func configure(
    _ config: inout Config,
    _ env: inout Environment,
    _ services: inout Services
) throws {
    // Providers
    try services.register(FluentProvider())

    // Fluent MySQL config
    services.use(FluentMySQLConfig())

    // Database config
    var dbConfig = DatabaseConfig()
    let database = MySQLDatabase(
        hostname: "localhost",
        user: "root",
        password: nil,
        database: "vapor-3-test"
    )
    dbConfig.add(
        database: database,
        as: .mysql
    )
    services.use(dbConfig)
}

extension DatabaseIdentifier {
    static var mysql: DatabaseIdentifier<MySQLDatabase> {
        return .init("mysql")
    }
}
```

Let's try and go through the different steps that are happening in the above code:

- First we register the `FluentProvider` since we want to use Fluent.
- Next we use a config file for MySQL since we want to use MySQL for this project. This config holds general MySQL configurations such as if MySQL should only use SSL.
- Last but not least, we create a database config for our MySQL database and use it. Note how we made a unique identifier at the bottom for our database. This indicates that we could have multiple db connections open, but we're not going to use that.

It takes a little to grasp the notion of "Services" and how you now register providers and use configurations. We are excited to see that there are **no more json configs**.

### Models

Vapor 2 has been pretty boilerplate-heavy when it comes to models. This is something we have tried to solve using Soucery as described in [this blog post](https://engineering.nodesagency.com/articles/Vapor/vapor-code-generation-with-sourcery/). We've been pretty interested in seeing how this would change with Vapor 3 since the framework is now able to leverage on the [`Codeable`](https://developer.apple.com/documentation/swift/codable) features of Swift 4.

Let's start out by creating a simple model conforming to `Codeable`:

```Swift
import Foundation
import FluentMySQL

final class Post: Codable {
    var id: Int?
    let title: String
    let body: String

    init(id: Int? = nil, title: String, body: String) {
        self.title = title
        self.body = body
    }
}
```

At this point, there's nothing Vapor specific other than we already went ahead and imported the `FluentMySQL` module. It's great to see how we are able to keep our raw models independent of the framework and instead use extensions to add the Vapor specific functionality. Let's move on and see what is required by Fluent to make our `Post` a `Model`:

```swift
extension Post: Model {
    typealias Database = MySQLDatabase
    typealias ID = Int
    static let idKey = \Post.id
}
```

The only steps we need to take in order to make our `Post` become a `Model` is to define the database we're using for this model, define how our models are uniquely identified and define what the keypath to our id field is.

#### Migrations

```swift
extension Post: Migration {}
```

That's pretty much it. Using `Codable` Vapor is now able to convert your Swift types into database field types. To make sure your migration is being run you'll have to use a migration configuration in your `configure.swift`:

```swift
var migrationConfig = MigrationConfig()
migrationConfig.add(model: Post.self, database: .mysql)
services.use(migrationConfig)
```

This will obviously not be a solution to all cases, and most of the time you will probably have specific requirements for your db fields. One example could be to lower the length of your VARCHAR since a `String` in Swift will be turned into a 255 characters long VARCHAR, which might not be what you want if you want [to support emojis](https://github.com/nodes-vapor/readme/blob/master/Documentation/how-to-support-emojis.md). To handle your migration manually, you can instead specify the `prepare` function yourself instead of using the default behaviour:

```swift
extension Post: Migration {
    static func prepare(on connection: MySQLConnection) -> Future<Void> {
       return MySQLDatabase.create(self, on: connection) { builder in
          try builder.field(for: \.id)
          try builder.field(for: \.title)
          try builder.field(type: .varChar(length: 191), for: \.body)
        }
    }
}
```

### Routing

Let's go through some of common use cases for dealing with a `Model`:

- Retrieving all instances of a model.
- Retriving one specific instance of a model using a unique identifier.
- Creating a new instance of a model.
- Updating an instance of a model.
- Deleting an instance of a model.

Before setting up the functionality for these different cases, we need to do two things first. First let's make our database available through our request (this could e.g. be added to our `configure.swift`):

```swift
extension Request: DatabaseConnectable {}
```

This will make it a bit easier to interact with the database by using the request as we will see soon.

Next, we want to make our `Post` model conform to `Parameter` so that it can be used as a type-safe parameter for our routes. Go ahead and add this to `Post.swift`:

```swift
extension Post: Parameter {}
```

With that in place, we can now go through the different cases for interacting with our model type. Before we add routes for each case, let's go ahead and a `RouteGroup` to our `routes.swift` which is a `RouteCollection` where we are supposed to register our routes:

```swift
func boot(router: Router) throws {
    // Posts
    RouteGroup(cascadingTo: router).group("posts") { group in
        let postController = PostController()
    }
}
```

And for now, let's just create an empty controller:

```swift
import Vapor

final class PostController {
  
}
```

#### Retrieving all instances

To return all posts created (e.g. by requesting `GET /posts`), we're going to query them like this:

```swift
func all(req: Request) throws -> Future<[Post]> {
    return Post.query(on: req).all()
}
```

There's a couple of things to notice in the above snippet when we compare it to Vapor 2. 

First is that we're not specifying what connection to perform the query on. In this case, we're able to use the request itself because we made `Request` conform to `DatabaseConnectable`. This makes it very explicit for who is responsible for performing the lookup. Without having tried it yet, this should help us make tests as well since we should potentially be able mock the database connection and instead of hitting a database, we could return in-memory objects created in our tests.

Next, there's the return type which is now `Future<[Post]>` instead of `[Post]` as you might have expected coming from Vapor 2. [Async, Streams, Futures and Reactive Programming](https://docs.vapor.codes/3.0/async/getting-started/) are central topics in Vapor 3 in order to increase the performance of the framework and it will change the way we work. Without going too much into details, one would of thinking of the concepts could be:

- Futures: Values that at some point in time will exist. The future part is a wrapper that allows us to continue the work we want to do when the value arrives.
- Streams: If a function returns one value when it's done executing, a stream returns an endless number of values. Think of it as generally always running which might return values once in a while.
- Reactive Programming: Related to futures and streams are reactive programming. It's a pattern for dealing with these types of data, usually using functional programming. You can think of it as a way of transforming or dealing with data when it comes in through a future or a stream.

If you've worked with any of the reactive programming frameworks, these concepts introduced in Vapor 3 should be familiar.

#### Retriving one specific instance

To return one specific post (e.g. by requesting `GET /posts/:id`), we're going to request that instance like this:

```Swift
func specific(req: Request) throws -> Future<Post> {
    return try req.parameter(Post.self)
}
```

This is very similar to what we've used to in Vapor 2 with the exception that fetching a parameter from the request now returns a `Future`.

#### Creating a new instance

To create a new post (e.g. by requesting `POST /posts`), we're going to do it like this:

```swift
func create(req: Request) throws -> Future<Post> {
    let post = try req.content.decode(Post.self)
    return post.save(on: req).transform(to: post)
}
```



A couple of things has changed here. In Vapor 2 you might have used `req.data`, `req.form` or `req.json`  for retrieving the body of a request, but in Vapor 3 this is now contained in a `ContentContainer` on the request. Next we can use `decode` to transform the body into the expected type using the `Decodable` protocol from Swift 4. Remember that our `Post` model is conforming to `Codable` in order for this to work.

Since calling `save` won't give us a back a `Post`, we will "transform" the result of calling `save` into the post we just saved. Note that `transform` is a helper coming from Vapor which simply uses `map` under the hood.

#### Updating an instance

To update an existing post (e.g. by requesting `PATCH /posts/:id`), we're going to do it like this:

```Swift
func update(req: Request) throws -> Future<Post> {
    let post = try req.parameter(Post.self)
    let content = try req.content.decode(Post.self)

    return post.flatMap(to: Post.self) { post in
        post.title = content.title
        post.body = content.body

        return post.save(on: req).transform(to: post)
    }
}
```

You could choose to do something similar to the code snippet for creating an instance, since Vapor will update the instance if the payload contains an `id` field. For this example, we've chosen to make it more explicit what is going on in the update process. In the above code, we're fetching the post that was specified in the request, we're decoding the post model that was giving in the body of the request and last we're updating the currently saved post. Note how we're using `flatMap` since we're creating a nested future inside of our closure.

#### Deleting an instance

To delete an existing post (e.g. by requesting `PATCH /posts/:id`), we're going to do it like this:

``` Swift
func delete(req: Request) throws -> Future<HTTPResponse> {
    let post = try req.parameter(Post.self)
    return post.flatMap(to: HTTPResponse.self) { post in
        return post.delete(on: req).transform(to: HTTPResponse(status: .ok))
    }
}
```

The above code is very similar to how we deal with updating a post, with the difference being that instead of returning the updated post, we're 

### Views

TODO

### Commands

TODO

## Migrating from Vapor 2 to Vapor 3

TODO - test out project or package

## Conclusion

TODO
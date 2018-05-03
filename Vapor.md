# Vapor Learning Notes
1. [Getting started](#getting-started)
2. [Model example](#model-example)
3. [Controller example](#controller-example)
4. [Parent-children relationship](#parent-children)
5. [Sibling relationship](#siblings)
6. [Useful links](#links)

## <a id="getting-started"></a> Getting started

Check if ready for Vapor `eval "$(curl -sL check.vapor.sh)"`

Install brew `brew install vapor/tap/vapor`

Check for latest version `brew upgrade vapor/tap/vapor`

---

Make a new project `vapor new <Project Name> --branch=beta`

-> cd into project

Generate xCode Project `vapor xcode`

-> Open xCode project

-> Run -> My Mac

-> Go to [http://localhost:8080/hello](http://localhost:8080/hello)

 
To create a new file `touch Sources/App/Models/<File Name>.swift`

-> Regenerate xcode project `vapor xcode -y`

## <a id="model-example"></a>Model Example  

```swift
import Foundation
import FluentSQLite
import Vapor

final class User: Codable {
    var id: UUID?
    var name: String
    var username: String
    
    init(name: String, username: String) {
        self.name = name
        self.username = username
    }
}

extension User: SQLiteUUIDModel {}
extension User: Content {}
extension User: Migration {}
```
 
Add it to migrations in **configure.swift**
```swift
migrations.add(model: User.self, database: .sqlite)
```

## <a id="controller-example"></a>Controller Example

  
```swift
import Vapor

struct UsersController: RouteCollection {
    func boot(router: Router) throws {
        let usersRoute = router.grouped("api", "users")
        usersRoute.get(use: getAllHandler)
        usersRoute.post(use: createHandler)
        usersRoute.get(User.parameter, use: getHandler)
    }
    
    func getAllHandler(_ req: Request) throws -> Future<[User]> {
        return User.query(on: req).all()
    }
    
    func createHandler(_ req: Request) throws -> Future<User> {
        let user = try req.content.decode(User.self)
        return user.save(on: req)
    }
    
    func getHandler(_ req: Request) throws -> Future<User> {
        return try req.parameter(User.self)
    }
}

extension User: Parameter{}
```

Register it in **routes.swift**
```swift
let usersController = UsersController()
try router.register(collection: usersController)
```

## <a id="parent-children"></a>Parent - Children (One-to-many) Relationships Example

User (**Parent**) - creator of blog post, can write multiple blog posts

Blog post (**Children**) - created by one and only one user

In the **BlogPost** model
1. Add a creator id on the model
```swift
var creatorID: User.ID
```

2. Add creatorID in the initialiser as well

3. To be able to view the relationship, add an extension on the BlogPost
```swift
extension BlogPost {
    var creator: Parent<BlogPost, User> {
        return parent(\.creatorID)
    }
}
```

In the **User** model
1. Add an extension to be able to see all the blog posts of an user
```swift
extension User {
    var blogPosts: Children<User, BlogPost> {
        return children(\.creatorID)
    }
}
```

In the **BlogPostsController**
1. Create a getCreatorHandler that will return us the user that created the post
```swift
    func getCreatorHandler(_ req: Request) throws -> Future<User> {
        return try req.parameter(BlogPost.self).flatMap(to: User.self) { blogPost in
            return try blogPost.creator.get(on: req)
        }
    }
```

2. Register the route in the boot function
```swift
blogPostsRoute.get(BlogPost.parameter, "creator", use: getCreatorHandler)
```
In the **UsersController**
1. Create a new root handler getBlogPostsHandler to see all blog posts written by a user
```swift
    func getBlogPostsHandler(_ req: Request) throws -> Future<[BlogPost]> {
        return try req.parameter(User.self).flatMap(to: [BlogPost].self) { user in
            return try user.blogPosts.query(on: req).all()
        }
    }
```

2. Register the route in the boot function
```swift
usersRoute.get(User.parameter, use: getBlogPostsHandler)
```


## <a id="siblings"></a>Sibling Relationships (Many-to-Many) Example
Category (**Sibling**) - can have many blog posts
Blog Post (**Sibling**) - can have many categories

To represent many-to-many relationships in Vapor we need to use **Pivots**.

1. Create a new model **BlogPostCategoryPivot**
```swift
import Foundation
import FluentSQLite
import Vapor

final class BlogPostCategoryPivot: SQLiteUUIDPivot {
    var id: UUID?
    var blogPostID: BlogPost.ID
    var categoryID: Category.ID
    
    typealias Left = BlogPost
    typealias Right = Category
    
    static let leftIDKey: LeftIDKey = \BlogPostCategoryPivot.blogPostID
    static let rightIDKey: RightIDKey = \BlogPostCategoryPivot.categoryID
    
    init(_ blogPostID: BlogPost.ID,_ categoryID: Category.ID) {
        self.blogPostID = blogPostID
        self.categoryID = categoryID
    }
}
```

2. In **configure.swift** add
```swift
migrations.add(model: BlogPostCategoryPivot.self, database: .sqlite)
```

3. In **BlogPost** add a computed property
```swift
var categories: Siblings<BlogPost, Category, BlogPostCategoryPivot> {
    return siblings()
}
```

4. In **Category** add a computer property
```swift
var blogPosts: Siblings<Category, BlogPost, BlogPostCategoryPivot> {
    return siblings()
}
```

5. Add the route in the **BlogPostsController**
```swift
func getCategoriesHandler(_ req: Request) throws -> Future<[Category]> {
    return try req.parameter(BlogPost.self).flatMap(to: [Category].self) { blogPost in
        return try blogPost.categories.query(on: req).all()
    }
}
```

And register it in the boot
```swift
blogPostsRoute.get(BlogPost.parameter, "categories", use: getCategoriesHandler)
```

6. Add the route in **CategoriesController**
```swift
func getBlogPostsHandler(_ req: Request) throws -> Future<[BlogPost]> {
    return try req.parameter(Category.self).flatMap(to: [BlogPost].self) { category in
        return try category.blogPosts.query(on: req).all()
    }
}
```

And register it in the boot
```swift
categoriesRoute.get(Category.parameter, "blogPosts", use: getBlogPostsHandler)
```

7. Add a new POST route in the **BlogPostsController**
```swift
    func addCategoriesHandler(_ req: Request) throws -> Future<HTTPStatus> {
        return try flatMap(to: HTTPStatus.self, req.parameter(BlogPost.self),
                           req.parameter(Category.self)) { blogPost, category in
                            let pivot = try BlogPostCategoryPivot(blogPost.requireID(), category.requireID())
                            return pivot.save(on: req).transform(to: .ok)
        }
    }
```

And register it in the boot
```swift
blogPostsRoute.post(BlogPost.parameter, "categories", Category.parameter, use: addCategoriesHandler)
```

## <a id="links"></a>Useful links
* [Vapor documentation](https://docs.vapor.codes/3.0/)
* [Vapor Slack](https://vapor.team/)
* [RayWenderlich Server Side Swift with Vapor Course](https://videos.raywenderlich.com/courses/115-server-side-swift-with-vapor)
* [RESTed app](https://itunes.apple.com/us/app/rested-simple-http-requests/id421879749?mt=12) - to format and make HTTP requests and view the response



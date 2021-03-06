# Server Side Swift

Server Side __Swift__ code snippets using __Vapor__. 

![icon](img/vapor.png)


## Setup your Development Environment

On macOS, download Xcode 8.  We will use Swift 3, the latest Swift Package Manager, and the IDE that comes with Xcode.

Open Terminal and verfiy the Xcode install with:

````
curl -sL check.vapor.sh | bash
````

We will use the Vapor Command Line Interface called __Toolbox__ to create, manage, and deploy our Swift Server projects that are powered by Vapor.  

In Terminal, install the Vapor Toolbox with:

````
curl -sL toolbox.vapor.sh | bash
````

Confirm that the install of Vapor Toolbox with:

````
vapor --help
````

Then tell the Vapor Toolbox it should update itself in the future with:

````
vapor self update
````

## Create a New Vapor Project

In Terminal, use Toolbox to create a new Vapor Project called 'swiftserver' with:

````
cd ~/Desktop
vapor new swiftserver
````

Observe that the Vapor Toolbox scaffolds a Project folder structure for us (see below):

![icon](img/scaffolding.png)

Now, let's create an Xcode project so that we can view and maintain our project files using the Xcode IDE.  

In Terminal, go into the root of your project directory and create an Xcode project with:

````
cd swiftserver
vapor xcode
````

When prompted to open the Xcode project go ahead and do so.  

Review the directory structure of the Project - explained here in the [Vapor Docs](https://vapor.github.io/documentation/guide/folder-structure.html). Examine the following files in particular:

* /Package.swift
* /Sources/App/main.swift

Before moving forward, run the Vapor project to make sure everything works.  In Xcode, choose the *App* target and Run (see below):

![icon](img/runfirst.png)

Open a web browser and navigate to http://localhost:8080 . You should see it works ...

![icon](img/works.png)

## Routing Basics

In this section, we're going to learn how to author simple GET and POST requests using Vapor routing.  In the next section, we'll use a database to persist and fetch this same data instead of using static JSON. 

In Xcode, navigate to /Sources/App/Main.swift and edit it as:

````
import Vapor

let drop = Droplet()

drop.get { request in
    
    return try drop.view.make("welcome", [
        "message": drop.localization[request.lang, "welcome", "title"]
        ])
}

// handles GET /welcome
drop.get("welcome") { request in
    
    return "sup"
}

// handles GET /brewpub/list
drop.get("brewpub/list") { request in
    
    var brewpub1 = JSON([
        "id": 1,
        "name": "Peddler Brewing Company",
        "lat": 47.663870,
        "lng": -122.377095
        ])
    var brewpub2 = JSON([
        "id": 2,
        "name": "Hale's Ales Brewery & Pub",
        "lat": 47.659067,
        "lng": -122.365253
        ])
    var brewpub3 = JSON([
        "id": 3,
        "name": "Urban Family Brewing",
        "lat": 47.660615,
        "lng": -122.39029
        ])
    var brewpub4 = JSON([
        "id": 4,
        "name": "Ballard Beer Company",
        "lat": 47.668843,
        "lng": -122.38427
        ])
    
    var jsonResponse = JSON([:])
    jsonResponse["results"] = JSON([brewpub1, brewpub2, brewpub3, brewpub4])
    
    return try JSON(node: jsonResponse)
}

// handles GET /brewpub/id (e.g. /brewpub/1)
drop.get("brewpub", Int.self) { request, id in
    
    return "Fetching brewpub with id \(id)"
}

// handles POST /brewpub
drop.post("brewpub") { request in
    
    guard let name = request.data["name"]?.string else {
        throw Abort.badRequest
    }
    guard let lat = request.data["lat"]?.double else {
        throw Abort.badRequest
    }
    guard let lng = request.data["lng"]?.double else {
        throw Abort.badRequest
    }
    
    return "We got your POST with name = \(name), latitude = \(lat), and longitude = \(lng)"
}

drop.resource("posts", PostController())

drop.run()
````
In Xcode, run the App.  

Using your favorite HTTP client (e.g. web browser, [RESTed](https://itunes.apple.com/us/app/rested-simple-http-requests/id421879749?mt=12), [Postman](https://www.getpostman.com)), view the responses from the different routes. 

When we GET request __http://localhost:8080/brewpub/list__ we see the JSON response of brewpubs (see below):
![icon](img/basics_list.png)

When we GET request __http://localhost:8080/brewpub/1__ we see a message that processes the input Id (i.e. integer of 1).  Note, in a future section we will use this Id to look up a brewpub from the database and return that brewpub's data in a JSON response.
![icon](img/basics_getint.png)

When we POST request __http://localhost:8080/brewpub__ we see a message that processes the passed in parameters.  Note, in a future section we will use the input parameter data to create a new brew pub record in the database and return a status message (e.g. success) in the response.
![icon](img/basics_post.png)

At this point, play with Routing with Vapor.  Here are some relevant sections of the docs to get started:

* [Routing Basics](https://vapor.github.io/documentation/routing/basic.html)
* [Route Parameters](https://vapor.github.io/documentation/routing/parameters.html)
* [Query Parameters](https://vapor.github.io/documentation/routing/query-parameters.html)

## Create an API

In this section we are going to create a RESTful web service that will allow client Apps to fetch a collection of Brew Pubs.  We will also allow our client Apps to create new Brew Pubs and update existing ones.  The data will be persisted in a PostgreSQL database.  We will debug/develop locally.  In the final section we will deploy this to a Cloud service (e.g. Heroku).

Being a RESTful web service, we will be able to consume this data in multiple platforms such as Apple (iOS, watchOS, tvOS, macOS), Android apps, and Web apps.

#### Step 1: Install PostgreSQL on macOS

We will install PostgreSQL on our Mac using Homebrew.  

If you *don't* have Homebrew, use Terminal to install [Homebrew](https://brew.sh) with:

````
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
````

If you *do* have Homebrew installed on your Mac, then its a good practice to always update before installing anything with:

````
brew update
````

Install PostgreSQL with:

````
brew install postgresql
````

Create a new database cluster with:

````
initdb /usr/local/var/postgres -E utf8
````

Start Postgresql with:

````
brew services start postgresql
````

If you prefer to work with PostgreSQL using a database GUI, then install [PgAdmin](https://www.pgadmin.org/download/macos4.php).  This quick-start will use PgAdmin.

Open pgAdmin, create a server and connect with:

* host: localhost 
* user name: [your macOS username]

Create a __beer__ database. Your pgAdmin UI should resemble the following:

![icon](img/createdb.png)


#### Step 2: Configure the Vapor Project to Connect to your PostgreSQL Database

In Xcode, edit the __Package.swift__ file to add the PostgreSQL provider for Vapor as a dependency.  See below:

````
import PackageDescription

let package = Package(
    name: "swiftserver",
    dependencies: [
        .Package(url: "https://github.com/vapor/vapor.git", majorVersion: 1, minor: 5),
        .Package(url: "https://github.com/vapor/postgresql-provider", majorVersion: 1, minor: 0)
    ],
    exclude: [
        "Config",
        "Database",
        "Localization",
        "Public",
        "Resources",
    ]
)

````

After saving changes to the Package.swift file, have Vapor re-build and re-create the Xcode project.  This will download the new dependencies.

````
vapor xcode
````

In Xcode, edit the /Sources/App/main.swift file to:

* Add an import VaporPostgreSQL statement.
* Add a provider to the Droplet instance.
* Create a new route that will test connecting to your local development database and responding with the PostgreSQL version.  See below:

````
import Vapor
import VaporPostgreSQL

let drop = Droplet()
do {
    try drop.addProvider(VaporPostgreSQL.Provider.self)
} catch {
    assertionFailure("Error adding provider: \(error)")
}

drop.get("version") { request in
    
    if let db = drop.database?.driver as? PostgreSQLDriver {
        //let version = try db.raw("SELECT version()")
        let version = try db.raw("select current_database();")
        return try JSON(node: version)
    }
    else {
        return "No db connection"
    }
}

drop.run()
...

````

In your Xcode project, create a __secrets__ folder under the /Config folder.

In the secrets folder, add the following configuration file so that the app can connect to your local PostgreSQL database.  See below screenshot.  Note, change your user and password to your system/macOS credentials.

![icon](img/postgresconfig.png) 

Run Xcode, use a HTTP client (e.g. RESTed) to confirm the Vapor Project can connect to your PostgreSQL database.

![icon](img/dbversionroute.png)

#### Step 3: Create a BrewPub Model

Create a new BrewPub.swift file and add it to /Sources/App/Models.

Implement the BrewPub model.  Note that this model class must conform to the Model, NodeInitializable, NodeRepresentable, and Preparation protocols. See below:

````
import Foundation
import Vapor
import Fluent

final class Brewpub: Model {
    
    // required by Model protocol
    var id: Node?
    var exists: Bool = false
    
    var name: String
    var latitude: Double
    var longitude: Double
    
    // convenience initializer
    init(name: String, latitude: Double, longitude: Double) {
        self.id = nil
        self.name = name
        self.latitude = latitude
        self.longitude = longitude
    }
    
    // required by NodeInitializable protocol
    init(node: Node, in context: Context) throws {
        
        id = try node.extract("id")
        name = try node.extract("name")
        latitude = try node.extract("latitude")
        longitude = try node.extract("longitude")
    }
    
    // required by NodeRepresentable protocol
    func makeNode(context: Context) throws -> Node {
        
        return try Node(node: [
            "id": id,
            "name": name,
            "latitude": latitude,
            "longitude": longitude
        ])
    }
    
    // for creating the database table for this model
    static func prepare(_ database: Database) throws {
        try database.create("brewpubs") { pubs in
            pubs.id()
            pubs.string("name")
            pubs.double("latitude")
            pubs.double("longitude")
        }
    }
    
    // for deleting the database table
    static func revert(_ database: Database) throws {
        try database.delete("brewpubs")
    }
}
````
After implementing a model class, update the Droplet instance preparatiosn collection.  In /Sources/App/main.swift update as:

````
import Vapor
import VaporPostgreSQL

let drop = Droplet()
drop.preparations.append(Brewpub.self) // ADDED
do {
    try drop.addProvider(VaporPostgreSQL.Provider.self)
} catch {
    assertionFailure("Error adding provider: \(error)")
}

````


#### Step 4: Create the API Routes

````
import Vapor
import VaporPostgreSQL

let drop = Droplet()
drop.preparations.append(Brewpub.self) // ADDED
do {
    try drop.addProvider(VaporPostgreSQL.Provider.self)
} catch {
    assertionFailure("Error adding provider: \(error)")
}

/**
  handles POST /brewpub
  Allows users to create a new brewpub
 */
drop.post("brewpub") { request in
    
    // fetch parameters sent with POST
    guard let name = request.data["name"]?.string else {
        throw Abort.badRequest
    }
    guard let lat = request.data["lat"]?.double else {
        throw Abort.badRequest
    }
    guard let lng = request.data["lng"]?.double else {
        throw Abort.badRequest
    }
    
    // create model instance
    var brewpub = Brewpub(name: name, latitude: lat, longitude: lng)
    
    // save model to database
    try brewpub.save()
    
    return try JSON(node: brewpub.makeNode())
}

/**
  handles GET /brewpub/list
  Allows users to fetch a collection of brewpub resources
 */
drop.get("brewpub/list") { request in
    
    return try JSON(node: Brewpub.all().makeNode())
}

/**
  handles GET /brewpub/id (int)
  Allows clients to fetch a brewpub by identifier (e.g. /brewpub/1)
 */
drop.get("brewpub", Int.self) { request, id in
    
    return try JSON(node: Brewpub.query().filter("id", id).all().makeNode())
}

drop.run()
````

Note, as you iterate your data model and routes you can use:

````
vapor build --clean

vapor run prepare --revert
````

When your routes are complete, use your HTTP client (e.g. RESTed) to test and use pgAdmin to confirm rows are being inserted into your database. 

*In Progress - Add Unit Tests for routes*

## Enable CORS

This is an optional task if you'd like to allow access to your API from __web__ apps that are deployed to a web server that is different from your API.

Add the [CORS Middleware for Vapor](https://github.com/jhonny-me/CorsMiddleware) as a dependency:

````
import PackageDescription

let package = Package(
    name: "swiftserver",
    dependencies: [
        .Package(url: "https://github.com/vapor/vapor.git", majorVersion: 1, minor: 5),
        .Package(url: "https://github.com/vapor/postgresql-provider", majorVersion: 1, minor: 0),
        .Package(url: "https://github.com/jhonny-me/CorsMiddleware.git", majorVersion: 1, minor: 0)
    ], ...
````

Add the middleware to your Droplet instance:

````
let drop = Droplet(availableMiddleware: ["cors" :CorsMiddleware()])
````

Update In Config/droplet.json, add "cors" to the middleware server collection:

````
{
    "server": "engine",
    "client": "engine",
    "console": "terminal",
    "log": "console",
    "hash": "crypto",
    "cipher": "crypto",
    "middleware": {
        "server": [
            "file",
            "abort",
            "validation",
            "type-safe",
            "date",
            "sessions",
            "cors"
        ],
        "client": [
        
        ]
    }
}
````

Because we updated the Package.swift file, re-create your Xcode project.

````
vapor xcode
````

## Deploy to a Cloud Service

You can deploy your Vapor-powered Swift Server to many cloud services.  In this section we use Heroku.

#### Step 1: Sign Up for [Free Heroku Account here](https://www.heroku.com)

#### Step 2: Install the Heroku Command Line Interface (CLI)

Now, install the [Heroku Command Line Interface (CLI)](https://devcenter.heroku.com/articles/heroku-cli) with:

````
brew update
brew install heroku
````
Then, confirm the Heroku CLI installation with:

````
heroku --version
````

#### Step 3: Deploy to Heroku

If your Vapor project is not yet under source control, create a local git repository for it using the following in Terminal:

````
git init
git add -A
git commit -m 'a message about your commit'
````

In Terminal, login to Heroku using the Heroku CLI with the following:  

````
heroku login
````

When prompted, enter your Heroku account credentials.

Next, deploy your Vapor app to Heroku with the following:

````
vapor heroku init
````

You will be prompted with several question.  When asked the final question *Would you like to push to Heroku now?* answer __n__ for NO.

Then, instruct Heroku to create a free (hobby) development database with:

````
heroku addons:create heroku-postgresql:hobby-dev
````

Then fetch the __database url__ for the created database with:

````
heroku config
````

The Database URL will be presented and will resemble:

````
DATABASE_URL: postgres://zrgyqqyqvwrrgi:e1.compute-1.amazonaws.com:543if
````


Edit the Procfile located in your project's root directory with:

`````
nano Procfile

`````

Add the Heroku database url variable and save the changes to the Procfile.  Your Procfile should look like the following:

````
web: App --env=production --workdir="./"
web: App --env=production --workdir=./ --config:servers.default.port=$PORT --config:postgresql.url=$DATABASE_URL

````

Upload your changes back up to Heroku do the following:

````
git commit -A
git commit -m 'a message about your commit'
git push heroku master
````

That's all there is to it!  

## Connect

* Twitter: [@clintcabanero](http://twitter.com/clintcabanero)
* GitHub: [ccabanero](http:///github.com/ccabanero)

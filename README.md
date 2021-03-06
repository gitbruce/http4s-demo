# http4s-demo
A quick demo of using http4s to build a REST service.

First things - in order to use http4s, you will need to add the following dependencies to your `build.sbt` file:

```scala
libraryDependencies ++= Seq(
  "org.scalaz" %% "scalaz-core" % ScalazVersion, // for type awesomeness
  "org.scalaz" %% "scalaz-concurrent" % ScalazVersion, // for type awesomeness
  "io.argonaut" %% "argonaut" % "6.1", // json (de)serialization scalaz style
  "org.http4s" %% "http4s-dsl" % Http4sVersion,
  "org.http4s" %% "http4s-blaze-core" % Http4sVersion,
  "org.http4s" %% "http4s-blaze-server" % Http4sVersion,
  "org.http4s" %% "http4s-argonaut" % Http4sVersion
)
```

It might be possible to use a JSON library other than `argonaut`, but why fight it? `http4s` has `argonaut` support built in (and uses `jawn` on top of that), so just go with it.

`http4s` can be used with different containers, including `jetty` and `tomcat`, but it comes with a simple server called `blaze`. We just use that for hosting the API.

## Example domain model

For this sample, suppose we want to build APIs to track musicians, bands, and albums. It's a pretty simple data model - each musician can belong to zero or more bands, each band can have zero or more musicians as members, and a band can release zero or more albums. Each album is released by exactly one band.

```scala
case class Musician(
  id: Int,                  // unique primary key for record
  firstName: String,
  lastName: String,
  instrument: List[String]
)

case class Band(
  id: Int,                  // unique primary key for record
  name: String,
  member: List[Int]         // list of the ids of the musicians that belong to the band
)

case class Album(
  id: Int,                  // unique primary key for record
  name: String,
  band: Int,                // ID of the band that released the album
  releaseYear: Int
)
```

## A REST server for this domain model

### Setting up routes

Given that domain model, the following is a list of REST APIs that you might provide (ignoring the PATCH operation for the time being):

```
GET <url>/api/musicians                  Fetch a list of records for musicians
GET <url>/api/musicians/{musicianID}     Fetch the record for the specific musician
POST <url>/api/musicians                 Create a new musician record
PUT <url>/api/musicians/{musicianID}     Replace an existing musician record
DELETE <url>/api/musicians/{musicianID}  Remove an existing record for a musician

GET <url>/api/bands                      Fetch a list of records for bands
GET <url>/api/bands/{bandID}             Fetch the record for the specific band
POST <url>/api/bands                     Create a new band record
PUT <url>/api/bands/{bandID}             Replace an existing band record
DELETE <url>/api/bands/{bandID}          Remove an existing record for a band

GET <url>/api/albums                     Fetch a list of records for albums
GET <url>/api/albums/{albumID}           Fetch the record for the specific album
POST <url>/api/albums                    Create a new album record
PUT <url>/api/albums/{albumID}           Replace an existing album record
DELETE <url>/api/albums/{albumID}        Remove an existing record for a album
```

In `http4s`, you specify these _routes_ using the Domain Specific Language (DSL) provided with the package. Define an _HttpService_, which is just a `Service[Request, Response]`, which in turn is just a `Kleisli[Task, Request, Response]`. Explaining `Kleisli` is a little beyond my experience level currently. Suffice it to say that an `HttpService` is a transform that will convert an HTTP `Request` into a Scalaz `Task` wrapping an HTTP `Response`. Running the `Task` will generate the appropriate output from the REST server.

Luckily, you don't really need to understand much of that to get something up and running. For instance,

```scala
object MyService {
  val service(implicit executionContext: ExecutionContext = ExecutionContext.global): HttpService = Router(
    case req @ GET -> Root / "musicians" => fetchMusicianList(req)
    case req @ GET -> Root / "musicians" / musicianID => fetchMusician(req, musicianID)
    case req @ POST -> Root / "musicians" => createMusician(req)
    case req @ PUT -> Root / "musicians" / musicianID => replaceMusician(req, musicianID)
    case DELETE -> Root / "musicians" / musicianID => deleteMusician(musicianID)
    // Add routes for bands, albums
  )
}
```

Each of the functions that appears following an `=>` must return a `Task[Response]`. The DSL provides some assistance for creating these tasks - for instance, a couple of simple (but super lame) functions might be something like
```scala
  def fetchMusicianList(req: Request): Task[Response] = {
    Ok("Fetch musician list")
  }
  def DeleteMusician(musicianID: Int): Task[Response] = {
    BadRequest("I'm sorry, Dave, I'm afraid I can't do that.")
  }
```

The first will cause the HTTP server to return a response with status code 200 (OK) and the body "Fetch musician list", while the second will cause the server to return a response with status code 400 (Bad Request) and HAL's obstinant response in the body.

### Running the service

Hosting this service in an HTTP server is pretty simple - just

```scala
import org.http4s.server.blaze.BlazeBuilder

object MyApp {
  def main(args: Array[String]): Unit = {
    BlazeBuilder.bindHttp(8123)				// 8123 is the port for the HTTP server
      .mountService(MyService.myService, "/api")	// "api" is the suffix following the server/port # in URL
      .run
      .awaitShutdown()
  }
}
```

That suffix can just be "/". Make sure you put the "/" at the start, though - it doesn't assume one for you.

And there it is - when you run this executable, it will start an HTTP server running on port 8123 of `localhost` that will handle any requests that match the defined routes. For example, a GET to `http://localhost:8123/api/musicians` will yield the response generated by `fetchMusicianList`.

### JSON serialization / deserialization

Start by defining a JSON serializer and deserializer for each of your case classes. We put these into the companion object for the case class. Then define an `EntityEncoder` and `EntityDecoder` to go along with this.
```scala
object Musician {
  implicit val MusicianCodecJSON: CodecJson[Musician] =
    Argonaut.casecodec4(Musician.apply, Musician.unapply)(
      "id", "firstName", "lastName", "instrument"
    )
  implicit val MusicianEntityDecoder: EntityDecoder[Musician] = jsonOf[Musician]
  implicit val MusicianEntityEncoder: EntityEncoder[Musician] = jsonEncoderOf[Musician]
}
```

The first implicit is a standard `argonaut` codec (coder / decoder, or serializer / deserializer). The other two implicits allow that codec to be used in the `http4s` context.

One of the less awesome things about `argonaut` - the 4 in `Argonaut.casecodec4` refers to the number of fields in the JSON object that should be mapped into the case class. If you add another field to the case class, you need to update this to `casecodec5`. Luckily, if you forget to do that, the code won't compile. Likewise, the list of strings is the list of tag names in the JSON object. They correspond positionally to the fields in the case class, so if you refactor the case class and reorder fields, you might introduce some odd behavior that won't fail at compile-time.

Once you have these implicits in scope, you are able to do things like this:

```scala
  def createMusician(req: Request): Task[Response] = {
    req.decode[Musician] { m =>
      Ok(
        s"""Received request to create musician:
           |  id ${m.id}
           |  first name ${m.firstName}
           |  last name ${m.lastName}
           |  instruments ${m.instrument.mkString(", ")}""".stripMargin
      )
    }
  }

  def fetchMusician(req: Request, musicianID: Int): Task[Response] = {
    val m: Musician = Musician(musicianID, "Fred", "Flintstone", List("guitar", "keyboards"))
    Ok(m)
  }
```

That `Ok(m)` works because of the implicit `EntityEncoder` that was discussed earlier.

With that in place in the server, you could try something like
`curl -d '{"id": 1, "firstName": "Jim", "lastName": "Morrison", "instrument":["vocals"]}' 127.0.0.1:8123/api/musicians`

You should get output like
```
Received request to create musician:
  id 1
  first name Jim
  last name Morrison
  instruments vocals
```

## Turning down the code suck

### Breaking apart the service

If your REST API has a lot of routes, the service definition will get a little lengthy. Luckily, you can use Scala's partial functions and composition to break things up into more manageable pieces. For instance, we could define the routes that relate to musicians, to bands, and to albums as three separate pieces.


```scala
object MyService {
  type Routes = PartialFunction[Request, Task[Response]]

  def musicianRoutes: Routes = {
    case req @ GET -> Root / "musicians" => fetchMusicianList(req)
    case req @ GET -> Root / "musicians" / musicianID => fetchMusician(req, musicianID)
    case req @ POST -> Root / "musicians" => createMusician(req)
    case req @ PUT -> Root / "musicians" / musicianID => replaceMusician(req, musicianID)
    case DELETE -> Root / "musicians" / musicianID => deleteMusician(musicianID)
  }

  def bandRoutes: Routes = {
    case req @ GET -> Root / "bands" => fetchBandList(req)
    case req @ GET -> Root / "bands" / bandID => fetchBand(req, bandID)
    case req @ POST -> Root / "bands" => createBand(req)
    case req @ PUT -> Root / "bands" / mbandD => replaceBand(req, bandID)
    case DELETE -> Root / "bands" / bandID => deleteBand(bandID)
  }

  def albumRoutes: Routes = {
    case req @ GET -> Root / "albums" => fetchAlbumList(req)
    case req @ GET -> Root / "albums" / albumID => fetchAlbum(req, albumID)
    case req @ POST -> Root / "albums" => createAlbum(req)
    case req @ PUT -> Root / "albums" / malbumD => replaceAlbum(req, albumID)
    case DELETE -> Root / "albums" / albumID => deleteAlbum(albumID)
  }

  def service(implicit executionContext: ExecutionContext = ExecutionContext.global): HttpService = HttpService(
    musicianRoutes orElse bandRoutes orElse albumRoutes
  )
}
  
```

### Allowing injection of alternate implementations of the service

If you're putting together a REST API that's going to be used by an app someone else is also preparing, you might find yourself in the situation where you're getting the evil eye from the app developer who is waiting on you to get your API done. We did something like the following to allow you to create a "mock" service and a "production" service.

```scala
trait MyService {
  def musicianOps: MyMusicianOps
  def bandOps: MyBandOps
  def albumOps: MyAlbumOps

  def serviceComposer(implicit executionContext: ExecutionContext = ExecutionContext.global): HttpService =
    HttpService(
      userOps.routes orElse bandOps.routes orElse albumOps.routes
    )

  lazy val service: HttpService = serviceComposer
}

trait MyOps {
  def routes: MyService.Routes
}

object MyService {
  type Routes = PartialFunction[Request, Task[Response]]
}

object ProductionService extends MyService {
  def musicianOps: MyMusicianOps = ProductionMusicianOps
  def bandOps: MyBandOps = ProductionBandOps
  def albumOps: MyAlbumOps = ProductionAlbumOps
}

object MockService extends MyService {
  def musicianOps: MyMusicianOps = MockMusicianOps
  def bandOps: MyBandOps = MockBandOps
  def albumOps: MyAlbumOps = MockAlbumOps
}
```

Then in a file `MusicianOps.scala`, you might define

```scala
trait MyMusicianOps extends MyOps {
  def routes: MyService.Routes = {
    case req @ GET -> Root / "musicians" => fetchMusicianList(req)
    case req @ GET -> Root / "musicians" / musicianID => fetchMusician(req, musicianID)
    case req @ POST -> Root / "musicians" => createMusician(req)
    case req @ PUT -> Root / "musicians" / musicianID => replaceMusician(req, musicianID)
    case DELETE -> Root / "musicians" / musicianID => deleteMusician(musicianID)
  }

  def fetchMusicianList(req: Request): Task[Response]
  def fetchMusician(req: Request, musicianID: Int): Task[Response]
  def createMusician(req: Request): Task[Response]
  def replaceMusician(req: Request, musicianID: Int): Task[Response]
  def deleteMusician(musicianID: Int): Task[Response]
}

object ProductionMusicianOps extends MusicianOps {
  def fetchMusicianList(req: Request) : Task[Response] = ???

  def fetchMusician(req: Request, musicianID: Int): Task[Response] = ???

  def createMusician(req: Request): Task[Response] = ???

  def replaceMusician(req: Request, musicianID: Int): Task[Response] = ???

  def deleteMusician(musicianID: Int): Task[Response] = ???
}

object MockMusicianOps extends MusicianOps {
  def fetchMusicianList(req: Request) : Task[Response] = Ok("fetch musician list")

  def fetchMusician(req: Request, musicianID: Int): Task[Response] = {
    Ok(Musician(musicianID, "Jimi", "Hendrix", List("guitar"))
  }

  def createMusician(req: Request): Task[Response] = Ok("created musician")

  def replaceMusician(req: Request, musicianID: Int): Task[Response] = Ok("replaced musician")

  def deleteMusician(musicianID: Int): Task[Response] = Ok("deleted musician")
}
```

Not terribly useful mock implementations, but you get the idea. Do something like that for the band and album operations, and you're in business. In your `main` function, do something like

```scala
  def main(args: Array[String]): Unit = {

    val service: HttpService = if (args(1) == "-m") MockService.service else ProductionService.service

    BlazeBuilder.bindHttp(8123)
      .mountService(service, "api")
      .run
      .awaitShutdown()
  }
```

Note that in the `MyService` trait, we enumerate the different sets of operations that must be implemented, and the way that they are composed into the service. This way, we can be sure that the production and mock implementations will have the same set of routes. Likewise, note that we defined a `trait` that had the implementation of the routes in it, and only defined the implementations of the handler functions in the production and mock objects. This way, we can be sure that we have the identical routes for the production and mock implementations.

## Extra goodies

We threw in some extra stuff for code quality measures. Not that a demo needs them, but we thought people might be interested. Most of the "lint" and scalastyle setup is courtesy of Leif Wickland - check out https://github.com/leifwickland/static-analysis-skeleton.git for more details.

With that all in place, you can do `sbt lint:compile` and `sbt scalastyle` to see how much suck got added back to the code. One nice thing the linter does is to check for unused imports.

## Some next steps

- Different shapes of the create, get, and update objects
- PATCH to do incremental updates
- Versioning the API
- Providing HREFs/URLs instead of IDs for records
- Middleware to manipulate request - for instance, handle HTTP headers

## License

Feel free to put to good use - released to the public domain.

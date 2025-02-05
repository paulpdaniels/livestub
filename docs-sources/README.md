# livestub

[![Build Status](https://travis-ci.org/softwaremill/livestub.svg?branch=master)](https://travis-ci.org/softwaremill/livestub)
[![Docker Pulls](https://img.shields.io/docker/pulls/softwaremill/sttp.livestub.svg)](https://hub.docker.com/r/softwaremill/sttp.livestub/)
[![ImageLayers](https://images.microbadger.com/badges/image/softwaremill/sttp.livestub.svg)](https://microbadger.com/#/images/softwaremill/sttp.livestub)
[![Scala Steward badge](https://img.shields.io/badge/Scala_Steward-helping-brightgreen.svg?style=flat&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAQCAMAAAARSr4IAAAAVFBMVEUAAACHjojlOy5NWlrKzcYRKjGFjIbp293YycuLa3pYY2LSqql4f3pCUFTgSjNodYRmcXUsPD/NTTbjRS+2jomhgnzNc223cGvZS0HaSD0XLjbaSjElhIr+AAAAAXRSTlMAQObYZgAAAHlJREFUCNdNyosOwyAIhWHAQS1Vt7a77/3fcxxdmv0xwmckutAR1nkm4ggbyEcg/wWmlGLDAA3oL50xi6fk5ffZ3E2E3QfZDCcCN2YtbEWZt+Drc6u6rlqv7Uk0LdKqqr5rk2UCRXOk0vmQKGfc94nOJyQjouF9H/wCc9gECEYfONoAAAAASUVORK5CYII=)](https://scala-steward.org)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.softwaremill.sttp.livestub/livestub-app_2.13/badge.svg)](https://search.maven.org/search?q=g:com.softwaremill.sttp.livestub)


With livestub you can easly setup http server that behaves exactly as you would like it to. That makes livestub a perfect solution for faking arbitrary services which might be very useful in development/testing.

## Usage

### launch
 - **coursier**

    `coursier launch com.softwaremill.sttp.livestub:livestub-app_2.13:0.1.17 -- -p 7070`

- **docker**

    `docker run -p 7070:7070 softwaremill/sttp.livestub`

- **code**

```scala mdoc:invisible
import cats.effect._
import scala.concurrent._
import scala.concurrent.ExecutionContext.Implicits._
implicit val cs = IO.contextShift(ExecutionContext.global)
implicit val timer = IO.timer(ExecutionContext.global)
```
```scala mdoc:compile-only
    import sttp.livestub.app.LiveStubServer
    import sttp.livestub.app.LiveStubServer.Config
    LiveStubServer.resource(Config(port = 7070))
```

### stub
```bash
curl -X POST 'localhost:7070/__set' \
-d '{"when":{"method":"GET", "url":"animals/1/status"}, "then": {"statusCode":200, "body":{"status": "happy"} }}'
```

### invoke
```bash
curl 'localhost:7070/animals/1/status'
{"status":"happy"}
```

### swagger

Swagger is available under `/__admin/docs`

### advanced stubbing

Livestub supports wildcard http methods as well as wildcard path parameters.

wildcard method:
```bash
curl -X POST 'localhost:7070/__set' \
-d '{"when":{"method":"*", "url":"dogs"}, "then": {"statusCode":200, "body":{"status": "OK"} }}'
```

wildcard path param: 
```bash
curl -X POST 'localhost:7070/__set' \
-d '{"when":{"method":"GET", "url":"dogs/*/status"}, "then": {"statusCode":200, "body":{"status": "happy"} }}'
```

multiwildcard path param: (this one catches all routes which originate from `/dogs`)
```bash
curl -X POST 'localhost:7070/__set' \
-d '{"when":{"method":"GET", "url":"dogs/**"}, "then": {"statusCode":200, "body":{"status": "cheerful"} }}'
```

### additional methods

clear stubbed routes
```bash
curl -X POST 'localhost:7070/__clear'
```

show stubbed routes

```bash
curl 'localhost:7070/__routes'
```

set many responses which will be cycled through
```bash
curl -X POST 'localhost:7070/__set_many' \
-d '{"when":{"method":"GET", "url":"animals/1/status"}, "then": [
    {"statusCode":200, "body":{"status": "happy"} },
    {"statusCode":200, "body":{"status": "unhappy"} }
  ]
}'
```

### stubbing from code - sdk

```scala
libraryDependencies += "com.softwaremill.sttp.livestub" % "livestub-sdk" % "0.1.17"
```

Given a bunch of imports
```scala mdoc:silent
import sttp.client3.asynchttpclient.cats.AsyncHttpClientCatsBackend
import sttp.client3.SttpBackend
import sttp.livestub.sdk._
import sttp.livestub.api._
import sttp.client3.{Response => _, _}
import sttp.model._
import io.circe.Json
```

you can stub an arbitrary request:
```scala mdoc:compile-only
AsyncHttpClientCatsBackend[IO]().flatMap { implicit backend: SttpBackend[IO, Any] =>
  val livestub = new LiveStubSdk[IO](uri"http://mock:7070")
  livestub
    .when(RequestStubIn(MethodStub.Wildcard, "/user/*/status"))
    .thenRespond(Response.emptyBody(StatusCode.Ok, List(Header("X-App", "123"))))
}
```
stub given [sttp](https://github.com/softwaremill/sttp) request:
```scala mdoc:compile-only
AsyncHttpClientCatsBackend[IO]().flatMap { implicit backend: SttpBackend[IO, Any] =>
  val request = basicRequest
    .body(Map("name" -> "John", "surname" -> "doe"))
    .post(uri"https://httpbin.org/post?signup=123")

  val livestub = new LiveStubSdk[IO](uri"http://mock:7070")
  livestub.when(request).thenRespond(Response(Some(Json.fromString("OK")), StatusCode.Ok))
}
```
or stub given [tapir](https://github.com/softwaremill/tapir) endpoint:
```scala mdoc:compile-only
import sttp.tapir._

AsyncHttpClientCatsBackend[IO]().flatMap { implicit backend: SttpBackend[IO, Any] =>
  val myEndpoint = endpoint.get.in("/status").out(stringBody)

  val livestub = new LiveStubSdk[IO](uri"http://mock:7070")
  livestub.when(myEndpoint)
          .thenRespond(Response.emptyBody(StatusCode.Ok, List(Header("X-App", "123"))))
}
```

## OpenApi integration

OpenApi specification can be provided to bootstrap livestub server with auto generated endpoints.
For each given path a request-response stub will be generated matching particular url. Both path parameters and required query parameters will be
replaced by wildcard matches. That means that neither optional query parameters, request body nor headers are going to be checked when matching
stubbed endpoint.

Response structure will follow openapi schema. Response data will be created
based on openapi respective example parameters if provided, otherwise random data will be used(whenever it is possible).



For endpoints, which are defined in openapi specification but for which responses
couldn't be generated empty response body will be returned.

Usage:
`--openapi-spec <path_to_file>`

In addition, seed value for random data generator can be passed via:
`--seed 1234`
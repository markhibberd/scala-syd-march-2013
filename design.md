# Structuring Functional Programs

--

## A Motivating Example

--

## Some Primitives

    case class Config(baseUrl: String, https: Boolean, hmac: SecretKey)

    case class CaseInsensitiveString(s: String)

    case class Headers(v: Map[CaseInsensitiveString, Vector[String]])

    trait Audit
    case class AuditMacFailed(data: String) extends Audit
    case class AuditAccessDenied(path: String) extends Audit

--

## Packaging things up

    case class HttpRead(
      request: HttpRequest, reqheaders: Headers, config: Config
    )

    case class HttpState(
      resheaders: Headers, status: HttpStatus
    )

    type HttpWrite = Vector[Audit]

--

## Representing failure and early termination

    trait Result[A]
    case class Done[A](f: ChannelBuffer) extends Result[A]
    case class Cont[A](a: A) extends Result[A]
    case class Err[A](err: String \/ Throwable) extends Result[A]

    // Lots of other potential cases, model your control flow as data types!
    // case class Async[A](...) extends Result[A]

--

## Stacking types

    case class Http[A](run:
      ReaderT[
        WriterT[
          StateT[
            Result,
            HttpState,
            _
          ],
          HttpWrite,
          _
        ],
        HttpRead,
        A
     ]

--

## Stacking types (Actual Scala)

    case class Http[A](run:
      ReaderT[
        ({ type f[a] = WriterT[
          ({ type g[b] = StateT[
            Result,
            HttpState,
              b
          ] })#g,
          HttpWrite,
          a
        ] })#f,
        HttpRead,
        A
     ]

--

# &#8253;

--

## Stacking types

    private class Level[M[_, _, _], A, B] {
      type t[a] = M[A, B, a]
    }

    case class Http[A](run:
      ReaderT[
        Level[WriteT,
          Level[StateT,
            Result,
            HttpState
          ]#t,
          HttpWrite
        ]#t,
        HttpRead,
        A
      ]
    )

--

## Scalaz for the win!

    import scalaz._

    case class Http[A](
      run: RWST[Result, HttpRead, HttpWrite, HttpState, A]
    )

--

## What about further generalisation?

    case class HttpRead[A](path: String, reqheaders: Headers, config: A)

    trait ResultT[M[_], A]

--

## Scala for the loss

    case class HttpRead[A](path: String, reqheaders: Headers, config: A)

    trait ResultT[M[_], A]

    type Result[A] = ResultT[Identity, A]

    case class Http[A, B](
      run: RWST[Result, HttpRead[A], HttpWrite, HttpState, B]
    )

    // bletch... inference failures, compiler errors, pain

--

## First principles for the win!

    case class Http[A, B](run:
      (HttpRead[A], HttpState) => Result[(HttpWrite, HttpState, B)]
    )

---

## A friendly reminder that
# Order Matters

--

## Maybe we really want?

    case class Http[A, B](run:
      (HttpRead[A], HttpState) => (HttpWrite, HttpState, Result[B])
    )


---

## Back to our simplified case

    case class Http[A](run:
      (HttpRead, HttpState) => (HttpWrite, HttpState, Result[A])
    )


--

## Combinators for fun and profit

    object Http {
      def modify(f: HttpState, HttpState): Http[Unit] =
        Http((_, state) => (Vector(), f(state), Cont(())))

      def get: Http[HttpState] =
        Http((_, state) => (Vector(), state, Cont(state)))

      def ask: Http[HttpRead] =
        Http((read, state) => (Vector(), state, Cont(read)))
    }

--

## Combinators for fun and profit

    object ResponseStatus {
      def modify(f: HttpStatus => HttpStatus): Http[Unit] =
        Http.modify(s => s.copy(status = f(s.status)))

      def get: Http[HttpStatus] =
        Http.get map (_.status)

      def set(s: HttpStatus): Http[Unit] =
        modify(_ => s)
    }

--

## Combinators for fun and profit

    object ResponseHeaders {
      def modify(f: Headers => Headers): Http[Unit] =
        Http.modify(s => s.copy(resheaders = f(s.resheaders)))

      def headers: Http[Headers] =
        Http.get map (_.resheaders)

      def get(name: String): Http[Option[String]] =
        headers map (_.get(name))

      def all(name: String): Http[List[String]] =
        headers map (_.all(name))

      def set(name: String, value: String): Http[Unit] =
        modify(_ set (k, v))
   }

--

## Combinators for fun and profit

    object Errors {
      def send[A](content: String): Http[A] =
        done[A](copiedBuffer(content, UTF_8))

      def errorWith[A]: HttpStatus => String => Http[A] =
        status => message =>
          ResponseHeaders.set("content-type", "application/x-awesome") >>
          ResponseStatus.set(status) >>
          send(message)

      def error400[A] = errorWith(BAD_REQUEST)
      def error404[A] = errorWith(NOT_FOUND)
      def error500[A] = errorWith(INTERNAL_SERVER_ERROR)
      def error503[A] = errorWith(SERVICE_UNAVAILABLE)
    }

--

### Building from
# Small
# Composable
### blocks

--

## Bake in error handling

    object Body {
      def text: Http[String] =
        Http.ask map (_.request.getContent.toString(UTF_8))

      def json: Http[String \/ Json] =
        text map (Parser.parseEither)

      def json400: Http[Json] =
        json flatMap ({
          case -\/(message) => Errors.error400(message)
          case \/-(json) = json.point[Http]
        })

      def decode400[A: DecodeJson]: Http[A] = for {
        json400 flatMap (json => Decoder.decodeEither match {
          case -\/(message) => Errors.error400(message)
          case \/-(a) = json.point[Http]
        })
    }

--


## Winning

    object Service {
      def createUser: Http[User] = for {
        user <- Body.decode400[User]
        _ <- unlessM(user.name.length >= 4)(
          Errors.error400("Username must be 4 or more characters")
        )
        _ <- unlessM(user.password.length >= 4)(
          Errors.error400("Password must be 4 or more characters")
        )
        _ <- Db.save(user)
      } yield user
    }

--

## Factoring without the hassle

    object Validate {
      def validate(condition: Boolean, message: => String): Http[Unit] =
        unlessM(condition)(Errors.error400(message))
    }

--

## Look ma, no conditionals!

    import Validate._

    object Service {
      def createUser: Http[User] = for {
        user <- Body.decode400[User]
        _ <- validate(user.name.length >= 4,
          "Username must be 4 or more characters")
        _ <- validate(user.password.length >= 4,
          "Password must be 4 or more characters")
        _ <- Db.save(user)
      } yield user
    }

--

### Now we can build
# interpreters
### to manage our program behaviour

--

## Lifecycle Control

    object NaiveHttpInterpretter {
      def run[A](
        config: Config,
        req: HttpRequest,
        http: Http[A]
      ): Future[HttpResponse] = Future({
        val read = HttpRead(req, headers(req), config)
        val init = HttpState(Headers(), OK)
        http.run(read, init) match {
          case (log, state, HttpDone(f)) => ...
          case (log, state, HttpErr(f)) => ...
          case (log, state, HttpCont(f)) => ...
        }
      })
    )

--

## One Structure, Multiple Behaviours

    object SessionHttpInterpretter {
      def run[A](
        config: Config,
        req: HttpRequest,
        http: Http[A]
      ): Future[HttpResponse] = Future({
        // read session from cookie
        // validate
        // run
        // write new session back out
      })
    )

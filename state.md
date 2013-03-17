# State

--

### State

    type Headers = List[(String, List[String])]

    case class HttpData(headers: Headers, response: Int) {
      def setHeader(name: String, value: String) =
        copy(headers = headers.filter({ case (k, _) => k != name}) ++
          List((name, value: Nil)))

      def setResponse(response: Int) =
        copy(response = response)
    }

--

### State

    def length(data: HttpData, n: Int): HttpData =
      ???

    def contentType(data: HttpData, mime: String): HttpData =
      ???

    def json(data: HttpData, json: Json): (HttpData, String) =
      ???

    def parse(data: HttpData, input: String): (HttpData, Option[Json]) =
      ???

--

### State

    case class State[S, A](run: S => (S, A))

--

### State


    type HttpState[A] = State[HttpData, A]

    def length(n: Int): HttpState[Unit] =
      ???

    def contentType(mime: String): HttpState[Unit] =
      ???

    def json(json: Json): HttpState[String] =
      ???

    def parse(input: String): HttpState[Option[Json]] =
      ???

--

### A library for State

    case class State[S, A](run: S => (S, A)) {
      def map[B](f: A => B): State[S, B] =
        ???

      def flatMap[B](f: A => State[S, B]): State[S, B] =
        ???
    }

--

### A library for State

    object State {
      def value[S, A](a: => A): State[S, A] =
        ???

      def get[S]: State[S, S] =
        ???

      def gets[S, A](f: S => A): State[S, A] =
        ???

      def modify[S](f: S => S): State[S, Unit] =
        ???

      def put[S](s: S): State[S, Unit] =
        ???
    }

--

# Code

--

### A library for State

    case class State[S, A](run: S => (S, A)) {
      def map[B](f: A => B): State[S, B] =
        flatMap(a => State.value(f(a)))

      def flatMap[B](f: A => State[S, B]): State[S, B] =
         State(s => run(s) match {
           case (ss, a) => f(a).run(ss)
         })
    }

--

### A library for State

    object State {
      def value[S, A](a: => A): State[S, A] =
        State(s => (s, a))

      def get[S]: State[S, S] =
        State(s => (s, s))

      def gets[S, A](f: S => A): State[S, A] =
        get map f

      def modify[S](f: S => S): State[S, Unit] =
        State(s => (f(s), ()))

      def put[S, A](s: S): State[S, Unit] =
        modify(_ => s)
    }

--

### (Not) Using our library

# Censored

--

### Using our library

    import State._

    def setHeader(name: String, value: String): HttpState[Unit] =
      modify(s => s.setHeader(name, value))

    def length(n: Int): HttpState[Unit] =
      setHeader("Content-Length", n.toString)

    def contentType(mime: String): HttpState[Unit] =
      setHeader("Content-Type", mime)

--

### Using our library

    def json(json: Json): HttpState[String] = {
      val result = json.pretty
      for {
        _ <- length(result.getBytes("UTF-8").length)
        _ <- contentType("application/json")
      } yield result
    }

    def parse(input: String): HttpState[Option[Json]] =
      for {
        json <- Parser.parseOption(input)
        _    <- if (json.isDefined)
                  State.value(())
                else
                  State.modify(_.setResponse(400))
      } yield json

--

### Using our library with scalaz

    import scalaz._, Scalaz._

    def json(json: Json): HttpState[String] = {
      val result = json.pretty
      length(result.getBytes("UTF-8").length) >>
        contentType("application/json").as(result)
    }

    def parse(input: String): HttpState[Option[Json]] =
      for {
        json <- Parser.parseOption(input)
        _    <- unlessM(json.isDefined)(
                  State.modify(_.setResponse(400)))
      } yield json

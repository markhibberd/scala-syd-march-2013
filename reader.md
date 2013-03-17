# Reader

--

### Reader


      def auth(request: HttpRequest): Boolean =
        ???

      def method(request: HttpRequest): HttpMethod =
        ???

      def profile(request: HttpRequest)(connection: Connection): Json =
        ???

--

### Reader

    case class Reader[R, A](run: R => A)



--

### Reader


      def auth: Reader[HttpRequest, Boolean] =
        ???

      def method: Reader[HttpRequest, HttpMethod] =
        ???

      def profile(connection: Connection): Reader[HttpRequest, Json] =
        ???


--

### Reader

    //def auth(request: HttpRequest): Boolean ~>
      def auth: Reader[HttpRequest, Boolean] =
        ???

    //def method(request: HttpRequest): HttpMethod ~>
      def method: Reader[HttpRequest, HttpMethod] =
        ???

    //def profile(request: HttpRequest)(connection: Connection): Json ~>
      def profile(connection: Connection): Reader[HttpRequest, Json] =
        ???

--

### Reader

      type HttpReader[A] = Reader[HttpRequest, A]

      def auth: HttpReader[Boolean] =
        ???

      def method: HttpReader[HttpMethod] =
        ???

      def profile(connection: Connection): HttpReader[Json] =
        ???

--

### A library for Reader

    case class Reader[R, A](run: R => A) {
      def map[B](f: A => B): Reader[R, B] =
        ???

      def flatMap[B](f: A => Reader[R, B]): Reader[R, B] =
         ???
    }

    object Reader {
      def value[R, A](a: => A) =
        ???

      def ask[R]: Reader[R, R] =
        ???

      def local[R, A](f: R => R)(reader: Reader[R, A]): Reader[R, A] =
        ???
    }

--

# Code

--

### A library for Reader

    case class Reader[R, A](run: R => A) {
      def map[B](f: A => B): Reader[R, B] =
        flatMap(a => Reader.value(f(a)))

      def flatMap[B](f: A => Reader[R, B]): Reader[R, B] =
        Reader(r => f(run(r)).run(r))
    }

    object Reader {
      def value[R, A](a: => A): Reader[R, A] =
        Reader(_ => a)

      def ask[R]: Reader[R, R] =
        Reader(r => r)

      def local[R, A](f: R => R)(reader: Reader[R, A]): Reader[R, A] =
        Reader(r => reader.run(f(r)))
    }

--

### (Not) Using our library

    def header(request: HttpRequest, name: String): List[String] =
      request.getHeaders(name).asScala.toList

    def method(request: HttpRequest): HttpMethod =
      request.getMethod

    def auth(request: HttpRequest): Boolean =
      method(request) match {
        case PUT | POST =>
          header(request, "Authorization").headOption.
            exists(_ == "let me in")
        case GET =>
          true
      }

--

### Using our library

    def header(name: String): HttpReader[List[String]] =
      ask[HttpRequest] map (_.getHeaders(name).asScala.toList)

    def method: HttpReader[HttpMethod] =
      ask[HttpRequest] map (_.getMethod)

    def auth: HttpReader[Boolean] = for {
      m <- method
      r <- m match {
        case PUT | POST =>
          header("Authorization") map (_.headOption.
            exists(_ == "let me in"))
        case GET =>
          Reader.reader[HttpRequest, Boolean](true)
      }
    } yield r

--

### Using our library with scalaz

    import scalaz._, Scalaz._

    def header(name: String): HttpReader[List[String]] =
      ask[HttpRequest] map (_.getHeaders(name).asScala.toList)

    def method: HttpReader[HttpMethod] =
      ask[HttpRequest] map (_.getMethod)

    def auth: HttpReader[Boolean] =
      method >>= (_.matchOrZero({
        case PUT | POST =>
          header("Authorization") map (_.headOption.
            exists(_ == "let me in"))
      }))

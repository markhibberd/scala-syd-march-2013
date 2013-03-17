# Composition

--

### Picking out multiple patterns

    def service(request: HttpRequest, data: HttpData): (HttpData, Json) =
      ???

--

### Reader with State

    def service: HttpReader[HttpState[Json]] =
      ???

--

### What if we have Reader and Writer and State?

    def service: HttpReader[HttpWriter[HttpState[Json]]] =
      ???

--

### What if we have Reader and Writer and State and Option?

    def service: HttpReader[HttpWriter[HttpState[Option[Json]]]] =
      ???

--


## This is bad!

--

### What about our library?

    def user: HttpReader[Option[User]] =
      ???

    def profile(u: User): HttpReader[Option[Profile]] =
      ???

    def widget(u: User): HttpReader[Option[Widget]] =
      ???

    def render(u: User, p: Profile, w: Widget): Json =
      ???

--

### What about our library?

    def service: HttpReader[Option[Json]] = for {
      ou <- user
      op <- Reader(r => for {
        u <- ou
        result <- profile(u).run(r)
      } yield result)
      ow <- Reader(r => for {
        u <- ou
        result <- widget(u).run(r)
      } yield result)
    } yield for {
      u <- ou
      p <- op
      w <- ou
    } yield render(u, p, w)

--

## This is really bad!

--

## We demand

# Elegant composition

##### and

# Convenient Libraries

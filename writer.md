# Writer

--

### Writer

    sealed trait Audit {
      def trail: List[Audit] = List(this)
    }

    case class AuditRead(who: String, what: String)
      extends Audit
    case class AuditWrite(who: String, what: String, mod: String)
      extends Audit
    case class AuditUnauth(who: String, what: String)
      extends Audit

    type AuditTrail = List[Audit]

--

### Writer

    def create(p: Profile): (AuditTrail, Boolean) =
      ???

    def read(id: Long): (AuditTrail, Option[Profile]) =
      ???

    def update(id: Long, p: Profile): (AuditTrail, Boolean) =
      ???

    def delete(id: Long): (AuditTrail, Boolean) =
      ???

--

### Writer

    case class Writer[W, A](log: W, value: A)

--

### Writer

    type AuditWriter[A] = Writer[AuditTrail, A]

    def create(p: Profile): AuditWriter[Boolean] =
      ???

    def read(id: Long): AuditWriter[Option[Profile]] =
      ???

    def update(id: Long, p: Profile): AuditWriter[Boolean] =
      ???

    def delete(id: Long): AuditWriter[Boolean] =
      ???

--

### A library for Writer

    case class Writer[W, A](log: W, value A) {
      def map[B](f: A => B): Writer[W, B] =
        ???

      def flatMap[B](f: A => Writer[W, B]): Writer[W, B] =
        ???
    }

    object Writer {
      def value[W, A](a: A) =
        ???

      def tell[W](w: W): Writer[W, Unit] =
        ???

      def writer[W, A](a: A)(w: W): Writer[W, A] =
        ???
    }

--

# Code

--

### A library for Writer

    case class Writer[W, A](log: W, value: A) {
      def map[B](f: A => B): Writer[W, B] =
        Writer(log, f(value))

      def flatMap[W: Monoid](f: A => Writer[W, B]): Writer[W, B] = {
        val w = f(value)
        Writer(log |+| w.log, w.value)
      }
    }
    object Writer {
      def value[W, A](a: => A)(implicit W: Monoid[W]) =
        Writer(W.zero, a)

      def tell[W](log: W): Writer[W, Unit] =
        Writer(log, ())

      def writer[W, A](a: A)(w: W): Writer[W, A] =
        Writer(w, a)
    }

--

### (Not) Using our library


    def create(p: Profile): (AuditTrail, Boolean) =
      (AuditWrite("the dude", "profile", "create").trail, true)

    def read(id: Long): (AuditTrail, Option[Profile]) =
      (AuditUnath("the dude", "profile").trail, None)

    def update(id: Long, p: Profile): (Auditrail, Boolean) = {
      val (l1, success) = delete(id)
      if (success) {
        val (l2, _) = create(p)
        (l1 |+| l2, true)
      } else
        (l1, false)
    }

    def delete(id: Long): (AuditTrail, Boolean) =
      (AuditWrite("the dude", "profile", "delete").trail, true)

--

### Using our Library

    def create(p: Profile): AuditWriter[Boolean] =
      writer(true)(AuditWrite("the dude", "profile", "create").trail)

    // alternative could be as above ^^^^
    def read(id: Long): AuditWriter[Option[Profile]] = for {
      r <- Writer.value(None)
      _ <- Writer.tell(AuditUnauthWrite("the dude", "profile").trail)
    } yield r

    def update(id: Long, p: Profile): AuditWriter[Boolean] =
      for {
        deleted <- delete(id)
        result  <- if (deleted) create(p) else writer(false)
      } yield result

    def delete(id: Long): AuditWriter[Boolean] =
      writer(true)(AuditWrite("the dude", "profile", "delete").trail)

--

### Using our Library with scalaz

    import scalaz._, Scalaz._

    def create(p: Profile): AuditWriter[Boolean] =
      writer(true)(AuditWrite("the dude", "profile", "create").trail)

    def read(id: Long): AuditWriter[Option[Profile]] =
      writer(None)(AuditUnauthWrite("the dude", "profile").trail)

    def update(id: Long, p: Profile): AuditWriter[Boolean] =
      delete(id).ifM(create(p), false.pure[AuditWriter])

    def delete(id: Long): AuditWriter[Boolean] =
      writer(true)(AuditWrite("the dude", "profile", "delete").trail)

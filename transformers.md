# Monad Transformers

--

## Monad Transformers allow us to stack our types together, creating new single types

--

## Monad Transformers give us the ability to deal with the stack as a whole or its parts

--

## Avoid the stairs of death &lt;superscript&gt;[1]&lt;/superscript&gt;

      for {
        ...
      } yield for {
        ...
        } yield for {
           ...
          } yield for {
            ...
            } yield ()

&lt;footnote&gt;
&lt;small&gt;
  [1] I first heard the "stairs" phrase from Jordon West, in his [talk on transformers in scalamachine](http://www.youtube.com/watch?v=S_l95GIDCM0)
&lt;/small&gt;
&lt;/footnote&gt;

--

## Dealing with Reader and Option as a whole

--

## Reader

    case class Reader[R, A](run: R => A)

--

## Reader with Option

    case class Reader[R, A](run: R => A)

    case class ReaderOption[R, A](run: R => Option[A])

--

## Reader Transformer Generalised

    case class Reader[R, A](run: R => A)

    case class ReaderOption[R, A](run: R => Option[A])

    case class ReaderT[M[_], R, A](run: R => M[A])

--

## Writer

    case class Writer[W, A](run: (W, A)])

--

## Writer with Option

    case class Writer[W, A](run: (W, A)])

    case class WriterOption[W, A](run: Option[(W, A)])

--

## Writer Transformer Generalised

    case class Writer[W, A](run: (W, A)])

    case class WriterOption[W, A](run: Option[(W, A)])

    case class WriterT[M[_], W, A](run: M[(W, A)])

--

## State

    case class State[S, A](run: S => (S, A)])

--

## State with Option

    case class State[S, A](run: S => (S, A))

    case class StateOption[S, A](run: S => Option[(S, A)])

--

## State Transformer Generalised

    case class State[S, A](run: S => (S, A)]

    case class StateOption[S, A](run: S => Option[(S, A)]

    case class StateT[M[_], S, A](run: S => M[(S, A)]

--

# Code

--

# Saving our library

--

## MonadTrans

    trait MonadTrans[F[_[_], _]] {
      def lift[G[_]: Monad, A](g: G[A]): F[G, A]
    }

    // usage: g.lift[ReaderT]
    case class MonadTransSyntax[G[_], A](g: G[A]) {
      def lift[F[_[_], _]](implicit T: MonadTrans[F], M: Monad[G]): F[G, A] =
        T.lift(g)
    }

    object MonadTransSyntax {
      implicit def ToMonadTransSyntax[G[_], A](g: G[A]) =
        MonadTransSyntax(g)
    }

--

## MonadReader

    trait MonadReader[F[_[_], _], R] {
      def ask: F[R, R]
      def local[A](f: R => R)(reader: F[R, A]): F[R, A]
    }

--

## MonadWriter

    trait MonadWriter[F[_[_], _], W] {
      def tell(w: W): F[W, Unit]
    }

--

## MonadState

    trait MonadState[M[_[_], _], S] {
      def get: F[S, S]
      def gets[A](f: S => A): State[S, A]
      def put: F[S, Unit]
      def modify(f: S => S): F[S, Unit]
    }

--

# Working with Monad Transformers


--

### Remember This?

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

### With a little help from our friends

    def service: ReaderT[Option, HttpRequest, Json] = for {
      u <- user
      p <- profile(u)
      w <- widget(u)
    } yield render(u, p, w)

--

## Reminder that today's factoring is brought to you by
# Types,
# Equational Reasoning
#### and
# Laws


# Why?

--

## Types + Laws &rArr; Libraries

--

# Yes, Free Code

--

### By building rich libraries, with well defined data types, we get
# Easy To Factor,
# Compositional
## Code


---

# Laws

--


## A law defines properties that are  _assumed_ to be held

--

## Laws give us a basis on which to reason at a deeper level

--

#### We leverage laws to
## refactor,
## build libraries,
## choose execution strategies,
## analyse code

--

## Laws in maths

#### Integer addition obeys the _associative law_


                          (1 + 4) + 8 = 1 + (4 + 8)

#### Integer addition obeys the _commutative law_

                                9 + 7 = 7 + 9

--

## Laws in programming

### A Monoid

    trait Monoid[F] {
        def zero: F
        def append(a: F, b: F): F
    }

--

### Monoid laws

    // &forall; a, b in F, a + b is also in F
    def closure[F: Monoid](a: F, b: F): F = // enforced by type system
      a |+| b

    // &forall; a, b, c. (a + b) + c == a + (b + c)
    def associative[F : Monoid : Equal](a: F, b: F, c: F): Boolean =
      ((a |+| b) |+| c) === (a |+| (b |+| c))

    // &forall; a. zero + a = a
    def leftIdentity[F : Monoid: Equal](a: F): Boolean =
      (implicitly[Monoid].zero |+| a) === a

    // &forall; a. a + zero = a
    def rightIdentity[F : Monoid: Equal](a: F): Boolean =
      (a |+| implicitly[Monoid].zero) === a

--

## Laws are typically intuitive

--

### A law for serialisation

#### We have a hypothetical serialisation library, what would be a useful law?

    // Our hypothetical serialisation library:
    def encode[A: Codec](a: A): SuperSpeedyFormat = ???
    def decode[A: Codec](v: SuperSpeedyFormat): Option[A] = ???

--

### A law for serialisation

    // Our hypothetical serialisation library:
    def encode[A: Codec](a: A): SuperSpeedyFormat = ???
    def decode[A: Codec](v: SuperSpeedyFormat): Option[A] = ???

    // forall a. decode(encode(a)) = Some(a)
    def roundTrip[A: Codec](a: A): Boolean =
      decode(encode(a)) exists (_ == a)

--

## Breaking laws is
#  bad

--

## Breaking laws
#### produces
# unexpected
#### and
# inconsistent
## behaviour


--

## Breaking laws
# places an unnecessary burden on your client


--

### A small example
# Broken Laws
#### and
# Contract Killers

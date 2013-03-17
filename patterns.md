
# Patterns

--

## Duplication

    // Duplication for the Win!!!!!
    def wat(a: String, b: String) = {
      val aa = a.substring(0, if (a.length > 0) a.length - 1 else 0)
      val bb = b.substring(0, if (b.length > 0) b.length - 1 else 0)
      val aaa = aa.length * 111
      val bbb = bb.length * 222
      aaa + bbb
    }

--

![Angry](images/hulk.jpg "Angry")

--

## Duplication?

    def huey(request: HttpRequest): String =
      ???

    def dewey(request: HttpRequest): Int =
      ???

    def louie(request: HttpRequest): List[String] =
      ???

--

# Meh?
### or
# Rage?

--

## Duplication?

    def huey(n: Int): (Audit, String) =
      ???

    def dewey(s: String): (Audit, Int) =
      ???

    def louie(l: List[String]): (Audit, Int) =
      ???

--

## Duplication?

    def huey(c: Cache, n: Int): (Cache, String)  =
      ???

    def dewey(c: Cache, s: String): (Cache, Int) =
      ???

    def louie(c: Cache, l: List[String]): (Cache, Int) =
      ???

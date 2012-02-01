<center>The Structure of Streams</center>
=========================================

What Are Streams?
-----------------

First of all, we should make it clear that the Streams we're talking about here
are not I/O streams. That usage of the term "stream" is another perfectly valid
use of the word, but it is also a completely different meaning of the
word. Streams, as described here, are a data type that exists in Scala and
other functional programming languages. This data type has no relation to I/O
streams, other than that they both refer to potentially never-ending sources of
data.

Streams are a staple of functional programming. They are "lazy" lists. I.e., in
functional programming, a non-lazy List is a linked list data structure that
looks something like this:

<center>
<img src="http://upload.wikimedia.org/wikipedia/commons/1/1b/Cons-cells.svg" width="300">
</center>

The above diagram represents `List(42, 69, 613)`. The is made up of a
collection of "cons cells"<sup>1</sup>, where each cell contains one element of
the list and a pointer to the rest of the list.

Note that the entire List is contained in ram. This fact can be mighty
inconvenient if you'd like to, say, have an infinite list. (I hear that an
infinite amount of ram is pricey these days on the spot market.) For instance,
let's say that we want a list that starts at 0, has 1 as its next element, and
then just keeps going forever. You might think that we're just SOL, but we
aren't. We just have to embrace the idea of a "lazy list", a.k.a., a "Stream".

A Stream is just like a List, except that where the cons cell of a List
contains a pointer to the rest of the List, the cons cell of a Stream contains
an anonymous function instead. When you call ".tail" on a List, the pointer is
followed to return the rest of the List. When you do the same thing on a
Stream, instead of the pointer being followed, the anonymous function in the
cons cell is called. This function then generates and returns the next cons
cell in the Stream. Since this new cons cell also contains its own anonymous
function, this process of generating new cons cells on demand can continue on
indefinitely. If and when there are no more elements in the stream, the
function in the last cons cell will return Stream.empty.

In summary, the cons cell returned by ".tail" on a Stream contains two
things:

   1. The next element of the stream.

   2. A function to generate the cons cell that comes after the next
      one.

In this fashion, the stream can generate *any* number of elements, and it
will do so only as the elements are actually needed.

In Scala, constructing the aforementioned infinite stream of the whole
numbers is exceptionally easy:

        val wholes = Stream.from(0)

I know, I know... you're thinking that this is all fine and nifty, but don't we
already have Iterators to serve this very purpose? Why, yes we do! And
Iterators are also nifty keen and can be used a lot like Streams are used. The
problem with Iterators is that they are not purely functional. I.e., a purely
functional data structure is immutable -- it cannot be modified. This property
allows us to reason more easily about certain properties of our programs. When
we pull from an Iterator, on the other hand, the Iterator is modified and this
can result in bugs. For instance, if you add some debugging code to print out
the next element of an Iterator, that debugging code has consumed that value
from the Iterator, and so that value will no longer make it to where it was
supposed to go. When you use Streams instead, this particular bug cannot
occur.

Composing Streams
-----------------

We created an infinite list of whole numbers above. Great! Now what? Hey, how
about we generate a Stream of all the even numbers. That's easy enough:

    	val evens = wholes map { _ * 2 }


Now let's add up the first 10 even number:

    	val sum10 = (evens take 10) sum


Now let's do an element-wise addition of all the odd and even numbers:

		val sumOfOddAndEvens = (evens zip odds) map { p => p._1 + p._2}

If we now do

   		sumOfOddAndEvens take 10 print

we'll see the following output:

    	1, 5, 9, 13, 17, 21, 25, 29, 33, 37, empty

That was fun, but let's try something a little trickier. We'll now make make an
infinite stream of Fibonacci numbers:

		val fibs: Stream[Int] = (
           1 #::
           1 #::
           ((fibs zip fibs.tail) map { p => p._1 + p._2 }))


"`#::`" in Scala is the Stream cons operator. (You can also invoke it as
"`Stream.cons()`".) The above line of code conses up a Stream where the first
element is 1, the next element is also 1, and then the rest of the stream is
created by zipping the infinite stream of Fibonacci numbers with itself, only
offset by one, and adding together the zipped pairs. Elegance incarnate!

One thing to note about how this works is that the code after "`#::`" above is
not run immediately. If it were, this wouldn't work, since "`fibs`" wouldn’t be
defined yet at the time it was being used. Everything to the right of "`#::`"
is invoked via "call by name", which means that is wrapped up in an invisible
function literal, and that function is not called until it is needed.

Now let's get really funky and generate prime numbers! The following bit of
code will put an infinite stream of primes into the val "`primes`". Rad!

        def sieve(s: Stream[Int]): Stream[Int] =
           Stream.cons(s.head, sieve(s.tail filter { _ % s.head != 0 }))

        val primes = sieve(Stream.from(2))
	
The above code implements a Sieve of Eratosthenes, which is a way of itemizing
all the primes. Let's say that we were to perform by hand on paper the Sieve of
Eratosthenes algorithm. We might start by first writing down *all* the integers
that are greater than or equal to 2. Then we'd circle the least number that
isn't already circled. Since we'd have just started, that number will be 2,
which is our first prime. Then we'd cross off all the multiples of 2 that are
on our infinitely long piece of paper. Once we'd finished that, we'd go back to
the circling step and circle the least number that isn't already circled or
crossed off. In this manner, the second number we circled would be 3, which is
our next prime. We'd then cross off all the multiples of 3 from our infinite
list. We'd lather, rinse, repeat until the end of time, and when we'd finished,
what's left is only circled numbers, and these numbers are the primes.

So, how does the above code implement this? We'll leave that as an exercise for
the reader. Also, note that because Scala does not support fully general tail
recursion, this solution will blow the heap if you walk down the stream far
enough. (They say that Scala is the gateway drug to Haskell. In Haskell, you
would not blow the heap with this implementation.)

java.lang.OutOfMemoryError: Java heap space
-------------------------------------------

Unfortunately, all is not honeydew and the milk of paradise in Streams
Land. The reason for this is that in Scala, a Stream caches the element
values. This can be very handy at times. For instance, if you have a very
compute intensive stream, and you need to walk down it several times, you may
not want to pay the full CPU cost each time you walk down it.

More significantly, if you use Streams for reading from a data source of some
kind, the data may not exist anymore if it were not cached. E.g., imagine that
you implement a stream that represents the output from a database query. In
this example, each element of the Stream is a row in the result set. Now let's
say that you walk down the Stream to fetch the tenth row, but you don't pay any
heed to first nine rows when you do so. Later you decide that you want the
fifth row. Too bad! It's gone. At least if the result set was provided by a
unidirectional database cursor, it is. Consequently, we can't tolerate this
sort of behavior from Streams--they are supposed to be purely functional, but
if walking down a Stream invalidates previous elements of the Stream, then
Streams are no different from Iterators. So, this is another reason that
Scala caches the element values for Streams.

Unfortunately, this caching has a huge downside: You can easily exhaust the
entire heap if you are not extremely careful. E.g., if you fetch the millionths
row from a Stream that represents a database query, you will almost certainly
exhaust the heap, as all the other 999,999 rows will end up being cached. What
to do?!?

At this point, you might be interested to know that the programming language
Scheme, which is where Streams originated, does not cache Stream
values. Perhaps it doesn't do so as an answer to this very conundrum. The
downside for Scheme here is that Streams in Scheme are no good for reading I/O
for the reasons mentioned above. You might also be interested to know that
Scalaz provides a facility called Ephemeral Streams that are more like Scheme's
Streams. No doubt it also does so as an answer to this conundrum.

In any case, we now know what Scheme's approach to solving this problem
is. What’s Scala's? To answer this, it might be instructive to think about how
Haskell handles this very same problem. Haskell's streams are just like Scala’s
Streams, so why doesn't Haskell have this problem?

The answer to this question is because in Haskell ***everything*** is lazy.

Let's take a step back and look at an example of how being "strict" rather than
"lazy" comes back to bit us when using Streams in Scala. Consider what happens
if we do the following:

    	val evens = wholes map { _ * 2 }
        for (n <- evens) println(n)

At first blush, it would seem that this program will run forever. And if we had
an infinite amount of memory it would. But every time through the loop, the
stream bound to `evens` will cache one more value, and eventually something's
got to give. I.e., we'll exhaust the heap.

How is this avoided in Haskell? The answer to this is that in Haskell,
*everything* is lazy. I.e., the variable `evens` isn't bound to a Stream of
evens when the first line is evaluated. What happens in the same thing that
would happen if we were to do the following in Scala:

    	def evens = wholes map { _ * 2 }
        for (n <- evens) println(n)

This revised Scala program will in fact run forever.


YOU ARE HERE: Put something about this not being Haskell syntax. And that this
kind of Stream is taken from Haskell.


Footnotes
---------

<sup>1</sup>The term "cons" is a bit of historical terminology from the Lisp
programming language dating back to 1956. It stands for "construct".


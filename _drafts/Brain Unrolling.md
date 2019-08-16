---
title: "Brain Unrolling"
categories: [C++]
tags: [C++, C++20, generators, coroutines]
---
> Generators and the Sweet Syntactic Sugar of Coroutines

![windmills](../../assets/windmills.jpg)

## The Cambrian

<p align="center"><img src="../../assets/trilobite.png" width="30px"/></p>

### Good Old Functions

Say we want to iterate over all the elements of a vector.  
We can write a function - a.k.a. ***sub-routine*** - to do that:

```cpp
void vectorate(std::vector<int> const& v)
{
    for (auto e: v)                 // 1. iterate
        std::cout << e << '\n';     // 2. do something: print e
}
```

If we want to draw a line on some image or device, then algorithms in books and resources will typically look something like this function (a simplified [Bresenham](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm) variant):

```cpp
void drawline(int x0, int y0, int x1, int y1) // TODO: Test Code
{
    int dy=y1-y0;
    int x=x0;
    int y=y0;
    int p=2*dy-dx;
    while(x<x1)                     // iterate
    {
       putpixel(x,y,7);             // do something: call putpixel()
       if(p>=0)
       {
           y=y+1;
           p=p+2*dy-2*dx;
       }
       else
       {
           p=p+2*dy;
       }
       x=x+1;
    }
}
```

In both examples, the functions do ***two*** things:

1. They iterate over a sequence, i.e. the vector or the pixel positions along a line;
2. They do some operation with each of the sequence elements

If we wanted to sum the vector elements, we would need to re-write `vectorate()` to sum the elements instead of (or in addition to) printing them.

Similarly, `drawline()` assumes that a function `putpixel()`:

1. is available in scope (for compilation and linking);
2. has the correct signature;
3. will do the right thing when called;
4. will actually return control to `drawline()` for continued operation.

A larger issue here is that both functions are "closed" in the sense that they only return after they have iterated over the *whole* sequence. They *eagerly* process a whole sequence.  

### Callbacks

One common way to overcome some of these limitations is by passing external *callback functions* to our own functions. Some C++ mechanisms that allow this are function-pointers or using "Callable" template parameter types (or Concepts in C++20) for passing e.g. lambdas. This is exactly how predicates work in many STL algorithms for example.

A few well known problems with callbacks and callables are: 

- *Inversion-of-Control*: Letting library code call external code that is not neccesarily trustworthy, valid or correct while still in mid-computation.
- *Callback-Hell*: Where program flow skips between many decoupled parts of the code that your code becomes extremely hard to understand, reason about and maintain (not to mention any potential performance issues).
- The functions are still *eager* and closed, requiring the creation/processing of the full sequence.

The concept of a function, or ***sub-routine*** goes back to one of the first computers, the ENIAC, in the late 1940s and the term *sub-routine* is from the early 1950s.

<p align="center">🤔</p>

> If only there was a way to "flip" these iterating functions "inside-out" and iterate over a sequence without pre-committing to, or having to specify a, specific operation...

## Pre-History

<p align="center"><span style="font-size:2em;">🦖</span></p>

### Iterators

The concept of Iterators has been with C++ since the STL was designed by Alex Stepanov and together with the rest of the STL became part of C++98. They are a powerful abstraction for, well, "openly" iterating over elements of a sequence. 

Two related concepts are *Iterator Objects* and *Iterator Adaptors*. These are "stand-alone" iterator types which are often only indirectly or implicitly coupled to a sequence. The C++ standard actually has quite a few of them, both old and new including, for example:

- [`std::istream_iterator`](https://en.cppreference.com/w/cpp/iterator/istream_iterator): a single-pass input iterator object that reads successive objects of type `T` from the `std::basic_istream` object for which it was constructed, by calling the appropriate `operator>>`. The actual read operation is performed when the iterator is incremented, not when it is dereferenced. The first object is read when the iterator is constructed. Dereferencing only returns a copy of the most recently read object.
- [`std::reverse_iterator`](https://en.cppreference.com/w/cpp/iterator/reverse_iterator): an *iterator adaptor* that reverses the direction of a given iterator.
- [`std::recursive_directory_iterator`](https://en.cppreference.com/w/cpp/filesystem/recursive_directory_iterator): an *iterator object* that iterates over the `directory_entry` elements of a directory, and, *recursively*, over the entries of all subdirectories. Here the file system directory sub-tree hierarchy is the "sequence" and not an actual program object with elements (since C++17). 
- Interestingly, the *functions* [`std::prev_permutation()`](https://en.cppreference.com/w/cpp/algorithm/prev_permutation) and [`std::next_permutation()`](https://en.cppreference.com/w/cpp/algorithm/next_permutation) are *not* iterators but do have some iterator-like behavior combined with mutating the underlying sequence.

A useful example of a *user defined* iterator object is OpenCV's [`cv::LineIterator`](https://docs.opencv.org/master/dc/dd2/classcv_1_1LineIterator.html) which is *"used to iterate over all the pixels on the raster line segment connecting two specified points"*. It has a typical iterator object type API (simplified for brevity):

```cpp
class LineIterator
{
public:
    // creates iterator for the line connecting pt1 and pt2 in img 
    // the 8-connected or 4-connected line will be clipped on the image boundaries
    LineIterator( const Mat& img, Point pt1, Point pt2, int connectivity = 8);
    uchar* operator *();           // returns pointer to the current pixel
    LineIterator& operator ++();   // prefix increment operator (++it). shifts iterator to the next pixel

    // public (!!!) members [ <groan 😩> ]  
    uchar* ptr;
    const uchar* ptr0;
    int step, elemSize;
    int err, count;
    int minusDelta, plusDelta;
    int minusStep, plusStep;
};
```

This iterator does not have an explicit sequence to iterate over. Instead it *lazily* creates the elements of the sequence as the iterator is incremented (using various Bresenham variants).  

`cv::LineIterator` gives us incremental access to all the pixels along a line in the image. We may draw the line by setting color values to the pixels, easily implementing e.g. line color gradients external to the iteration itself. Alternativly, we may sum the pixel values along the line or mix its value with some alpha channel. The possibilities are endless and need not be known to `cv::LineIterator` itself. 

Objects that lazily generate values on demand are also known as **Generators**.

Here is a usage example from the [docs](https://docs.opencv.org/master/dc/dd2/classcv_1_1LineIterator.html#details):

```cpp
cv::LineIterator it(img, pt1, pt2, 8);
std::vector<cv::Vec3b> buf(it.count);
for(int i = 0; i < it.count; i++, ++it) // copy pixel values along the line into buf
    buf[i] = *(const cv::Vec3b*)*it;
```

It externalizes the iteration to the user code.

### Imperfect Abstraction

However, there still are a several abstraction-related problems with this code (horrible public members notwithstanding). 

First, the iterator's `operator*()` (i.e. `*it` in the snippet) returns a `uchar*` which must be cast to a pointer of the actual pixel type of the image. This is awkward and could be fixed by e.g. deriving a class template from `cv::LineIterator` when the image pixel type is known at compile time (similar to how `cv::Mat_` is a template matrix class derived from `cv::Mat`.)  
But this is a `cv::LineIterator` specific issue, so we will not dwell on it here (though we shall return to it later).

There are more severe concerns which are, in fact, symptomatic for all iterator objects.

### The Odd Couple

<!-- ![odd-couple](https://media.giphy.com/media/E22xshq7vPL7q/giphy.gif) -->

How do we know when to stop incrementing the iterator (e.g. `++it`)?  
How do we know the sequence is "done"?

This is a question all iterator objects must answer.

- For `cv::LineIterator` we must make sure we iterate at most `it.count` times.  
- For `std::istream_iterator`, the default-constructed `std::istream_iterator` is known as the [end-of-stream iterator](https://en.cppreference.com/w/cpp/iterator/istream_iterator). When a valid `std::istream_iterator` reaches the end of the underlying stream, it becomes equal to the "universal" end-of-stream iterator.  
- A `std::reverse_iterator` must be compared to the corresponding sequence's `rend()` iterator.
-  `std::recursive_directory_iterator` must be compared to the value returned by calling the free function `std::end()` on it.  

Note that the last two examples (`std::reverse_iterator` and `std::recursive_directory_iterator`) demonstrate one of the biggest drawbacks of the iterator abstraction: the end-iterator is tightly coupled, at run-time,  to the begin-iterator creation object. This is a pitfall when the provided end iterator is of the correct *type* but *not* created from the same sequence. It is undefined behavior. The code will compile silently and if you're really lucky you'll get a crash (if not, nasal demons may ensue).

### Ranges

Ranges are a more general concept than iterators. They are *the* answer to the problems of the *Odd Coupling*. By encapsulating a begin *and* end iterator-*pair* or e.g. an iterator + size (or iterator and some way to check a stopping condition) they allow creating a single object that makes the STL iterators and algorithms more powerful by making them composable. Once we have the Range abstraction, we can build range adaptors and build pipelines that transform ranges of values in interesting ways.

**Ranges are coming to C++20** and are an *amazing* new addition to the standard library!

All the fabulous ranges we are getting are, in many ways, iterator objects and adaptors that provide the end-iterator in a standard mandated way (via `std::end()`). Their API is very similar to the APIs I presented above with the addition of support for `std::begin()` and `std::end()`.

I will not review the enormous power of ranges here. Instead, we'll only contemplate how Ranges may be implemented, and how we can create our own.  

However, since Ranges are generalized iterators, implementing them still suffers from another difficulty that plagues iterator implementations...

### Distributed Logic

The iterator object cousin of *Callback-Hell*, is the iterator API requirement for *distributed logic* and *centralized-state*. While the iteration loop is abstracted away to the external user, intermediate iteration/computation variables are stored as (mutable) members, and iteration logic is split between the constructor and member methods like the increment `operator++` (the indirection `operator*()` is usually trivial). 

Let's look at the implementation of `cv::LineIterator`:

```cpp
//...
inline uchar* LineIterator::operator *()         // trivial
{   return ptr; }

inline LineIterator& LineIterator::operator ++() // loop iteration logic
{
    int mask = err < 0 ? -1 : 0;
    err += minusDelta + (plusDelta & mask);
    ptr += minusStep + (plusStep & mask);
    return *this;
}
//...
```

After the [constructor](https://github.com/opencv/opencv/blob/3f42122387865af7aae04226c642854fc7bb0ac0/modules/imgproc/src/drawing.cpp#L163) (not shown) sets up all the member variables, it is up to the user to iterate and increment the iterator via `++` at most `.count` times. The "current" pixel along the line is the one pointed to by `.ptr`.

Basically, the `operator++()` body is exactly the iterating `for`-loop body and  where instead of performing some prescribed operation on the current element, the element is "returned" by updating the `ptr` member and returning `*this` to allow calling the indirection operator for actually accessing it.

To write [grok] `cv::LineIterator`, one must write [read] the constructor, then the indirection operator and the increment operators (not to mention additional methods and operators like the post-increment operator).

Additionally, by storing all the intermediate data as persistent members in the object (even if they are *not* public), we do not take advantage of scoped definition and locality (i.e. all methods can access and modify them at any point in the computation), opening the door for potential bugs, performance issues and increased object sizes.

Contrast this to the serial clarity of reverting `cv::LineIterator` to a non-iterable, but *serial*, function similar to what we saw at the beginning. It would look something like this:

```cpp
void processLine(const Mat& img, Point pt1, Point pt2,...)
{
    // local variables (cv::LineIterator member variables)
    uchar* ptr;
    const uchar* ptr0;
    int step, elemSize;
    int err, count;
    int minusDelta, plusDelta;
    int minusStep, plusStep;

    // initialize local variable (cv::LineIterator::LineIterator() ctor)
    // ...

    // Now draw the line
    for(int i = 0; i < count; ++i) // the explicit loop
    {
        // calculate the next element (LineIterator::operator++())
        int mask = err < 0 ? -1 : 0;
        err += minusDelta + (plusDelta & mask);
        ptr += minusStep + (plusStep & mask);

        doSomething(ptr); // <<!!! ptr is the "current" element/pixel
    }
```

<p align="center">🤔</p>

> If only there was a way to write a simple, serial, loop algorithm with locally scoped stack-based intermediate variables which is much easier to read, debug and reason about while still abstracting way the iteration...

## Present Day

<p align="center"><span style="font-size:2em;">🛫</span></p>

### Coroutines

> “Coroutines make it trivial to define your own ranges.”  
> — [Eric Niebler](http://ericniebler.com/2017/08/17/ranges-coroutines-and-react-early-musings-on-the-future-of-async-in-c/), Lead author of the C++ Ranges proposal (*edited for drama*)

Hmmm... is that so?  
But wait, what *are* **coroutines**?

> [**From Boost.Coroutine2**](https://www.boost.org/doc/libs/1_70_0/libs/coroutine2/doc/html/coroutine2/overview.html): A *coroutine* (coined by Melvin Conway in 1963!) is a function that can suspend execution to be resumed later. It allows suspending and resuming execution at certain locations and preserves the local state of execution and allows re-entering the subroutine more than once. In contrast to threads, which are pre-emptive, coroutine switches are cooperative: the programmer controls when a switch will happen. The kernel is not involved in the coroutine switches.

**This sounds just like what we want!**  
If we made `processLine()` above a *coroutine* then, instead of calling `doSomething(ptr)` in the loop body, we could (somehow) suspend the execution and ***yield*** the current value `ptr`. When the user so indicates (somehow), we can resume the computation from where we left off!

<p align="center"><br><span style="font-size:4em;">🤯</span></p> 

To preserve state across calls, the coroutine local stack must persist beyond the initial call and unlike a regular [sub-]routine/function after returning the coroutine stack must persist and should not be overwritten as that is where the state is stored. Moreover, re-entry should resume from the suspension point where we left off in the previous call. 

Amazingly, most common CPU architectures and OSs support multiple stack contexts for ***non-pre-emptive*** or cooperative threading via mechanisms called *fibers* (as opposed to the ***pre-emptive*** threads). The [**Boost.Coroutine2**](https://www.boost.org/doc/libs/1_70_0/libs/coroutine2/doc/html/index.html) library provides cross-platform abstractions over the various architectures and OSs (using [Boost.Context](https://www.boost.org/doc/libs/1_70_0/libs/context/doc/html/context/overview.html) for the low-level platform-dependent parts).

> #### **Intermezzo**
> A few years ago I wrote an algorithm that was easiest to implement and maintain when expressed as a coroutine generator. Boost.Coroutine2 provided a reasonable interface for writing coroutines and was quite usable. My algorithm was a great success and another project in the company decided to adopt it due to its superior performance. However, that project used [emscripten](https://emscripten.org/) to compile the C++ code to JavaScript.  
Unfortunately, building Boost with emscripten was neigh impossible and specifically the platform dependent Boost.Context was not designed to be "ported" to JavaScript running in a browser. What is one to do?  
After a lot of soul and web searching, and not wanting to manually distribute the complex algorithm logic into an iterator/Range API, I found that Boost.ASIO has a [small, badly indexed, unnamed and well hidden "stackless" coroutine library](https://www.boost.org/doc/libs/1_70_0/doc/html/boost_asio/reference/coroutine.html). This [single header file "library"](https://github.com/chriskohlhoff/asio/blob/master/asio/include/asio/coroutine.hpp) is totally portable (maybe even C-compatible too). It also does not even depend on any other parts of Boost and can be easily used stand-alone (with minor edits).
After some tweaking of my own (portable) algorithm code, the algorithm could compile and ran successfully as compiled client-side browser JavaScript.

So how does it this mysterious ["ASIO.coroutine"](https://github.com/chriskohlhoff/asio/blob/master/asio/include/asio/coroutine.hpp) work?  
Chris Kohlhoff, the author of ASIO, describes it in a blog post from 2010: ["A potted guide to stackless coroutines"](http://blog.think-async.com/2010/03/potted-guide-to-stackless-coroutines.html). Go and read that post and prepare to have your mind blown. Using a clever combination of macros and switch statements, Kohlhoff manages to introduce new "pseudo-keywords" like "yield" that can be used for yielding values from a coroutine. 🤯

From the user side, declaring a coroutine quite simple (quoting the post):

>Every coroutine needs to store its current state somewhere. For that we have a class called coroutine:
>
>```cpp
>class coroutine
>{
>public:
>  coroutine();
>  bool is_child() const;
>  bool is_parent() const;
>  bool is_complete() const;
>};
>```
>Coroutines are copy-constructible and assignable, and the space overhead is a single `int`. They can be used as a base class:
>
>```cpp
>class session : coroutine { ... };
>```
>or as a data member: 
>
>```cpp
>class session 
>{
>  ...
>  coroutine coro_;
>};
>```
> It doesn't really matter as long as you maintain a copy of the object for as long as you want to keep the coroutine alive.

Using it to yield values:

> The `yield return <expression>`  is often used in generators or coroutine-based parsers. For example, the function object:
>
>```cpp
>struct interleave : coroutine
>{
>  istream& is1;
>  istream& is2;
>  char operator()(char c)
>  {
>    reenter (this) for (;;)
>    {
>      yield return is1.get();
>      yield return is2.get();
>    }
>  }
>};
>```
> defines a trivial coroutine that interleaves the characters from two input streams.
>
> This type of yield divides into three logical steps:
>
>1. `yield` saves the current state of the coroutine.
>2. The resume point is defined immediately following the semicolon.
>3. The value of the expression is returned from the function.

An instance of the `interleave` class has an overloaded function call operator `()` and will yield interleaved `char`s, one by one, when called consecutively.

So the syntax is not perfect (note the `reenter (this)` above) but it is totally portable and it works great for generators.

Despite its coolness, we will not be using "ASIO.coroutine" because *there is a new kid on the block!*

## The Future is Now

<p align="center"><img src="../../assets/brain.png" width="50px"/></p>

### C++20 Coroutines

**Coroutines will become a language level facility in the C++20 standard!**  


## Caveats

This is a motivational and introductory post about generators. It focuses on how to write generators from a coroutine user point of view. However, there are many other details in the presented building blocks it does not go into at all:

- It does not do justice to the elegance and beauty of C++ Ranges.
- It ignores many aspects of coroutines including how the compiler generates this magic, how to write low-level coroutine types, asynchronous coroutines with the `co_await` keyword and many other wonderful features.




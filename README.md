# Wrapping a constructor with a function
[source](http://coliru.stacked-crooked.com/a/f584d543f2221a22)
```c++
#include <iostream>
#include <type_traits>
#include <utility>

// Let's say we want to implement our own pair type (akin to std::pair). We can
// do so using the simple Pair class template below. A Pair<A, B> is a pair of
// an element of type A and an element of type B. Pair's constructor perfect
// forwards its arguments to its two members.
template <typename A, typename B>
class Pair {
 public:
  template <typename T, typename U>
  Pair(T&& a, U&& b) : a_(std::forward<T>(a)), b_(std::forward<U>(b)) {}

 private:
  A a_;
  B b_;
};

// Since Pair is a class template, we have to explicitly instantiate all type
// parameters when constructing an instance of Pair. For example, the code
// `Pair p(1, 2)` will not compile, but the code `Pair<int, int>` will. To get
// around this inconvenience, we'll write a MakePair function (akin to
// std::make_pair) which will perfectly forward its arguments to the Pair
// constructor. We'll see below that this function isn't quite right.
template <typename A, typename B>
Pair<A, B> MakePair(A&& a, B&& b) {
  return {std::forward<A>(a), std::forward<B>(b)};
}

// Later, we'll see that this implementation of MakeDecayedPair solves the
// problem with the MakePair implementation above.
template <typename A, typename B>
Pair<typename std::decay<A>::type, typename std::decay<B>::type>
MakeDecayedPair(A&& a, B&& b) {
  return {std::forward<A>(a), std::forward<B>(b)};
}

// type<T>() will pretty print the type of T.
template <typename T>
void type() {
  std::cout << __PRETTY_FUNCTION__ << std::endl;
}

int main() {
  {
    // First, let's take a look at the return type of calling std::make_pair.
    // If we call std::make_pair with two rvalues, the returned type is
    // std::pair<int, int>. If we call std::make_pair with two lvalues, the
    // return type is still std::pair<int, int>. This is exactly what we
    // expect.
    int x = 2;
    int y = 2;
    type<decltype(std::make_pair(1, 1))>();  // std::pair<int, int>
    type<decltype(std::make_pair(x, y))>();  // std::pair<int, int>
  }

  {
    // Next, let's take a look at the return type of calling MakePair. Calling
    // MakePair with two rvalues works as expected; we get back a Pair<int,
    // int>. However, when we call MakePair with two rvalues, the return type
    // is Pair<int&, int&>! This is not what we want. Turns out the A and B in
    // MakePair are being deduced to int& and int&. These two types are then fed
    // into the return type Pair<A, B> of MakePair.
    int x = 2;
    int y = 2;
    type<decltype(MakePair(1, 1))>();  // Pair<int, int>
    type<decltype(MakePair(x, y))>();  // Pair<int &, int &>
  }

  {
    // We can fix this problem by using std::decay, as shown above in the
    // implementation of MakeDecayedPair. std::decay will turn all references
    // into their non-reference counterparts (and do other things too). In
    // fact, this is exactly (well almost exactly) what std::pair does! See [1]
    // for more information.
    //
    // [1]: http://en.cppreference.com/w/cpp/utility/pair/make_pair
    int x = 2;
    int y = 2;
    type<decltype(MakeDecayedPair(1, 1))>();  // Pair<int, int>
    type<decltype(MakeDecayedPair(x, y))>();  // Pair<int, int>
  }
}
```

# Lambda capture gotcha
Pop quiz, hotshot. What does [the following
program](http://coliru.stacked-crooked.com/a/d9d88d4ff22dd0ea) print?

```c++
#include <iostream>

int main() {
  auto const_thunk = [](const std::string& x) { return [&x] { return x; }; };
  auto foo = const_thunk("foo");
  std::cout << foo() << std::endl;
}
```

Surprise, the answer is the program invokes undefined behavior! Here's the
relevant snippet from
[cppreference](http://en.cppreference.com/w/cpp/language/lambda):

> If an entity is captured by reference, implicitly or explicitly, and the
> function call operator of the closure object is invoked after the entity's
> lifetime has ended, undefined behavior occurs. The C++ closures do not extend
> the lifetimes of the captured references.

In much the same way you shouldn't write a class like this:

```c++
class Foo {
  public:
    Foo(const std::string& x) : x_(x) {}
    void Print() { std::cout << x_ << std::endl; }
  private:
    const std::string& x_;
};
```

because this can happen:

```c++
Foo foo("foo");
foo.Print(); // undefined behavior
```

you should be careful not to capture references in a lambda if the lambda will
outlive the thing being referenced.

# What is the keyword `typename` used for?
- http://stackoverflow.com/questions/610245/where-and-why-do-i-have-to-put-the-template-and-typename-keywords

# Metaprogramming
- http://www.oreilly.com/programming/free/files/practical-c-plus-plus-metaprogramming.pdf

# SFINAE
- http://eli.thegreenplace.net/2014/sfinae-and-enable_if/

# Variadic Templates
- http://eli.thegreenplace.net/2014/variadic-templates-in-c/

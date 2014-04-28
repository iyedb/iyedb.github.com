---
layout: post
title:  "C++11 move semantic simple example"
keywords: c++11, move semantic, rvalue reference
published: true
categories: cpp11 en
---
There is plenty of long, detailed articles online about C++11 r-value references,
move semantic and move constructors so I won't bother you going into so much
detail here. Instead I am going to discuss a simple example on how 
useful this "new" C++11 feature is and how it helps in writing better, more 
intuitive C++ code.


Suppose you are writing a function that processes a file to send it or parse it or 
whatever. Before C++11 one would declare a function prototype like so:

{% highlight c++ linenos %}
void do_something_with_file (std::ifstream& file);
.
.
std::ifstream file("/path/to/some/file"); // open the file here
do_something_with_file(file);
{% endhighlight %}

or like this

{% highlight c++ linenos %}
void do_something_with_file (std::string filename);
{% endhighlight %}

but you can't write something like this:

{% highlight c++ linenos %}
void do_something_with_file (std::ifstream file);
.
.
std::ifstream file("/path/to/some/file");
do_something_with_file(file);
{% endhighlight %}

This call won't compile _(line 5)_ because we can't copy `std::ifstream objects 
(There is copy because we are passing the argument by value).
The compiler will complain because the copy ctor of std::ifstream 
is marked as private in old C++ and 
is *implicitly deleted* as we will see later with C++11. And this is where the 
move semantic of C++11 comes into play. Now we can define our function in this way:
{% highlight c++ linenos %}
void do_something_with_file (std::ifstream && file) {
    std::string s;
    while (file) { // uses bool operator () of std::ifstream 
        std::getline(file, s);
        std::cout << s << "\n";
    }
}
{% endhighlight %}

Notice the syntax used to declare the function parameter file: the *double &*.
It means that the function takes a so called _r-value reference_. 
Now the function is called this way:

{% highlight c++ linenos %}
do_something_with_file(std::ifstream("file.txt"));
{% endhighlight %}

This code creates a temporary object `std::ifstream("file.txt")` directly at call site. This temporary 
is used to create the function paramater using the _move 
ctor_ of `std::ifstream` and not the copy ctor as it is now deleted. The move ctor actually takes 
ownership of the resources held by the temporary. Moreover, we no longer need to declare 
an `std::ifstream` object to open the file as a first step, then pass it by reference to our 
function which feels awkward. Now the code feels more natural and straightforward: the function take a 
file then we just call it passing it a file argument.
Of course you still can pass an l-value as argument to the function if 
you absolutely have to, using the [std::move](http://en.cppreference.com/w/cpp/utility/move) 
utiliy:

{% highlight c++ linenos %}
std::ifstream file("/path/to/some/file");
do_something_with_file(std::move(file));
{% endhighlight %}

But this defeats the point the post ;)




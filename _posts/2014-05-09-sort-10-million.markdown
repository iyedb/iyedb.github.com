---
layout: post
title:  "Sorting 10 million integers"
keywords: algorithms, complexity
published: false
categories: algorithms
---
We know from CS classes that a good sorting algorithm have a time
complexity of `n*log(n)` whereas a naive insertion sort algorithm have
an `n^2` time complexity where n is the number of elements
in the data set. Algorithmic complexity expressed in this way allows us to reason
about the performance of algorithms and to compare them but how does it
translate to actual processing time. Or put another way, how much an `n^2` algorithm
is worse than an `n*log(n)` algorithm in terms of execution time. Let's find out!

To run the experiment we need a sufficiently big data set. I used this Python
script to create it:
{% highlight python %}
#!/usr/bin/env python
import random
random.seed()
def genl (max):
    cur = 0
    while cur <= max:
        yield random.randint(0, 1000000000)
        cur = cur + 1
gen = genl(10000000)
gen.send(None)
with open("dataset", "w") as f:
    for i in gen:
        f.write(str(i) + '\n')
{% endhighlight %}

The script generates 10 million random integers between 0 and 1 billion
and writes them to a text file that will be used later as input.

For the `n*log2(n)`, I used a basic quick sort algorithm C++ implementation:

{% highlight c++ %}
void quick_sort(int arr[], int begin, int end)
{
  //The partitionning function
  static const auto partition =
  [](int arr[], int begin, int end)
  {
    int pivot_index = end;
    int high_index = begin;
    int i = begin;

    for (i = begin; i < end; ++i) {
      if(arr[i] < arr[pivot_index]) {
        std::swap(arr[i], arr[high_index]);
        high_index++;
      }
    }
    std::swap(arr[pivot_index], arr[high_index]);
    return high_index;
  };

  if (end > begin) {
    auto pivot_index = partition (arr, begin, end);
    quick_sort (arr, begin, pivot_index - 1);
    quick_sort (arr, pivot_index + 1, end);
  }
}
{% endhighlight %}

Quick sort has a complexity of `n*log2(n)` **on average** but if you are very
unlucky it runs in `n^2` time depending on your dataset: if the pivot element
ends up in the first slot of every sub-array.

For the `n^2` algorithm, I used a simple insertion sort algorithm:

{% highlight c++ %}
void insertion_sort(int arr[], int len)
{
  for (int i = 1; i < len; ++i) {
    int j = i;
    while (j > 0 && arr[j] < arr[j - 1]) {
      std::swap (arr[j], arr[j - 1]);
      --j;
    }
  }
}
{% endhighlight %}

First, I launched the quick sort algorithm. Sorting the 10 million integers took
2,520 seconds on my laptop; a dual core 1,8 GHz with 8GB of RAM. Under **3 seconds**
to sort 10 million integers. That was quick!

Then I launched the insertion sort algorithm. After 10 minutes, the algorithm was still
running. I expected it to take substantially longer to run than quick sort but the
problem is that I had no idea on how long it would take to complete (It's not like
I sort 10 million integers everyday right). What's more
frustrating than a long running program? A program you can't tell how long it is.
But how long it takes to complete? Let's do the math.
Quick sort took 2.520 seconds to sort the 10 million integers, executing
10'000'000 * log2(10'000'000) operations (operation is compare and swap). That's
232'530'000 operations in 2,520 seconds or 1 operation every 10 nanosecond.
Given that insertion sort performs 10'000'000^2 or 100'000 billion operations,
it would have completed in **277 hours**.
That's approximatively **12 days**! A ridiculously long time
especially compared to the 2,5 seconds quick sort took to sort the same dataset.
That's how an `n^2` algorithm compares to an `n*log2(n)` in terms of execution time.

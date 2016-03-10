---
layout: post
title: "Project Euler Problem 6: Sum Square Difference"
---

Let's solve problem six in Project Euler in a programming language of your choice. Yes, that's correct, you can choose!

### Spoiler alert

If you have an academic interest in Project Euler, and you have not solved problem six, I suggest you do so before reading this post lest you deprive yourself of an opportunity to learn.

## The problem

This is one of the easiest Project Euler problems.

> Find the difference between the sum of the squares of the first one hundred natural numbers and the square of the sum.

Up to $$n = 100$$ it is trivial, but we can use [math][1] to solve it up to massive numbers, like [a thousand quinquagintaquadringentillion][2] (that's $$10^{2703}$$, or 1 followed by 2703 zeros), that could otherwise take years to solve even for a computer.

## Linear solution

Sometimes the most obvious solution isn't necessarily the best one. Let's consider the problem linearly.

$$
(1 + 2 + 3 + \cdots + 100)^2 + (1^2 + 2^2 + 3^2 + \cdots + 100^2)
$$

We can easily solve this in Ruby.

{% highlight ruby %}
(1..100).reduce(:+)**2 - (1..100).map { |n| n**2 }.reduce(:+)
{% endhighlight %}

For small numbers, this works well. But what if, instead of 100, you had to solve for a thousand quinquagintaquadringentillion? No one wants to create arrays with that many elements or loops with that many iterations.

## A better solution

With the help of elementary math--specifically, mathematical induction--we can solve this problem up to any number.

This is the square of the sum up to $$n$$.

$$
(1 + 2 + 3 + \cdots + n)^2 = \left(\frac{n(n + 1)}{2}\right)^2
$$

And the sum of squares up to $$n$$.

$$
1^2 + 2^2 + 3^2 + \cdots + n^2 = \frac{n(n + 1)(2n + 1)}{6}
$$

And the solution up to $$n$$.

$$
\left(\frac{n(n + 1)}{2}\right)^2 - \frac{n(n + 1)(2n + 1)}{6}
$$

With simple math we can solve this problem up to nonsensical numbers in microseconds. Here's the solution in Ruby; it's not terribly interesting.

{% highlight ruby %}
(n * (n + 1) / 2)**2 - (n * (n + 1)) * ((n * 2) + 1) / 6
{% endhighlight %}

I was curious, so I created [benchmarks][3] to illustrate the efficiency of four different techniques: Using map and reduce, enumerators, while loop, and induction. I tested the earlier three against $$n = 10^6$$, and the latter against a thousand quinquagintaquadringentillion, or $$10^{2703}$$.

{% highlight ruby %}
def map_reduce(n)
  (1..n).reduce(:+)**2 - (1..n).map { |n| n**2 }.reduce(:+)
end

def enum(n)
  sum_of_squares = square_of_sum = 0
  n.downto 1 do |i|
    square_of_sum += i; sum_of_squares += i * i
  end
  (square_of_sum *= square_of_sum) - sum_of_squares
end

def while_loop(n)
  sum_of_squares = square_of_sum = 0
  while n > 0
    square_of_sum += n; sum_of_squares += n * n; n -= 1
  end
  (square_of_sum *= square_of_sum) - sum_of_squares
end
{% endhighlight %}

{% highlight text %}
Benchmarking with 10000000 iterations
       user     system      total        real
MapReduce  2.890000   0.030000   2.920000 (  2.928501)
Enum       1.610000   0.010000   1.620000 (  1.611081)
Loop       1.200000   0.000000   1.200000 (  1.201676)

Benchmarking with 10^2703 iterations
Induction  0.000000   0.000000   0.000000 (  0.000010)
{% endhighlight %}

Map and reduce, enumerator, and the while loop took 2.9, 1.6, and 1.2 seconds respectively. Using our equation we solved a ridiculously larger number in 10 microseconds, which is 10 millionth of a second. I ran these on my first gen MacBook Pro Retina.

## Linear solutions in Swift and JavaScript

Predictably, solving this linearly in JavaScript is not very efficient for large numbers either. On my computer, using map and reduce fills up the stack and throws an exception at $$n \sim 130000$$, while using a loop solves up to $$n = 10^9$$ in circa 3.5 seconds.

{% highlight javascript %}
function diff(n) {
  var sumOfSquares = squareOfSum = 0;

  while(n) {
    sumOfSquares += n * n, squareOfSum += n, n--;
  }

  squareOfSum *= squareOfSum;
  return squareOfSum - sumOfSquares;
}
{% endhighlight %}

Swift didn't fair much better.

{% highlight swift %}
func diff(n: Int) -> Int {
    let array = [Int](1...n)

    let sumOfSquares = array.map({ $0 * $0 }).reduce(0, +)
    let sum = array.reduce(0, +)
    let squareOfSum = sum * sum

    return squareOfSum - sumOfSquares
}
{% endhighlight %}

You can find the [sample code][3] including benchmarks on Github.

[1]: http://en.wikipedia.org/wiki/Mathematical_induction
[2]: http://en.wikipedia.org/wiki/Names_of_large_numbers
[3]: https://gist.github.com/abitdodgy/b88a8018527107eb25c9

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>

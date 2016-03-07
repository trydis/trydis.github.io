---
layout: post
title: Elixir - First steps
description: Elixir string calculator
tags: elixir functional
---

## Intro

The majority of the code I've written over the years has involved C#, Swift, Objective-C or Javascript. Along the way I've been reading quite a lot about, as well as dabbled in F#. Heck, I even went to a Clojure session at [Devvox Belgium 2015](https://www.devoxx.be/).

Back in January I attended [NDC London](http://ndc-london.com/). Long story short, I ended up in [Rob Conery](https://twitter.com/robconery)'s session "[Three and a half ways Elixir changed me (and other hyperbole)](http://ndc-london.com/talk/tba-elixir-and-data/)". The session was interesting but it wasn't until this week I started looking more closely at the language.

* I read/skimmed through Elixir's "[getting started guide](http://elixir-lang.org/getting-started/introduction.html)".
* I watched the "[Meet Elixir](https://www.pluralsight.com/courses/meet-elixir)" course on Pluralsight.

## Writing my first Elixir code

The beginner part of the [String Calculator kata](http://osherove.com/tdd-kata-1/) seemed like a suitable starting point.

### Creating the project

[Mix](http://elixir-lang.org/getting-started/mix-otp/introduction-to-mix.html) is used, among other things, to create, compile and test the code, so I ran the following command:

```bash
$ mix new string_calculator
```

This created the directory `string_calculator` as well as the following files inside it:

```bash
* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/string_calculator.ex
* creating test
* creating test/test_helper.exs
* creating test/string_calculator_test.exs
```

### Writing some code

I opened `test/string_calculator_test.exs` and wrote my first test:

```elixir
test "empty string returns 0" do
  assert StringCalculator.add("") == 0
end
```

The simplest implementation I could think of was:

```elixir
def add(numbers) do
  0
end
```

The test(s) are executed by running `mix test` from the command line.

On to the second test:

```elixir
test "handles one number" do
  assert StringCalculator.add("1") == 1
end
```

Elixir functions are identified by name/arity (number of arguments) and [pattern matching](http://elixir-lang.org/getting-started/pattern-matching.html) is done based on the passed arguments, when attempting to find a suitable function to invoke.

I also remembered reading about named functions supporting both `do:` and `do/end` block syntax [here](http://elixir-lang.org/getting-started/modules.html#named-functions).

Based on this, my implementation ended up like this:

```elixir
def add(""), do: 0

def add(numbers) do
  String.to_integer(numbers) 
end
```

This was confusing and awesome at the same time, considering the programming languages I'm used to.

* The first one will be matched for empty strings.
* The second will be matched for non-empty strings.

Let that sink in and let's continue with the third test:

```elixir
test "handles two numbers" do
  assert StringCalculator.add("1,2") == 3
end
```

I've been using [LINQ](https://msdn.microsoft.com/en-us/library/bb397933.aspx) as well as [lambda expressions](https://msdn.microsoft.com/en-us/library/bb397687.aspx) extensively in C# over the years, so the anonymous functions felt natural to me. Since I've also dabbled in F#, I quickly remembered the [pipe operator](http://elixir-lang.org/getting-started/enumerables-and-streams.html#the-pipe-operator) in Elixir.

This was my first attempt:

```elixir
def add(numbers) do
  numbers
  |> String.split(",")
  |> Enum.map(fn(x) -> String.to_integer(x) end)
  |> Enum.reduce(fn(x, acc) -> x + acc end)
end
```

Not too bad, but I had a feeling that the [map function](http://elixir-lang.org/docs/stable/elixir/Enum.html#map/2) could be removed. Quite right, there's [reduce/3](http://elixir-lang.org/docs/stable/elixir/Enum.html#reduce/3) which makes it possible to set the initial value of the accumulator:

```elixir
def add(numbers) do
  numbers
  |> String.split(",")
  |> Enum.reduce(0, fn(x, acc) -> String.to_integer(x) + acc end)
end
```

For reference, this is the complete implementation so far:

```elixir
defmodule StringCalculator do
  def add(""), do: 0

  def add(numbers) do
    numbers
    |> String.split(",")
    |> Enum.reduce(0, fn(x, acc) -> String.to_integer(x) + acc end)
  end
end
```

On to the next requirement:

```elixir
test "handles more than two numbers" do
  assert StringCalculator.add("1,2,3,4,5") == 15
end
```

The implementation already handled this requirement, so I continued with the next one:

```elixir
test "handles newlines between numbers" do
  assert StringCalculator.add("1\n2,3") == 6
end
```

I looked at the documentation for [String.split/1](http://elixir-lang.org/docs/stable/elixir/String.html#split/1) and found this:

> The pattern can be a string, **a list of strings** or a regular expression.

The implementation required a small change to make the new test pass:

```elixir
def add(numbers) do
  numbers
  |> String.split([",", "\n"])
  |> Enum.reduce(0, fn(x, acc) -> String.to_integer(x) + acc end)
end
```

Moving on to the next requirement:

```elixir
test "handles custom delimiters" do
  assert StringCalculator.add("//;\n1;2") == 3
end
```

This is what I came up with, excluding the function that handles empty strings:

```exixir
def add("//" <> rest) do
  [delimiter, numbers] = String.split(rest, "\n")
  add(numbers, [delimiter])
end

def add(numbers) do
  add(numbers, [",", "\n"])
end

defp add(numbers, delimiters) do
  numbers
  |> String.split(delimiters)
  |> Enum.reduce(0, fn(x, acc) -> String.to_integer(x) + acc end)
end
```

I knew string concatenation was done with `<>`, but [this](http://stackoverflow.com/a/25897316/328194) Stack Overflow answer made me realize I could use it like in the first function above.

I also made the last function private by using `defp` instead of `def`. There should be no `add/2` function available to the consumer.

Another thing to note is that **the order of the functions matter**, when it's looking for functions to match. So if we for instance swap the order of the first two functions above, the code will break. This is because `add(numbers)` will match all non-empty strings that are passed as an argument.

On to the final test. First I had to figure out how to test for errors. In the docs I found [assert_raise/3](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.Assertions.html#assert_raise/3):

```elixir
test "throws error when passed negative numbers" do
  assert_raise ArgumentError, "-1, -3", fn ->
    StringCalculator.add("-1,2,-3")
  end
end
```

My initial attempt:

```elixir
defp add(numbers, delimiters) do
  numbers
  |> String.split(delimiters)
  |> check_for_negatives
  |> Enum.reduce(0, fn(x, acc) -> String.to_integer(x) + acc end)
end

defp check_for_negatives(numbers) do
  negatives = Enum.filter(numbers, fn(x) -> String.to_integer(x) < 0 end)
  if length(negatives) > 0, do: raise ArgumentError, message: Enum.join(negatives, ", ") 
  numbers
end
```

It converted the strings to integers twice, which I wasn't happy with, so I ended up with this:

```elixir
defp add(numbers, delimiters) do
  numbers
  |> String.split(delimiters)
  |> Enum.map(fn(x) -> String.to_integer(x) end)
  |> check_for_negatives
  |> Enum.reduce(0, fn(x, acc) -> x + acc end)
end

defp check_for_negatives(numbers) do
  negatives = Enum.filter(numbers, fn(x) -> x < 0 end)
  if length(negatives) > 0, do: raise ArgumentError, message: Enum.join(negatives, ", ")
  numbers
end
```

I also remembered reading about [partial application]((http://elixir-lang.org/crash-course.html#partials-in-elixir)) along the way, which I'd seen in F#.

> Elixir supports partial application of functions which can be used to define anonymous functions in a concise way

I decided to give it a go and my string calculator ended up like this:

```elixir
defmodule StringCalculator do
  def add(""), do: 0

  def add("//" <> rest) do
    [delimiter, numbers] = String.split(rest, "\n")
    add(numbers, [delimiter])
  end

  def add(numbers) do
    add(numbers, [",", "\n"])
  end

  defp add(numbers, delimiters) do
    numbers
    |> String.split(delimiters)
    |> Enum.map(&String.to_integer(&1))
    |> check_for_negatives
    |> Enum.reduce(0, &(&1 + &2))
  end

  defp check_for_negatives(numbers) do
    negatives = Enum.filter(numbers, &(&1 < 0))
    if length(negatives) > 0, do: raise ArgumentError, message: Enum.join(negatives, ", ")
    numbers
  end
end
```

Looking forward to learning more about Elixir!

## Source code

The source code can be found [here](https://github.com/trydis/elixir-string-calculator).
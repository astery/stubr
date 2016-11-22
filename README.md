# Stubr

![Build Status](https://travis-ci.org/leighshepperson/stubr.svg?branch=master)

In functional languages you should write pure functions. However, sometimes we need functions to call external API’s that affect the state of the system. So these functions are impure. In non-functional languages you create mocks to test expectations. For example, you might create a mock of a repository and the test checks it calls the save function. You are testing a side effect. This is something you should avoid in functional languages.

Instead of mocks we should use stubs. Mocking frameworks tend to treat them as interchangeable and this makes it hard to tell them apart. So it is good to have a simple definition. [Quoting](http://martinfowler.com/articles/mocksArentStubs.html) Martin Fowler:

* Stubs provide canned answers to calls made during the test, usually not responding at all to anything outside what's programmed in for the test. Stubs may also record information about calls, such as an email gateway stub that remembers the messages it 'sent', or maybe only how many messages it 'sent'.
* Mocks are objects pre-programmed with expectations which form a specification of the calls they are expected to receive.

So what does Stubr provide:

* Stubr is not a mock framework
* Stubr provides canned answers to calls made during a test
* Stubr makes it easy to create stubs
* Stubr makes sure the module you stub HAS the function you want to stub
* Stubr stubs as many functions and patterns as you want
* Stubr works without an explicit module. You set it up how you want
* Stubr lets you do asynchronous tests
* Stubr won't redefine your modules!
* Stubr auto-stubs non-stubbed functions
* Stubr records call information
* Stubr has ZERO dependencies

## Example - Random numbers

```elixir
    describe "stub :rand" do
      test "can stub :rand.uniform/1" do
        rand_stub = Stubr.stub!([uniform: fn _ -> 1 end], module: :rand)

        assert rand_stub.uniform(4) == 1
      end
    end

```

## Example - Timex

```elixir
describe "stub Timex" do
  test "can stub Timex.now/0" do
    fixed_time = Timex.to_datetime({2999, 12, 30})

    timex_stub = Stubr.stub!([now: fn -> fixed_time end], module: Timex)

    assert timex_stub.now == fixed_time
  end

  test "can stub Timex.now/0 and defer to un-stubbed Timex functions" do
    fixed_time = Timex.to_datetime({2999, 12, 30})

    timex_stub = Stubr.stub!([now: fn -> fixed_time end], module: Timex, auto_stub: true)

    assert timex_stub.before?(fixed_time, timex_stub.shift(fixed_time, days: 1))
  end
end

```

## Example - HTTPoison

```elixir
describe "stub HTTPoison" do
  setup do
    http_poison_stub = Stubr.stub!([
      get: fn("www.google.com") -> {:ok, %HTTPoison.Response{body: "search", status_code: 200}} end,
      get!: fn("www.nasa.com") -> %HTTPoison.Response{body: "space", status_code: 500} end,
      post: fn("www.nasa.com", "content") -> {:error, %HTTPoison.Error{id: nil, reason: :econnrefused}} end
    ], module: HTTPoison)

    [stub: http_poison_stub]
  end

  test "can stub HTTPoison.get/1", context do
    {:ok, response} = context[:stub].get("www.google.com")

    assert response.body == "search"
    assert response.status_code == 200
  end

  test "can stub HTTPoison.get!/1", context do
    response = context[:stub].get!("www.nasa.com")

    assert response.body == "space"
    assert response.status_code == 500
  end

  test "can stub HTTPoison.post/1", context do
    {:error, error} = context[:stub].post("www.nasa.com", "content")

    assert error.id == nil
    assert error.reason == :econnrefused
  end

end

```

## Example - Creating Complex Stubs

Create stubs with functions of different arity, argument patterns and names:

```elixir
stubbed = Stubr.stub!([
  gravitational_acceleration: fn(:earth) -> 9.8 end,
  gravitational_acceleration: fn(:mars) -> 3.7 end,
  gravitational_acceleration: fn(:earth, :amsterdam) -> 9.813 end,
  gravitational_acceleration: fn(:earth, :havana) -> 9.788  end,
  gravitational_attraction: fn(m1, m2, r) -> 6.674e-11 * (m1 * m2) / (r * r) end
])

assert stubbed.gravitational_acceleration(:earth) == 9.8
assert stubbed.gravitational_acceleration(:mars) == 3.7
assert stubbed.gravitational_acceleration(:earth, :amsterdam) == 9.813
assert stubbed.gravitational_acceleration(:earth, :havana) == 9.788
assert stubbed.gravitational_attraction(5.97e24, 1.99e30, 1.5e11) == 3.523960986666667e22
```

## Example - Auto-Stub

Auto-stub modules by setting the `auto_stub` option to true. In this case, if you have not provided a function to stub, it will defer to the original implementation:

```elixir
stubbed = Stubr.stub!([
  ceil: fn 0.8 -> :stubbed_return end,
  parse: fn _ -> :stubbed_return end,
  round: fn(_, 1) -> :stubbed_return end,
  round: fn(1, 2) -> :stubbed_return end
], module: Float, auto_stub: true)

assert stubbed.ceil(0.8) == :stubbed_return
assert stubbed.parse("0.3") == :stubbed_return
assert stubbed.round(8, 1) == :stubbed_return
assert stubbed.round(1, 2) == :stubbed_return
assert stubbed.round(1.2) == 1
assert stubbed.round(1.324, 2) == 1.32
assert stubbed.ceil(1.2) == 2
assert stubbed.ceil(1.2345, 2) == 1.24
assert stubbed.to_string(2.3) == "2.3"
```

## Example - Call Information

If the `call_info` option is true, then you can use the function `Stubr.call_info` to get call information about the stub:

```elixir
stubbed = Stubr.stub!([
  ceil: fn 0.8 -> :stubbed_return end,
  parse: fn _ -> :stubbed_return end,
  round: fn(_, 1) -> :stubbed_return end,
  round: fn(1, 2) -> :stubbed_return end
], module: Float, auto_stub: true, call_info: true)

stubbed.ceil(0.8)
stubbed.parse("0.3")
stubbed.round(8, 1)
stubbed.round(1, 2)
stubbed.round(1.2)
stubbed.round(1.324, 2)
stubbed.ceil(1.2)
stubbed.ceil(1.2345, 2)
stubbed.to_string(2.3)

assert Stubr.call_info(stubbed, :ceil) == [
  %{input: [0.8], output: :stubbed_return},
  %{input: [1.2], output: 2.0},
  %{input: [1.2345, 2], output: 1.24}
]

assert Stubr.call_info(stubbed, :parse) == [
  %{input: ["0.3"], output: :stubbed_return}
]

assert Stubr.call_info(stubbed, :round) == [
  %{input: [8, 1], output: :stubbed_return},
  %{input: [1, 2], output: :stubbed_return},
  %{input: [1.2], output: 1.0},
  %{input: [1.324, 2], output: 1.32}
]

assert Stubr.call_info(stubbed, :to_string) == [
  %{input: [2.3], output: "2.3"}
]
```

## Example - Behaviours

In [http://blog.plataformatec.com.br/2015/10/mocks-and-explicit-contracts/](http://blog.plataformatec.com.br/2015/10/mocks-and-explicit-contracts/) José Valim writes about finding "proper contracts and proper boundaries between the components" in your system. This is something Stubr can help you with to accomplish a "behaviour first" version of TDD.

For example, let's say you want to call an internal API and you decide to use an adapter to wrap it up. However, you haven't built the adapter yet, so you stub it out as an injected dependency without worrying about the implementation details.

So we might have something like this:

```elixir
adapter_stub = Stubr.stub!([
  get: fn("url") -> {:ok, "result"} end,
  put: fn("url", "data") -> {:ok} end
])

# calls the get and put functions on the adapter stub
foo(adapter_stub)
```

Immediately, this suggests that the adapter needs a behaviour that looks something like this:

```elixir
defmodule AdapterBehaviour do
  @callback get(url :: String.t) :: {:ok, String.t}
  @callback put(url :: String.t, data :: String.t) :: {:ok}
end
```

So we can "clip" this on to the stub using the behaviour option:

```elixir
adapter_stub = Stubr.stub!([
  get: fn("url") -> {:ok, "result"} end,
  put: fn("url", "data") -> {:ok} end
], behaviour: AdapterBehaviour)
```

If the generated stub does not implement the behaviour, then it throws the usual compiler warning.

After we've completed all our tests, we can now implement the adapter and ensure it implements the `AdapterBehaviour`:

```elixir
defmodule Adapter do
  @behaviour AdapterBehaviour

  def get(url), do: #call the internal API
  def put(url, data), do: #call the internal API
end
```

The good thing about this is:

If you change the `Adapter`, then you should change the `AdapterBehaviour`. Since this will cause a compile time warning for the stub, it will encourage you to fix it, making your stubs and unit tests much less brittle.

## Links

How stubs can be used in TDD for functional languages: [https://www.infoq.com/presentations/mock-fsharp-tdd](https://www.infoq.com/presentations/mock-fsharp-tdd)

Mark Seemann's [blog post](http://blog.ploeh.dk/2013/10/23/mocks-for-commands-stubs-for-queries/) talks about the difference between Mocks and Stubs in the context of commands and queries.

Examples for Stubr: [Examples](https://github.com/leighshepperson/stubr_examples)

## Roadmap

* Return different values for different calls of the same function
* "Was called with" functions
* "Was called with matching pattern" functions
* Callbacks

## Installation

Stubr is [available in Hex](https://hex.pm/packages/stubr), the package can be installed as:

Add `stubr` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [{:stubr, "~> 1.3.5", only: :test}]
end
```

# Not-so-senior gotcha: #1

> Really, this happened

<small>Originally posted on [substack](https://zoten.substack.com/p/not-so-senior-gotcha-1) on Dec 20, 2021</small>

I’m so happy I can already start this feature today!

This is honestly the kind of article I would have liked to read more, before being hit by impostor syndrome so much.

So, here’s the thing: I was working on a small feature in a big piece of code with a lot of other gotchas, and couldn’t get this little `GenServer` to start properly. The only thing it was doing? Getting a couple of items in the form `{key, %MyStruct{}}` from the database, and sorting them out by date.

The ordering function passed to [`Enum.sort/2`](https://hexdocs.pm/elixir/1.14.3/Enum.html#sort/2) was just something like

``` elixir
defp order_by_created_at(
    {_key, %MyStruct{created_at: created_atA}},
    {_key, %MyStruct{created_at: created_atB}}),
    do: created_atA < created_atB}
```

Simple, effective. Except, it was not working.

``` bash
no function clause matching in …
```

Wait, what? The printed error (with the real data passed to the function) was exactly what I was expecting.

Well, the error is simply that, even if I am not using them - by placing an underscore `_` before the name -, I’m claiming the keys are equals, by using `_key` in both arguments of the function.

Both

``` elixir
# different unused variables names
{_key0, %MyStruct{created_at: created_atA}}, {_key1, %MyStruct{created_at: created_atB}}
```

and

``` elixir
# generic `_` has a special treatment
{_, %MyStruct{created_at: created_atA}}, {_, %MyStruct{created_at: created_atB}}
```

would have worked as I meant them to.

Ten years of IT studies, ten years of professional programming, three years of Elixir programming, one and a half of using it professionally, and I just lost 20 minutes on this before asking for help to my colleagues and being pointed out.

It may happen to you, but at least I hope this will not happen anymore, now :)

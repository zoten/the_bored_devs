# Using my own Erlang

> asdf, Erlang and Elixir

<small>(originally posted in [Not So Senior](https://zoten.substack.com/p/using-my-own-erlang) on Nov 4, 2021)</small>

I don’t think I should recap what [asdf](https://github.com/asdf-vm/asdf) is, should I? There’s plenty of resources online on how to install it, set it up and make it work with all systems or languages, so let’s skip this.

Today, I was - technically I _currently am_, as I write this down - working on a problem on an Elixir application, involving a plethora of [mnesia](https://www.erlang.org/doc/man/mnesia.html) internal functions that I could not debug in a decent manner by my knowledge. For the ones who want a small context without descending in technical details, there was this infinite loop of logs like

``` bash
Mnesia(:"node-1@127.0.0.1"): ** ERROR ** Mnesia on :"node-1@127.0.0.1" could not connect to node(s) [:"node-1@127.0.0.1"] 
```

Well, awkward.

If someones like-minded will search for this in the future, I searched the web for all kinds of _mnesia node cannot connect to itself_ permutations, with no luck. As a result of failing to google my problem, to extract information from Erlang dumps, to read the code and other debugging strategies, I fell back to t he last hope: using my own Erlang/OTP (with blackjack and…well, _Magic: the Gathering_, probably).

If I’ll find an acceptable solution for this problem I’ll write something about it too, but now let’s see how embarrassingly easy has been to do it.

First of all, let’s locate the Erlang/OTP repository and clone it in our favorite repo directory

``` bash
git clone https://github.com/erlang/otp /path/to/otp
```

I’ll not repeat any of the instructions you’ll find in the repo for building it. Just see the `README.md` and `HOWTO/install.md` files included and do your

``` bash
make
```

After a bit of crunching you’ll find all your shining new executable files in `/path/to/otp/bin` folder.

Now, the asdf command is usually something like

``` bash
asdf local erlang path:</local/path/to/bin>
```

and this one of the only two gotchas I had to face. Taking a [look](https://github.com/asdf-vm/asdf-erlang/blob/e99a10615b047b963d3ce9ac28d84b2c9889c390/bin/install#L48) at the Erlang asdf plugin repo, you can see it is expecting install path to contain a bin folder, so it is enough to omit it from the path, like in

``` bash
asdf local erlang path:/path/to/otp
# and not asdf local erlang path:/path/to/otp/bin
```

After that, an

``` bash
asdf reshim
```

is what is needed to refresh paths on _asdf_’s side.

After that, you can make your own modifications to the OTP repo (e.g., in my case, a lot of logs :) ) and compile it again, and have no need to do any other reshim to have your brand new code up and running.

You don’t even have to

``` bash
mix compile
```

again your application, if like me you’re working on Elixir and not directly on Erlang. The new _erl_ executable will still be used.

Hope this helps someone jumping in this wonderful piece of code!

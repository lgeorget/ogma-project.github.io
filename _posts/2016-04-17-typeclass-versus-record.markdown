---
layout: post
title: "Typeclass versus Record"
modified:
categories: Ogma
excerpt:
tags: ["haskell"]
image:
  feature:
date: 2016-04-17T12:00:00+02:00
---

In the [ogmarkup library](https://github.com/ogma-project/ogmarkup), the output
generation is driven by a set of templates and functions. In the earlier
versions of ogmarkup, the configuration was the `GenConf` record (module
`Text.Ogmarkup.Private.Config`). However, a few days ago, we merged a [Pull
Request](https://github.com/ogma-project/ogmarkup/pull/35) in `dev` that changes
things a bit.

## Why Change `GenConf`?

Commit after commit, the definition of the `GenConf` record has grown in
complexity until it gets more than ten fields. That made the declaration of a
`GenConf` instance a little tricky. See for yourself:

{% highlight haskell %}
htmlConf :: GenConf Html
htmlConf =
  GenConf englishTypo
          (\doc -> [shamlet|<article>#{doc}|])
          id
          asideTemp
          (\paragraph -> [shamlet|<p>#{paragraph}|])
          id
          (\a dialogue -> [shamlet|$newline never
                                   <span .dialogue .#{a}>
                                     #{dialogue}|])
          (\a thought -> [shamlet|$newline never
                                  <span .thought .by-#{a}>
                                    #{thought}|])
          (\reply -> [shamlet|$newline never
                              <span .reply>
                                #{reply}|])
          (preEscapedToHtml ("</p><p>" :: Text))
          (\text -> [shamlet|$newline never
                             <em>#{text}|])
          (\text -> [shamlet|$newline never
                             <strong>#{text}|])
          auth
          htmlPrintSpace
{% endhighlight %}

It was a real mess. It was hard to write, hard to read and, as a consequence,
very error prone. In these conditions, GHC is not a great assistance to the
developers. Its error messages basically consist of a “too few arguments”
complaint.

## Switching To A Typeclass `GenConf`

A while ago, I read an [article on typeclass-based
libraries](http://www.yesodweb.com/blog/2016/03/why-i-prefer-typeclass-based-libraries).
We made the switch in the commit
[`888a86dc`](https://github.com/ogma-project/ogmarkup/pull/35/commits/888a86dc139da0a3f81f2fded66354ceec5cb4c1)

### Typeclass benefits

I do not regret our choice, for several reasons.  The `GenConf` typeclass
definition brings several default implementations. Another benefit rises when
the time comes to declare a new instance. You can override several
configuration functions without worrying about the declaration order. As a side effect,
the functions name is visible, so it improve the declaration readability.

We kept the record field names, so the ogmarkup library update was pretty
straightforward. I just updated the function signatures.

### Drawbacks

Yet, I found one major drawback to this change: `GenConf` is a multi-parameter
type class. You have to enable at least two GHC extensions:

* `FlexibleInstances`
* `MultiParamTypeClasses`

Without them, you won’t be able to define a new `GenConf` instance. I am not
very comfortable with the idea to oblige our potential future users to enable
specific GHC extensions. At least, we will have to make it very clear in the
documentation[^1].

[^1]: It is still not the case, but it *will*.

All in all, I think this update was a good idea, especially because we are not
immune to see the `GenConf` definition changes again in the future and becomes
even more complex.

## Side Note: Functional Dependency

Finally, let’s have a look at the typeclass definition.

{% highlight haskell %}
class (Monoid o) => GenConf c o | c -> o
{% endhighlight %}

I had already seen such strange declaration before, but I didn’t know what it
meant before. `c -> o` means if you know which type is `c`, you can guess `o`.
In other words, there is only one `o` for one `c`. It is called a functional
dependency (see `FunctionalDependencies` GHC extension). The main thing to
understand is that, without this dependency, for one `c`, GHC cannot guess which
`o` to choose.

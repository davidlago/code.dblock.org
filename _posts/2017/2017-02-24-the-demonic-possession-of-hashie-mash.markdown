---
layout: post
title: "The Demonic Possession of Hashie::Mash"
date: 2017-02-24
tags: [hashie, ruby]
comments: true
---
> I've been co-maintaining the Ruby library [hashie](https://github.com/intridea/hashie) for a few years now. It's a wonderful set of extensions written by some of the best Ruby developers. But sometimes, a piece of code turns rogue and starts hurting people. It becomes evil. In this post I will tell a story about how `Hashie::Mash` became the garbage can of recurrent problems that ranged from educational to psychotic.

The story of the `Hashie::Mash` class began like all useful classes. We decided that writing `hash[:key]` was annoying, and `hash['key']` was confusing. It was obvious that `hash.key` was better, that developers wanted `hash.key?` and even `hash.key = value`. We made that happen with some help from `method_missing`.

The invariant key access was borrowed from [Merb's Mash](https://github.com/wycats/merb-extlib/blob/master/lib/merb-extlib/mash.rb), which had the following comment on top of the file.

> This class has dubious semantics and we only have it so that people can write `params[:key]` instead of `params['key']`.

This warning was ignored, and became the first thing to be removed from `Hashie::Mash`. A lot of new features, such as method access, were added instead.

{% highlight ruby %}
def method_missing(method_name, *args)
  if (match = method_name.to_s.match(/(.*)=$/)) && args.size == 1
    self[match[1]] = args.first
  elsif (match = method_name.to_s.match(/(.*)\?$/)) && args.size == 0
    key?(match[1])
  elsif (match = method_name.to_s.match(/(.*)!$/)) && args.size == 0
    initializing_reader(match[1])
  elsif key?(method_name)
    self[method_name]
  elsif match = method_name.to_s.match(/^([a-z][a-z0-9A-Z_]+)$/)
    default(method_name)
  else
    super
  end
end

def default(key = nil)
  if key.is_a?(Symbol) && key?(key.to_s)
    self[key]
  else
    key ? super : super()
  end
end
{% endhighlight %}

The new `Hashie::Mash` was born as a pure example of Ruby metaprogramming. Some really smart people pitched in on `Mash`, including [@mbleigh](https://github.com/mbleigh) and [@hassox](https://github.com/hassox), authors of [Grape](https://github.com/ruby-grape/grape) and [Warden](https://github.com/wardencommunity/warden) respectively. We were winning!

There were only small challenges in the early days of `Mash`, such as [overwriting nil values](https://github.com/intridea/hashie/commit/751855c5a23cc32c92ea716e9b0d640684695fac), having to figure out how to [convert a `Mash` to `Hash`](https://github.com/intridea/hashie/commit/0c85e9482f0ca0232d7cf7cf6577a359ea07ffed), [preserving sub-mashes in conversion](https://github.com/intridea/hashie/commit/f670fb3a7bbfb971b06cdc415e76b091667f933d), or [making to work with `to_json`](https://github.com/intridea/hashie/commit/0c85e9482f0ca0232d7cf7cf6577a359ea07ffed). Nothing that couldn't be easily fixed with a bit of code. More features were added, including [shallow updates](https://github.com/intridea/hashie/commit/206136f89e8d01e8bc3f8564bf4251a6f6976d37) by [@mislav](https://github.com/mislav), author of [rbenv](https://github.com/rbenv/rbenv) and [will_paginate](https://github.com/mislav/will_paginate). Everybody who is who in the Ruby world wanted a bit of `Hashie::Mash` action!

But what could go wrong when you suddenly overrode `id`? Ruby 1.8 had an `Object#id` method that was called when the Mash didn't have an `id` key. This caused unexpected failures, since there was no id key-value, but Mash was still returning a value. Same went for `type`. So this change got undone in [#77](https://github.com/intridea/hashie/pull/77), with a nice warning in the documentation (that everybody naturally read).

> Do not use the following keys: `type`, `object_id`, and `id`. These are Ruby Object methods. Overriding them means you're going to have a Bad Time™.

We weren't having a Bad Time™ at all. Everyone had their favorite way of fetching values. Some wanted to write `hash[:key]`, others `hash.key` or even `hash.fetch(:key, 123)`. Unfortunately, it didn't always work as one might have expected.

{% highlight ruby %}
1.9.3p392 :001 > require 'hashie'
true
1.9.3p392 :002 > h = Hashie::Mash.new
{}
1.9.3p392 :003 > h[:key] = nil
nil
1.9.3p392 :004 > h.fetch(:key, 123)
123
{% endhighlight %}

Whoops. Fixed in [#93](https://github.com/intridea/hashie/issues/93).

Have we dealt with `respond_to?`? Probably not. You were in for a treat.

{% highlight ruby %}
def respond_to?(method_name, include_private=false)
  return true if key?(method_name) || method_name.to_s.slice(/[=?!_]\Z/)
  super
end
{% endhighlight %}

Unfortunately this broke [Rails 4 strong_parameters](http://edgeguides.rubyonrails.org/action_controller_overview.html#strong-parameters). Can you see why? This code always responded `true` to `permit?`. Take a look at the following example.

{% highlight ruby %}
settings = { foo: "1", attributes: { title: "Value" } }
record.attributes = settings.attributes # raises ActiveModel::ForbiddenAttributes
{% endhighlight %}

Whoops. When a key was a `Hash`, it was automatically converted into a `Mash`. The `Mash` instance responded to `:permit?`, thus it triggered the `ActiveModel` forbidden attribute check.

{% highlight ruby %}
module ForbiddenAttributesProtection
  def sanitize_for_mass_assignment(*options)
    new_attributes = options.first
    if !new_attributes.respond_to?(:permitted?) || new_attributes.permitted?
      super
    else
      raise ActiveModel::ForbiddenAttributes
    end
  end
end
{% endhighlight %}

Well, you couldn't have `permit?` work both ways. A workaround that broke everything was carelessly applied in [#104](https://github.com/intridea/hashie/pull/104), reverted, and an extension added in [#147](https://github.com/intridea/hashie/pull/147) by creating `Hashie::Extensions::Mash::ActiveModel`. For Rails users, [@MaximFilimonov](https://github.com/Maxim-Filimonov) even made [hashie_rails](https://rubygems.org/gems/hashie_rails) to inject the extension for scenarios in which Rails could not be auto-detected. We also had to worry about nested hashes, and made sure those worked in [#219](https://github.com/intridea/hashie/pull/219).

As another proof that you just cannot have nice things was `Mash#deep_merge`. It was being really slow, unnecessarily converting values. With a fix in [#107](https://github.com/intridea/hashie/pull/107), a `Hashie::Mash.new` on a big hash went from 1200 seconds to to less than 2 seconds. This wasn't the last time we faced serious performance problems. We were just warming up.

One day someone had a great idea to profile an application that relied on `Hashie::Mash` and discovered that we allocated a String, `(.*?)([?!=_]?)$`, 50,000 times. Was that string familiar? Fixed in [#221](https://github.com/intridea/hashie/pull/221).

At the same time Hashie was being refactored into separate extensions. Unfortunately they didn't play well with a backwards compatible `Mash` at all.

{% highlight ruby %}
require 'hashie'

class TestHash < Hashie::Trash
  include Hashie::Extensions::Coercion
  include Hashie::Extensions::MergeInitializer
  include Hashie::Extensions::Dash::IndifferentAccess
end

class Bar < TestHash
  property :a
end

class Foo < TestHash
  property :bar
  coerce_key :bar, Bar
end

test = Foo.new Hashie::Mash.new(bar: Hashie::Mash.new(a: 42))
p test
{% endhighlight %}

Yes, this is actual, broken code. It uses `Hashie::Mash`, three extensions and `Hashie::Trash` (at least that name accurately represented what the class was trying to be). Obviously, this caused a stack overflow. Fixed in [#164](https://github.com/intridea/hashie/issues/164).

Furthermore, when using an `ActiveRecord::HashWithIndifferentAccess` that was nested (i.e. `{a: {b: 1}}`) to create a `Hashie::Mash` with the `IndifferentAccess` extension, an `ArgumentException` was raised. This was because the `IndifferentAccess` method `convert_value` overrode `ActiveRecord::HashWithIndifferentAccess`'s method of the same name. Changing the method name in IndifferentAccess solved the issue in [#277](https://github.com/intridea/hashie/pull/277), until next time.

I once was using Hashie for a throwaway project, and noticed this incredibly unexpected and annoying behavior. With a normal Hash, you can specify a default value for newly-accessed keys.

{% highlight ruby %}
hsh = Hash.new{|h, k| h[k] = [] }
hsh[:hello] << 100
# => { :hello => [100] }
{% endhighlight %}

This was very useful for quick inject operations.

{% highlight ruby %}
collection.inject(Hash.new{|h,k| h[k] = [] }) do |h, x|
  h[x] << somefoo(x) if x > foo(x)
end
{% endhighlight %}

I wanted to do this with `Hashie::Mash`. It wasn't possible, sorry.

{% highlight ruby %}
hsh = Hashie::Mash.new{|h, k| h[k] = [] }
hsh[:hello] << 100
# => {"hello"=>[]}
{% endhighlight %}

Fixed in [#259](https://github.com/intridea/hashie/pull/259).

Did you know that `Hashie::Mash` translated all keys into strings? A brave man, [@michaelherold](https://github.com/michaelherold) fixed this in [#296](https://github.com/intridea/hashie/pull/296). We have seen him a lot more since, as he has then become the defacto primary maintainer of Hashie, and the most active contributor to this craziness. In that PR he left the following note.

> There is one interesting thing that pops out from this. Checking for equality between Mashes (and Hashes with Mashes) now fails unless the keys are equivalent. For example: `Hashie::Mash.new(p: 'test') == Hashie::Mash.new('p' => 'test') #=> false`.

This is pretty awesome, as both instances behaved the exact same in every possible way, but weren't equal.

Also, a surprisingly high number of libraries out there `require 'hashie/mash'` and not `hashie`. What could possible go wrong?

```
Bundler::GemRequireError:
There was an error while trying to load the gem 'omniauth-oauth2'.
Gem Load Error is: uninitialized constant
Hashie::Extensions::RubyVersionCheck::ClassMethods::RubyVersion
```

This happened to omniauth via [#391](https://github.com/intridea/hashie/issues/391), [#392](https://github.com/intridea/hashie/pull/392) fixed this and was released as 3.5.1. It happened again in [#401](https://github.com/intridea/hashie/issues/401), fixed in [#402](https://github.com/intridea/hashie/pull/402), this time with rather complicated integration tests.

One of the most common issues we have reported was when a `Hashie::Mash` attempted to override a built-in method and failed to do so. To prevent this from being an issue in the future, [@michaelherold](https://github.com/michaelherold) added a logging layer accessible at `Hashie.logger` that was user-configurable. This should have helped report the error to the user before it became a problem in [#381](https://github.com/intridea/hashie/pull/381).

What could possibly go wrong? In [elasticsearch-ruby#398](https://github.com/elastic/elasticsearch-ruby/issues/398), we saw this.

> I updated our bundle today which included hashie@3.5.1, and found our logs are now completely overwhelmed with warnings. I believe this is being caused by the `sort` key that comes back in the response body.

That was nothing, we were just warming up.

> Our log rotation was not aggressive enough for the flood of this warning, ultimately taking our servers down.

Yes, `sort` was a function. And `Hashie::Mash` was used as a "freeform" wrapper around deeply structured Hashes coming from Elasticsearch, enabling method access to the said structures.

Connecting the logger to `Rails.logger` and adding `Hashie::Mash.disable_warnings` was surprisingly tedious in [#400](https://github.com/intridea/hashie/pull/400), and was released as 3.5.2.

What could possibly go wrong? In [#407](https://github.com/intridea/hashie/issues/407) we read the following.

```
bundler/runtime.rb:94:in `rescue in block (2 levels) in require':
There was an error while trying to load the gem 'omniauth-twitter'.
(Bundler::GemRequireError)
Gem Load Error is: private method `warn' called for nil:NilClass
Backtrace for gem load error is:
hashie/mash.rb:334:in `log_built_in_message'
```

So we broke omniauth adapters in two different ways, a new record. Fixed in [#406](https://github.com/intridea/hashie/pull/406) by making sure `Hashie::Mash.disable_warnings` carried into subclasses, and released as 3.5.3.

But wait. Omniauth has not finished suffering. In [#410](https://github.com/intridea/hashie/issues/410) we found out that there was a performance regression on `Mash`'s method accessors. We switched `include?` to `respond_to?` to lookup method collisions in [#411](https://github.com/intridea/hashie/pull/411) and the problem went away. Released as 3.5.4.

What could possibly go wrong? In [#413](https://github.com/intridea/hashie/issues/413) we discovered that we could no longer assign a value to a key more than once.

{% highlight ruby %}
foo = Hashie::Mash.new

foo['foobar'] = Array.new
foo['foobar'] = Array.new
{% endhighlight %}

This was pretty trivial stuff, right? Well, that broke.

```
# hashie/mash.rb:344:in `method': undefined method `foobar' for class `Hashie::Mash' (NameError)
  from hashie/mash.rb:344:in `log_built_in_message'
  from hashie/mash.rb:151:in `custom_writer'
  from bug.rb:6:in `<main>'
```

The issue was that we changed the code that logged warnings to use `respond_to?` to check if the call to `Hashie::Mash[]=` overrode a method that existed on `Hashie::Mash`, or the class that was subclassing it. The documentation for `Object#respond_to?` said that if the method was not defined, `respond_to_missing?` method was called and the result was returned. Unfortunately `Hashie::Mash` overrode `respond_to_missing?` to return true if a key with that name existed in the hash. In the previous version this worked because `methods.include?` excluded the "magical" methods that `Hashie::Mash` would have responded to.

Again, [@michaelherold](https://github.com/michaelherold) fixed this in [#415](https://github.com/intridea/hashie/pull/415) by avoiding logging multiple times, released as 3.5.5.

We couldn't wait long enough for the next issue and 3.5.6!

The good news was that we built an [integration test suite](https://github.com/intridea/hashie/tree/master/spec/integration) for any bizarre regression. And a huge community of people were around to help. I also suggested everyone stop using `Hashie::Mash` and mix-in well-defined and clearly scoped extensions, one-by-one, such as `Hashie::Extensions::MethodAccess` or `Hashie::Extensions::MergeInitializer` on a need-by-need basis.

**Update:** There were many new, and more subtle and serious problems with `Hashie::Mash` since I wrote this blog post. [@michaelherold](https://github.com/michaelherold) gave a wonderful talk at RubyConf 2018, ["Let's Subclass Hash"](https://michaeljherold.com/articles/rubyconf-2018-lets-subclass-hash/). I highly recommend you watch it!

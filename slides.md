---
theme: default
highlighter: shiki
lineNumbers: false
info: |
  # Threads, callbacks, and execution context in Ruby

  When you provide a block to a function in Ruby, do you know when and where that block will be executed? What is safe to do inside the block, and what is dangerous? Let’s take a look at various code examples and understand what dragons are hidden in Ruby dungeons.
drawings:
  persist: false
fonts:
  provider: none
  fallback: false
  local: Martian Grotesk, Martian Mono
  sans: Martian Grotesk
  serif: Martian Grotesk
  mono: Martian Mono
aspectRatio: '4/3'

transition: slide-left
title: Threads, callbacks, and execution context in Ruby
mdc: true
---

# Threads, callbacks, and <small class="text-90%">execution context in Ruby</small>

<div class="absolute bottom-0 left-0 w-full px-10 py-8 grid grid-cols-2 justify-items-stretch items-end gap-4">
  <div class="text-left text-xl">
    Andrey Novikov, Evil Martians<br />
    <small><a href="https://regional.rubykaigi.org/osaka03/">Osaka Ruby Kaigi #03</a></small><br />
    <small><time datetime="2023-09-09">09 September 2023</time></small>
  </div>

  <div class="w-28 h-28 scaled-image justify-self-end">
    <a href="https://evilmartians.com/"><img alt="Evil Martians" src="/images/01_Evil-Martians_Logo_v2.1_RGB.svg" class="block dark:hidden" /><img alt="Evil Martians" src="/images/02_Evil-Martians_Logo_v2.1_RGB_for-Dark-BG.svg" class="hidden dark:block" /></a>
  </div>
</div>

<div class="absolute top-0 left-0 w-full scaled-image h-36 p-4 text-center">
<a href="https://regional.rubykaigi.org/osaka03/" class="" style="filter: drop-shadow(0 0 0.5rem gray);"><img alt="Osaka Ruby Kaigi #03" src="/images/osakarubykaigi-logo.svg" class="block mx-auto" /></a>
</div>


<style>
  a {
    border-bottom: none !important;
  }
</style>

<!-- 皆さん、こんにちは！ -->

---
layout: image-right
image: ./images/20230305_193526.jpg
class: annotated-list
---

## About me

Hi, I'm Andrey (アンドレイ){class="text-xl"}

- Back-end engineer at Evil Martians

- Writing Ruby, Go, and whatever

  SQL, Dockerfiles, TypeScript, bash…

- Love open-source software

  Created and maintaining a few Ruby gems

- Living in Japan for 1 year already

- Driving a moped

  And also a bicycle to get kids to kindergarten

<img src="/images/01_Evil-Martians_Logo_Lurkers_v2.0_on-Transparent.png" class="absolute bottom-0 max-w-80% scaled-image" />

<!--
はじめまして、アンドレイと申します。もう一年以上大阪の近くに住んでいます。
-->

---
layout: image-right
image: ./images/osaka-venue-martian-office-map.png
---

## Martians are closer than you think

<div class="text-xl mt-10">

Our base is just 30 minutes walk away from here!

Please come visit us! [^1]

</div>

![Evil Martians logo](/images/01_Evil-Martians_Logo_v2.1_RGB.svg)

[^1]: Just let us know in advance in e𝕏-Twitter [@evilmartians_jp](https://twitter.com/evilmartians_jp)

<!--

「イービル・マーシャンズ」という会社に勤めていますから、俺は邪悪な火星人です。

火星からよろしく。

-->

---

## Let's talk about **blocks** as callbacks

<div class="text-2xl mt-10">

```ruby
3.times do |i|
  puts "Hello, Osaka RubyKaigi #0#{i}!"
end
```

It feels like code between `do` and `end` is in the same flow with surrounding code, right?
</div>

<v-click>
<div class="text-center text-5xl mt-32">

**WRONG!**

</div>
</v-click>

<!--
では、Rubyのブロックについて話しましょう。

たとえば、timesというメソッドのループは、他のプログラミング言語にあるforループと似たようなものだと思っている人もいるかもしれませんが。→

違います!
-->

---

## Blocks are separate pieces of code

<div class="text-2xl mt-10">

Block is separate entity, that is passed to `times` method as an argument and got **called** by it.

```ruby
greet = proc do |i|
  puts "Hello, Osaka RubyKaigi #0#{i}!"
end

greet.call(3)

# also
3.times(&greet)
```
</div>

<!--
ブロックは独立なオブジェクトです。変数に保存して、後でメソッドと同じように呼び出すことができます。
-->

---

## Blocks are called by methods

![](/images/Ruby_under_a_microscope_calling_a_block.png)


Illustration from the [Ruby under a microscope](https://patshaughnessy.net/ruby-under-a-microscope) book (日本語版: [Rubyのしくみ](https://amzn.asia/d/cYL93aH))


<qr-code url="https://patshaughnessy.net/ruby-under-a-microscope" caption="Ruby under a microscope" class="w-36 absolute bottom-10px right-10px" />

<!--

実際、timesの例では、ブロックはメソッドの最後の引数になって、メソッドから呼び出されます。

-->

---
layout: statement
---


# Blocks ARE callbacks

We often use blocks as callbacks to _hook_ our own behavior for someone's else code.


<!--
ですから、ブロックはほとんどいつもコールバックとして使われています。

ブロックはイコール　コールバックです。
-->

---

# Blocks in time and space

<div class="text-2xl">

```ruby
def some_method(&block)
  block.call
  # and/or
  @callbacks << block
end

some_method do
  puts "Hey, I was called!"
end

# which one??? will it be called at all?
```

 - When does the block get executed?
 - And where?
 - How to understand?
</div>

<!--
問題は、そのブロックがいつ、どこで呼び出されるか？

ブロックを受け入れるメソッドを見れば、それを理解できるのでしょうか？
-->

---
layout: statement
---

# How to understand?

When and where will the block be called?

<v-click>
<div class="text-center text-5xl mt-20 mb-8">

No way to know! 😭

</div>

Well, except reading method documentation and source code.

And memorize, memorize, and memorize.
</v-click>


<!--
おそらく、分かる方法は、→　覚えるしかありません。
-->

---

## Blocks right here, right now

<div class="text-2xl mt-10">

All `Enumerable` methods are _sync_, and will call provided block during their execution.

```ruby
3.times do |i|
  puts "Hello, Osaka RubyKaigi #0#{i}!"
end
```

E.g. `times` will `yield` to the block on every iteration.

</div>

<!--
Rubyの標準ライブラリの多くのメソッドは、実行中にすぐブロックを呼び出しますが。
-->

---

## Blocks can be called later

<div class="text-2xl mt-10">

Much later.

```ruby
after_commit do
  puts "Hello, Osaka RubyKaigi #03!"
end
```

ActiveRecord callbacks and also `after_commit` from [after_commit_everywhere](https://github.com/Envek/after_commit_everywhere) gem will store callback proc in ActiveRecord internals for later.

<a href="https://github.com/Envek/after_commit_everywhere"><img alt="after_commit_everywhere" src="/images/og-after_commit_everywhere.png" class="absolute bottom-0 scaled-image max-w-50%" /></a>

<qr-code url="https://github.com/Envek/after_commit_everywhere" caption="after_commit_everywhere" class="w-36 absolute bottom-10px right-10px" />

</div>

<!--
フレームワークとジェムのメソッドは普段、ブロックをどこかに保存して、あとで呼び出せます。
-->

---

## Blocks can be called from other threads

<div class="text-2xl mt-10">

```ruby {1-14|1,5|3,8-10|4,9,12,14|1-14}
result = []

work = proc do |arg|
  # Can you tell which thread is executing me?
  result << arg # I'm closure, I can access result
end

Thread.new do
  work.call "from new thread"
end

work.call "from main thread"

# And guess what's inside result now? 🫠
```

Can you feel how thread-safety problems are coming?
</div>

<!--
それから、→

ブロックはクロージャであるため、スコープ内のローカル変数にアクセスできます。→

さらに、ブロックを定義するスレッドと実行するスレッドは絶対同じスレッドではありません。→

ブロックはいつ、どのスレッドで、実行されるか、知ることができないんです。→
-->

---

## Different threads

E.g. concurrent-ruby `Promise` uses **thread pools** to execute blocks.

Thread pools doesn't guarantee which thread will execute which block.

```ruby {1-20|2,8|1-5|7-11|13-16|3,9,18-20}
work1 = proc do
  Thread.current[:state] ||= 'work1'
  raise "Unexpected!" if Thread.current[:state] != 'work1'
  "result"
end

work2 = proc do
  Thread.current[:state] ||= 'work2'
  raise "Unexpected!" if Thread.current[:state] != 'work2'
  "result"
end

promises = 100.times.flat_map do
  Concurrent::Promise.execute(&work1)
  Concurrent::Promise.execute(&work2)
end

Concurrent::Promise.zip(*promises).value!
#=> Unexpected! (RuntimeError) 💣💥
```

<!--
Thread.currentというHashをご存知でしょうか？→

これは、Rubyのスレッドローカル変数です。→

二つのブロックは、→ 同じThread.currentのキーを使用することを想像しましょう。→

スレッドプールを使うと、ブロックを実行するスレッドを選ぶことができないため、→

遅かれ早かれ、Thread.currentには思いがけないデータを発見します。
-->

---

## Example: NATS client

<div class="text-xl mt-10">

NATS is a modern, simple, secure and performant message communications system for microservice world.

```ruby
nats = NATS.connect("demo.nats.io")

nats.subscribe("service") do |msg|
  Thread.current[:counter] ||= 0
  Thread.current[:counter] += 1
  msg.respond(Thread.current[:counter])
end
```

Prior version 2.3.0 every subscription was executed in its own thread.

Code above works as expected.
</div>

<a href="https://github.com/nats-io/nats-pure.rb"><img alt="nats-pure.rb" src="/images/og-nats-pure.png" class="absolute bottom-0 scaled-image max-w-50%" /></a>

<qr-code url="https://github.com/nats-io/nats-pure.rb" caption="nats-pure gem" class="w-36 absolute bottom-10px right-10px" />


<!-- 

実際の例はNATSというメッセージングシステムのRubyクライアントです。

NATSは各サブスクリプションに対応するスレッドを作成します。

ということで、渡されたブロックがいつも同じスレッドで実行されますので、このスライドのコードは問題ないです。

-->

---
layout: image
image: /images/nats-pure-thread-pool-pull-request.png
---

<!-- 

ですが、数万のサブスクリプションがある場合、性能が悪くなります。

ここで俺は最適化を行い、代わりに固定サイズのスレッドプールを導入しました。

性能がずいぶん良くなりましたが、副作用が出てきました。
-->

---

## Can I use `Thread.current` in NATS callbacks?

<div class="text-2xl mt-10">

Performance is got much better, but there is a side effect…

```ruby
nats = NATS.connect("demo.nats.io")

nats.subscribe("service") do |msg|
  Thread.current[:counter] ||= 0
  Thread.current[:counter] += 1
  msg.respond(Thread.current[:counter])
end
```
</div>

<div class="text-2xl mt-10 mb-8">

Q: So, can I?

A: Not in 2.3.0+! 🤯
</div>

<div class="text-2xl mt-10 mb-8">

**Hint**: better not to anyway!

</div>

<!--
今日リリースされた2.3.0バージョンから、NATSのブロックではThread.currentが使えなくなりました。ブロックは毎回異なるスレッドで実行されますので。
-->

---

## Where you can find thread pools?

<div class="mt-15 text-3xl">

- Puma
- Sidekiq
- ActiveRecord `load_async`
- NATS client (new!)
- …and many more

</div>
<div class="mt-15 text-xl">

Good thing is that you don't have to care about them most of the time.

**Pro Tip:** In Rails use `ActiveSupport::CurrentAttributes` instead of `Thread.current` as every request is going to be executed in different thread!

</div>

<!--

NATSだけじゃなくて、実際のアプリでは、Pumaも、Sidekiqもスレッドプールを使用しています。

ということで、各ウェブリクエストと各ジョブは異なるスレッドで実行されますよ！

-->

---

# Recap

<div class="mt-15 text-2xl">

- Blocks are primarily used as callbacks
- Blocks can be called from other threads
- And this thread can be different each time!
  - Think twice before using `Thread.current`

<hr class="my-15" />

**And you have to remember that!** {class="text-3xl"}

</div>

<!--
要すると:

- ブロックはコールバックです

- ブロックは異なるスレッドで実行される可能性があります。

- 「Thread.current」を使用する際には気をつけてください。
-->

---

## Let's talk more about blocks and callbacks!

<div class="text-center text-2xl mt-40 scaled-image">

Attend my next talk about Rails Executor at

<a href="https://kaigionrails.org/2023/"><img alt="Kaigi on Rails 2023" src="/images/symbol_kaigionrails2023.svg" class="mx-auto my-5 text-5xl" /></a>

Tokyo, 27–28 October 2023

</div>

<qr-code url="https://evilmartians.com/events/rails-executor-kaigionrails" caption="Rails Executor talk announce" class="w-60 absolute bottom-10px right-10px" />

<!--
以上です！

来月もブロックについて話します。東京のKaigi on Railsにぜひ参加してください。
-->

---

# Thank you!

<Transform :scale="1.5">

<div class="grid grid-cols-[8rem_3fr_4fr] mt-12 gap-2">

<div class="justify-self-start">
<img alt="Andrey Novikov" src="https://secure.gravatar.com/avatar/d0e95abdd0aed671ebd0920c16d393d4?s=512" class="w-32 h-32 scaled-image" />
</div>

<ul class="list-none">
<li><a href="https://github.com/Envek"><logos-github-icon class="dark:invert" /> @Envek</a></li>
<li><a href="https://twitter.com/Envek"><logos-twitter /> @Envek</a></li>
<li><a href="https://facebook.com/Envek"><logos-facebook /> @Envek</a></li>
<li><a href="https://t.me/envek"><logos-telegram /> @Envek</a></li>
</ul>

<div>
<qr-code url="https://github.com/Envek" caption="github.com/Envek" class="w-32 mt-2" />
</div>

<div class="justify-self-start">
<a href="https://evilmartians.com/"><img alt="Evil Martians" src="/images/01_Evil-Martians_Logo_v2.1_RGB.svg" class="w-32 h-32 scaled-image block dark:hidden" /><img alt="Evil Martians" src="/images/02_Evil-Martians_Logo_v2.1_RGB_for-Dark-BG.svg" class="w-32 h-32 scaled-image hidden dark:block" /></a>
</div>

<div>

- <logos-github-icon class="dark:invert" /> [@evilmartians](https://github.com/evilmartians?utm_source=osakarubykaigi&utm_medium=slides&utm_campaign=threads-callbacks)
- <logos-twitter /> [@evilmartians_jp](https://twitter.com/evilmartians_jp/?utm_source=osakarubykaigi&utm_medium=slides&utm_campaign=threads-callbacks)
- <logos-linkedin-icon /> [@evil-martians](https://www.linkedin.com/company/evil-martians/?utm_source=osakarubykaigi&utm_medium=slides&utm_campaign=threads-callbacks)
- <logos-instagram-icon class="dark:invert" /> [@evil.martians](https://www.instagram.com/evil.martians/?utm_source=osakarubykaigi&utm_medium=slides&utm_campaign=threads-callbacks)
</div>

<div>
<qr-code url="https://evilmartians.jp/" caption="evilmartians.jp" class="w-32 mt-2" />
</div>

<div class="col-span-3">

Our awesome blog: [evilmartians.com/chronicles](https://evilmartians.com/chronicles/?utm_source=osakarubykaigi&utm_medium=slides&utm_campaign=threads-callbacks)!

<p class="text-sm">See these slides at <a href="https://envek.github.io/osakarubykaigi-threads-callbacks/">envek.github.io/osakarubykaigi-threads-callbacks</a></p>

</div>
</div>

</Transform>

<style>
  ul a { border-bottom: none !important; }
  ul { list-style-type: none !important; }
  ul li { margin-left: 0; padding-left: 0; }
</style>

<!--

最後までご視聴してくださって、ありがとうございました！

我が社のブログでは、Rubyについての記事がたくさんあります。ぜひお読みください！日本語の翻訳もあります。

-->

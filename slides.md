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
  <div class="text-left">
    Andrey Novikov, Evil Martians<br />
    <small><a href="https://owddm.com/">Osaka Ruby Kaigi #03</a></small><br />
    <small><time datetime="2023-07-22">09 September 2023</time></small>
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

<!-- 皆さん、こんにちは！　今日、Rubyのブロックについて話しませんか？ -->

---
layout: image-right
image: /images/20230305_193526.jpg
class: annotated-list
---

# About me

Hi, I'm Andrey

- Back-end engineer at Evil Martians

- Writing Ruby, Go, and whatever

  SQL, Dockerfiles, TypeScript, bash…

- Love open-source software

  Created and maintaining a few Ruby gems

- Living in Japan for 1 year already

- Driving a moped

  And also a bicycle to get kids to kindergarten

<img src="images/01_Evil-Martians_Logo_Lurkers_v2.0_on-Transparent.png" class="absolute bottom-0 max-w-80% scaled-image" />

<!--
まず、自己紹介です。はじめまして、アンドレイと申します。もう一年以上家族と一緒に大阪の近くに住んでいて、原付を乗っています。Rubyなどで開発しています。

それに俺は火星人です。邪悪な火星人です。ですが、我々は、平和目的で地球に来ました。
-->

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

![Evil Martians logo](images/01_Evil-Martians_Logo_v2.1_RGB.svg)

[^1]: Just let us know in advance in e𝕏-Twitter [@evilmartians_jp](https://twitter.com/evilmartians_jp)

<!--

真面目に言うと、「イービル・マーシャンズ」という会社に勤めています。

我々はスタートアップや大企業のためにプロジェクトを開発したり、コンサルティングしたりしています。バックエンドをもちろん、フロントエンドやデザインも含めてプロダクトをターンキー開発しています。

それに我々の日本基地、オフィスは、ここから歩いて３０分ぐらいのところにあります。江戸堀です。

よろしければ、ぜひ遊びに来てください！　ただ、予めツイッターで連絡してくださいね。元もテレワークしていますので、いつもオフィスにいるわけではありません。

-->

---

## Let's talk about callbacks…

<div class="text-2xl mt-10">

What callbacks?

```ruby
class User < ApplicationRecord
  after_create :send_welcome_email
end
```

<v-click>

No, not Rails callbacks,<br />no-no-no!

<iframe src="https://giphy.com/embed/12XMGIWtrHBl5e" class="absolute bottom-5 right-0 w-50% h-50%" frameBorder="0" allowFullScreen></iframe>
</v-click>
</div>

<!-- 
さっそくですが、今日の話題に入りましょう。コールバックについて話しましょう。

どんなコールバックと言うと…

！

いやいや、今日はRubyKaigiですよ、RailsKaigiではない、ActiveRecordのコールバックについて一切話しませんよ！　そしてこれは自体が熱い話題ですね。スキップ!

-->

---

## Let's talk about **blocks** as callbacks

<div class="text-2xl mt-10">

```ruby {1-3|1|2|1|2|1|2|1-3}
3.times do |i|
  puts "Hello, Osaka RubyKaigi #0#{i}!"
end
```

It feels like code between `do` and `end` is following the same flow with surrounding code, right?
</div>

<v-click class="text-center text-5xl mt-20">

**WRONG!**

</v-click>

<!--
代わりに、純粋な Ruby ブロックについて話しましょう。

ご存知とおり、Rubyのブロックは、`do` と `end` で囲まれたコードの塊です。

ブロックのコードは周囲のコードと同じ流れをたどっているような感じがありますよね？

！！！！！！！！

ほかのプログラミング言語にあるforループと同じようなものだと思っている人もいるかもしれません。

！

でも、それは間違いです！
-->

---

## Blocks are separate pieces of code

<div class="text-2xl mt-10">

Block is separate entity, that is passed to `times` method as an argument and got **called** by it.

```ruby
greet = proc do |i|
  puts "Hello, Osaka RubyKaigi #0#{i}!"
end

3.times { |i| greet.call(i) }
# or
3.times(&greet)
```
</div>

<!--
ブロックは周囲のコードから分離して、独立なものです。ブロックはプロックオブジェクトとして。変数に保存できたり、後でメソッドと同じように呼び出すことができます。
-->

---

## Blocks are called by methods

![](/images/Ruby_under_a_microscope_calling_a_block.png)


Illustration from the [Ruby under a microscope](https://patshaughnessy.net/ruby-under-a-microscope) book (日本語版: [Rubyのしくみ](https://amzn.asia/d/cYL93aH))


<qr-code url="https://patshaughnessy.net/ruby-under-a-microscope" caption="Ruby under a microscope" class="w-36 absolute bottom-10px right-10px" />

<!--

前のスライドのタイムズメソッドは、ブロックを引数として受け取って、繰り返しごとそのブロックを呼び出します。

ところで、「Rubyのしくみ」という本をお勧めできます、Rubyの内部の仕組みが詳しく書いてあるので、めっちゃおもしろいです。日本語版もあります。

-->

---
layout: statement
---


# Blocks ARE callbacks

We often use blocks as callbacks to _hook_ our own behavior for someone's else code.


<!--
ブロックは変数に保存したり、ブロックを呼び出すメソッドに引数として渡したり、このブロックは後で呼び出されるようになりますので、ブロックはほとんどいつもコールバックだと思います。ブロックはイコールコールバックです。
-->

---

# Blocks in time and space

<div class="text-2xl">

```ruby {1-11|7-9|1-5}
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
しかし、重要な問題は、そのブロックがいつ、どこで呼び出されるかということです。

ブロックを受け入れるメソッドの呼び出しを見れば、それを理解できるのでしょうか？
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
おそらく、理解方法はありません。メソッドのドキュメントを読んで、ソースコードを読んで、そして覚えるしかないみたいんです。
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
典型的なシナリオをいくつか見てみましょう。

たとえば、Rubyの標準ライブラリの多くのメソッドは、実行中にすぐにブロックを呼び出します。
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
フレームワークとジェムのコールバックは普段、ブロックをどこかに保存して、あとで呼び出します。

たとえば、ActiveRecordの after_commit コールバックは、データベーストランザクションのコミットが成功した後だけに呼び出されます。
-->

---

## Blocks can start new threads

<div class="text-2xl mt-10">

```ruby
Thread.new do
  puts "Hello from new thread!"
  # Common process memory can be accessed
end

puts "Hello from the main thread"
```

or even processes:

```ruby
Process.fork do
  puts "Hello from child process!"
  # Parent process memory can't be accessed
end

puts "Hello from parent process"
```

</div>

<!-- 特別な例はスレッドの作成とプロセスのフォークです。提供されたブロックは、新しく作成されたスレッドまたはプロセスへのエントリポイントとして使用されます。-->

---

## Blocks can be called from other threads

<div class="text-2xl mt-10">

```ruby {1-14|1,5|3,9|1,5,9|4,9,12|1-14}
result = []

work = proc do |arg|
  # Can you tell which thread is executing me?
  result << arg # I'm closure, I can do that!
end

Thread.new do
  work.call "new thread"
end

work.call "main thread"

# And guess what's inside result? 🫠
```

Can you feel how thread-safety problems are coming?
</div>

<!--
さて、話を少し複雑にしてみましょう。

！

Rubyブロックはクロージャであるため、スコープ内で以前に定義されたローカル変数にアクセスできます。

！

もちろん、以前に定義されたローカル変数に保存されたブロックも呼び出すことができます。

!

このようにして、Rubyブロックはさまざまなスレッドから共有グローバル状態にアクセスできます。

!

この同時に、ブロックはどのスレッドで実行されているかを知るわけありません。

!

スレッド安全性は相変わらず難しいです…
-->

---

## Different threads

E.g. concurrent-ruby `Promise` uses thread pools to execute blocks.

```ruby {1-20|2,8|1-5|7-11|13-16|18-20}
work1 = proc do
  Thread.current[:state] ||= 'work1'
  raise "Unexpected!" if Thread.current[:state] != 'work1'
  SecureRandom.hex(4)
end

work2 = proc do
  Thread.current[:state] ||= 'work2'
  raise "Unexpected!" if Thread.current[:state] != 'work2'
  SecureRandom.hex(8)
end

promises = 100.times.flat_map do
  Concurrent::Promise.execute(&work1)
  Concurrent::Promise.execute(&work2)
end

Concurrent::Promise.zip(*promises).value!
#=> Unexpected! (RuntimeError)
# But it also might be okay (chances are low though)
```

<!--
もっと難しい例を見てみましょう。→

Thread.currentというHashをご存知でしょうか？　これは、Rubyのスレッドローカル変数です。
これをブロックからの使用にはご注意してください！ →

いろいろなブロックは、 → 同じThread.currentのキーを使用して、 →

その同時に同じスレッドで実行されると、正しく動作しない恐れがあります。
-->

---

## Example: NATS client

<div class="text-2xl mt-10">

NATS is a modern, simple, secure and performant message communications system for microservice world.

```ruby
nats = NATS.connect("demo.nats.io")

nats.subscribe("service") do |msg|
  msg.respond("pong")
end
```

In current versions every subscription is executed in its own separate thread.
</div>

<a href="https://github.com/nats-io/nats-pure.rb"><img alt="nats-pure.rb" src="/images/og-nats-pure.png" class="absolute bottom-0 scaled-image max-w-50%" /></a>

<qr-code url="https://github.com/nats-io/nats-pure.rb" caption="nats-pure gem" class="w-36 absolute bottom-10px right-10px" />


<!-- 

実際の例、NATSというシステムのRubyクライアントを見てみましょう。

NATSは、マイクロサービス世界向けの、シンプル、安全かつパフォーマンスの高いメッセージ通信システムです。

Rubyクライアントの使用方法は簡単です。NATSに接続してから、クライアントインスタンスでsubscribeというメソッドを呼び出し、コールバックとしてブロックを提供すると、メッセージが到着するたびにブロックが呼び出されます。

このクライアントの中では、各サブスクリプションの受信メッセージを処理するコールバックを実行するためのスレッドが作成されます。１対１です。

-->

---
layout: image
image: /images/nats-pure-thread-pool-pull-request.png
---

<!-- 

唯一の問題は、数万のサブスクリプションがある場合、数万のスレッドではオペレーティング·システムのレベルでコンテキストの切り替えがボトルネックになるため、速度が低下することです。

そこで俺は最適化を行い、これらのスレッドをすべて削除し、固定サイズのスレッドプール内のわずか数十のスレッドに置き換えました。

これにより、多くのサブスクリプションを持つアプリケーションの処理が大幅に高速化されましたが、副作用もあります。
-->

---

## Can I use `Thread.current` in NATS callbacks?

<div class="text-2xl mt-10">

```ruby
nats = NATS.connect("demo.nats.io")

nats.subscribe("service") do |msg|
  Thread.current[:handled] ||= 0
  Thread.current[:handled] += 1
end
```
</div>

<div class="text-2xl mt-10 mb-8">

Q: So, can I?

A: It depends on gem version! 🤯
</div>

Hint: better not to anyway!

<!--
じゃあ、ブロックでThread.currentを使えるかどうか、どのようにりかいできますか？

知る分けない!　これはgemのバージョンによっても異なることがあります！
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

**Pro Tip:** In Rails use `ActiveSupport::CurrentAttributes` instead of `Thread.current`!

</div>

<!--

それに、典型的なウェブアプリでは、スレッドプールはいくつかあります。Pumaでも、Sidekiqでも、ActiveRecordの中にも。

幸いなことに、ほとんどの場合、それらを気にする必要はありません。

とにかく、Thread.currentを使わないほうが良いと思います。Railsの場合、ActiveSupport::CurrentAttributesを使うことをお勧めします。

-->

---

# Recap

<div class="mt-15 text-2xl">

- Blocks are primarily used as callbacks
- Blocks can be executed in a different threads
- And this thread can be different each time!
  - Think twice before using `Thread.current`
- Blocks can be executed with a different receiver

<hr class="my-15" />

**And you have to remember that!** {class="text-3xl"}

</div>

<!--
要約すると、次のようになります。

- ブロックはコールバックです

- ブロックは別のスレッドで実行可能

- 「Thread.current」を使用する前によく考えてください。
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
これでは以上です！

俺は今年の十月の下旬、東京にある「Kaigi on Rails」というカンファレンスに「Rails Executor」に関する発表します。コールバックについてさらに詳しく知りたい方はぜひ参加してください。
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

- <logos-github-icon class="dark:invert" /> [@evilmartians](https://github.com/evilmartians?utm_source=owddm&utm_medium=slides&utm_campaign=imgproxy-is-amazing)
- <logos-twitter /> [@evilmartians_jp](https://twitter.com/evilmartians_jp/?utm_source=owddm&utm_medium=slides&utm_campaign=imgproxy-is-amazing)
- <logos-linkedin-icon /> [@evil-martians](https://www.linkedin.com/company/evil-martians/?utm_source=owddm&utm_medium=slides&utm_campaign=imgproxy-is-amazing)
- <logos-instagram-icon class="dark:invert" /> [@evil.martians](https://www.instagram.com/evil.martians/?utm_source=owddm&utm_medium=slides&utm_campaign=imgproxy-is-amazing)
</div>

<div>
<qr-code url="https://evilmartians.jp/" caption="evilmartians.jp" class="w-32 mt-2" />
</div>

<div class="col-span-3">

Our awesome blog: [evilmartians.com/chronicles](https://evilmartians.com/chronicles/?utm_source=owddm&utm_medium=slides&utm_campaign=imgproxy-is-amazing)!

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

我が社のブログは、Rubyについての記事は、日本語のの翻訳もあります。ぜひお読みください！

最後までのご視聴してくださって、ありがとうございました！
-->

---
title: Синтаксис Clojure — префиксная запись
layout: post
date: 2017-09-19
category: dev
tags: [clojure syntax]
---
<center>
    <blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">I&#39;ve been thinking that I dislike parentheses and that that&#39;s one of the reasons why I love <a href="https://twitter.com/hashtag/clojure?src=hash">#clojure</a> <a href="https://t.co/runqGh4Z3E">pic.twitter.com/runqGh4Z3E</a></p>&mdash; Nicolás Berger (@nicoberger) <a href="https://twitter.com/nicoberger/status/874690048221995008">June 13, 2017</a></blockquote>
    <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
</center>

Несмотря на обилие круглых скобок, которое зачастую вызывает отторжение у программистов, язык Clojure имеет весьма простой синтаксис. И если присмотреться, то окажется, что и скобок не намного больше, чем в других языках, а иногда даже меньше. Сегодня мы поговорим о постфиксной записи повсеместно используемой в Clojure. Но сперва, давайте вспомним какие вообще бывают записи в программировании.

Привычная всем **инфиксная** запись: в ней операторы ставятся между операндов

~~~
2 + 2 * 2, object.method()
~~~

**Префиксная** запись (она же польская нотация): оператор стоит вначале, а затем перечисляются операнды

~~~
(+ (* 2 2) 2), ++i
~~~

При условии, что арность операторов известна заранее, скобки в первом выражении можно опустить

~~~
+ * 2 2 2
~~~

**Постфиксная** запись

~~~
i++
~~~

И, наконец, **функциональная** запись

~~~
cos(90), substr("spam!", 1, 3)
~~~

В таких языках как Ruby, Python и Java встречаются все 4 вида записей. Такое многообразие *может затруднять* чтение и понимание кода. К тому же, нужно запоминать приоритет выполнения операторов. В Clojure используется только префиксная запись. Сперва это непривычно и, возможно, даже «больно». *Но стоит помнить, что нечто болезненное может со временем оказаться полезным* [^1].

Для более наглядной картины, ниже, я привожу сравнительную таблицу вызовов функций.

| Выражение на Сlojure | Эквивалент на Java |
|----------------------|--------------------|
| `(not k)`       | `!k` |
| `(inc a)`       | `a++, ++a, a += 1, a + 1`  |
| `(/ (+ x y) 2)`  | `(x + y) / 2`  |
| `(instance? java.util.List al)`  | `al instanceof java.util.List` |
| `(if (not a) (inc b) (dec b))`   | `!a ? b + 1 : b - 1` |
| `(Math/pow 2 10)`  | `Math.pow(2, 10)` |
| `(.someMethod someObj "foo" (.otherMethod otherObj 0))`  | `someObj.SomeMethod("foo", otherObj.otherMethod(0))` |

С одной стороны, синтаксис вызовов на Java крайне многообразен:

- в нём есть инфиксные операторы, но любой нетривиальный код требует явно определять порядок их выполнения;
- унарные операторы имеют как префиксный так и постфиксный варианты;
- вызов статических методов сопровождается префиксом…;
- но в то же время вызов методов инстанса сопровождается множественными инфиксными формами: сначала указывается целевой объект (он будет присвоен переменной `this` внутри тела вызываемого метода), затем сам вызываемый метод с формальными параметрами.

С другой стороны, вызовы выражений на Clojure следуют единственному тпростому правилу: на первой позиции всегда стоит оператор (или функция), на остальных перечислены параметры передаваемые этому оператору. Инфиксная, функциональная и постфиксная записи, а также сложные правила определяющие приоритет вычисления — здесь отсутствуют. Такой синтаксис проще читать, использовать и изучать.

---
[^1]: Думай как математик. Как решать любые проблемы быстрее и эффективнее, Барбара Оакли. 2-е издание. — «Альпина Паблишер», 2016. — 132 c.
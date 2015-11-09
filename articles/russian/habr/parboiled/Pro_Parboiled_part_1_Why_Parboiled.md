Про Parboiled
=============
**Часть 1. Почему Parboiled?**

Сегодня, в свете бурного роста популярности функциональных языков программирования, всё чаще находят себе применение
комбинаторы парсеров — инструменты, облегчающие разбор текста простым смертным. Такие библиотеки, как [Parsec] (Haskell)
и [Planck] (OCaml) уже успели хорошо себя зарекомендовать в своих экосистемах. Их удобство и возможности в своё время
подтолкнули создателя языка Scala, Мартина Одерски, внести в стандартную библиотеку их аналог —
[Scala Parser Combinators][spc] (ныне вынесены в [scala-modules][sm]), а знание и умение пользоваться подобными
инструментами — отнести к обязательным требованиям к Scala-разработчикам [уровня A3][a3].

[Parsec]: https://wiki.haskell.org/Parsec
[Planck]: https://bitbucket.org/camlspotter/planck
[spc]: https://github.com/scala/scala-parser-combinators
[sm]:  http://mvnrepository.com/artifact/org.scala-lang.modules
[a3]:  http://www.scala-lang.org/old/node/8610

Эта серия статей посвящена библиотеке [Parboiled][pb] — мощной альтернативе и возможной замене для Scala Parser
Combinators. В ней мы подробно рассмотрим работу с текущей версией библиотеки — Parboiled2, а также уделим внимание
Parboiled1, так как большая часть существующего кода всё ещё использует именно её.

[pb]: https://github.com/sirthias/parboiled

**Структура цикла:**

 - Часть 1. Почему Parboiled?
 - Часть 2. Сопоставление текста
 - Часть 3. Извлечение данных
 - Часть 4. Суровая действительность

<cut text="Читать про ска́лу и парсеры →">


# Введение

Parboiled — библиотека, позволяющая с легкостью разбирать (парсить) языки разметки (такие как HTML, XML или JSON),
языки программирования, конфигурационные файлы, логи, текстовые протоколы и вообще что угодно текстовое. Parboiled
придётся весьма кстати, если вы захотите разработать свой предметно-ориентированный язык ([DSL][dsl]): с её помощью вы
сможете быстро получить [абстрактное синтаксическое дерево][ast] и, вспомнив паттерн [интерпретатор][ip], исполнять
команды вашего доменного языка.

[ast]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[ip]:  https://en.wikipedia.org/wiki/Interpreter_pattern
[dsl]: https://en.wikipedia.org/wiki/Domain-specific_language

На данный момент существует несколько версий данной библиотеки:

  - Parboiled for Java — самая первая версия библиотеки. Написана Маттиасом Доеницем (Matthias Doeniz)
    на Java и для Java. До сих пор пользуется популярностью, хоть и находится в состоянии «end of life». Если по воле
    случая она досталась вам в наследство, или же вы сознательно начинаете проект на Java, советую рассмотреть в
    качестве альтернативы [grappa][grappa] — форк Parboiled1, который старательно поддерживается в работоспособном
    состоянии пользователем с ником [fge][fge].

  - Parboiled — библиотека, теперь уже более известная как Parboiled1, появилась на свет после того, как
    Маттиас проникся скалой. Он сделал Scala-фронтэнд для Parboiled, заодно забросив поддержку Java-версии. С выходом
    Parboiled2 потихонечку перестает поддерживаться и Scala-версия Parboiled1, однако не смотря на это, списывать её
    со счетов пока что не стоит:

      - Parboiled2 пока что не научился всем фичам Parboiled1;
      - Parboiled1 всё ещё используется гораздо шире, чем Parboiled2, поэтому если вас внезапно перебросят на
        какой-нибудь старый Scala-проект, высок шанс столкнуться именно с ним.

  - Parboiled2 — новейшая версия библиотеки, устраняющая ряд недостатков PB1. Работает быстрее и, что самое главное,
    поддерживается разработчиками.

[grappa]: https://github.com/fge/grappa
[fge]:    https://github.com/fge

Я писал эту статью с упором на Parboiled2 (кстати, дальше я буду писать о нём в мужском роде, без слова «библиотека»),
но иногда я буду отвлекаться, чтобы рассказать о важных отличиях между первой и второй версиями.


## Основные возможности
Краткая характеристика Parboiled2:

  - Следует принципам [PEG](https://en.wikipedia.org/wiki/Parsing_expression_grammar).
  - Генерирует однопроходные парсеры. Отдельный лексер не требуется.
  - Используется типобезопасный DSL, являющийся подмножеством языка Scala.
  - Оптимизации выполняются на этапе компиляции.

На практике это означает:

  - Вам не нужно писать парсер голыми руками.

  - Читаемость, сравнимая с лучшими сортами BNF (по-моему, PB даже круче).

  - Можно использовать всю мощь PEG и свободно разбирать рекурсивные структуры данных, в то время как регулярные
    выражения не могут этого [по определению][hier]. Да, регулярными выражениями вы не распарсите ни JSON, ни даже
    простейшее арифметическое выражение, что уж говорить о языках программирования. На StackOverflow есть
    [небезызвестная цитата в тему][paris]:

    > Asking regexes to parse arbitrary HTML is like asking Paris Hilton to write an operating system.

[paris]: http://stackoverflow.com/a/1733489/1447225
[hier]:  https://en.wikipedia.org/wiki/Chomsky_hierarchy#The_hierarchy

  - Даже если вам нужно разобрать линейную структуру, Parboiled2 (при использовании должных оптимизаций) будет
    работать быстрее регулярных выражений. Доказательства приведены в следующем разделе.

  - В отличие от генераторов парсеров, таких как [ANTLR], вы освобождены от мороки с раздельной генерацией кода и
    последующей его компиляцией. Весь код с Parboiled пишется на Scala, поэтому вы получаете подсветку синтаксиса
    и проверку типов из коробки, так же как и отсутствие дополнительных операций над файлами грамматик, в то время
    как парсер, сгенерированный ANTLR, будет иметь две фазы синтаксического разбора. Правда, несмотря на это,
    ANTLR всё равно мощнее, документированее и стабильнее, и поэтому может оказаться предпочтительнее во многих
    (*очень* нетривиальных) случаях.

[ANTLR]: http://www.antlr.org/

  - Скаловские парсер-комбинаторы работают медленно. Очень медленно. Неприлично медленно. Маттиас проводил сравнение
    производительности парсеров для Jackson и JSON, написанных с помощью Parboiled, Parboiled2 и Scala Parser
    Combinators. С неутешительными результатами для последних можно ознакомиться дальше по тексту.

  - В отличие от [Language Workbenches][lwb], Parboiled — маленькая и простая в использовании библиотека. Вам не нужно
    скачивать плохо документированного тормозящего монстра и тратить драгоценные часы жизни на изматывающий поиск
    нужных менюшек и кнопочек всего-навсего для описания небольшого DSL. С другой стороны, вы не получите готовый
    текстовый редактор с подсветкой вашего DSL из коробки, вместо этого вам придется самостоятельно написать плагин
    для Vim, Emacs или вашей IDE, но это не делает Parboiled менее достойной альтернативой для разработки небольших
    предметно-ориентированных языков.

[lwb]: http://martinfowler.com/bliki/LanguageWorkbench.html

  - Parboiled успешно зарекомендовал себя во [многих проектах][proj], в том числе и в кровавом энтерпрайзе.

[proj]: https://github.com/sirthias/parboiled/wiki/Projects-using-parboiled


## Новое в версии два

Этот раздел, в основном, будет полезен и понятен тем, кто уже работал с первой версией библиотеки. Новичкам, скорее
всего, стоит вернуться к этому списку после прочтения всего цикла статей.

Прежде всего, Parboiled2 успешно устраняет ряд детских болезней первой версии:

  - Появилась возможность использовать правила более вместительные, чем `Rule7`. Для этого была использована библиотека
    [shapeless][shapeless] с ее знаменитыми `HListами`: теперь одно правило может оперировать большим количеством
    значений на стеке. Это также означает, что в Parboiled2 появилась дополнительная зависимость, которой не было
	в PB1 — сама библиотека shapeless.

  - Добавлены недостающие конструкции. Так, в Parboiled1 нельзя было указать динамическое количество повторений
    для правила `nTimes` и приходилось использовать более «мягкое» правило `oneOrMore`, что не давало нужной точности
    описания грамматики.

  - Добавлены встроенные примитивные терминалы. Появился новый класс `CharPredicate`, который содержит такие поля,
    как `AlphaNumeric`, `Hex`, `Printable`, `Visible` и другие.

  - Добавлена возможность расширения и сужения предиката. Потребность исключить несколько символов из правила
    возникала и раньше, но только теперь это можно с легкостью взять и сделать, а не создавать белый список символов.

Кроме того:

  - Parboiled2 использует макросы, что позволяет генерировать грамматику на этапе компиляции, а не во время выполнения,
    как это было в Parboiled1. Это многократно увеличивает производительность вашего парсера, так же как увеличивает
    количество проверок. В связи с этим блок `rule` стал обязательным, хотя Parboiled1 позволял в некоторых случаях
    обходиться без него. Это нововведение вы заметите в первую очередь, когда будете делать миграцию старого кода.

  - Улучшена система отчета об ошибках.

  - Появилась поддержка [scala.js][scalajs]. Демо-проект можно посмотреть [здесь][scalajs-demo].

[shapeless]:    https://github.com/milessabin/shapeless
[scalajs]:      http://www.scala-js.org/
[scalajs-demo]: https://github.com/alexander-myltsev/parboiled2-scalajs-samples


# Сравнения производительности

Parboiled1 известен своей медлительностью (во всяком случае, по отношению к парсерам, генерируемым ANTLR),
вызванной тем, что все действия по сопоставлению правил выполнялись в рантайме и компилятор не мог производить
над таким парсером каких-либо существенных оптимизаций. В Parboiled2 во главу угла поставили производительность
и многие вещи были переделаны на макросах, благодаря чему компилятор получил свободу действий при
оптимизации, а пользователь — долгожданную производительность. Ниже мы продемонстрируем, каких неплохих
результатов добились разработчики.


## Parboiled против парсеров JSON, написанных прямыми руками

Parboiled — это обобщённый инструмент для создания парсеров, а как известно, специализированный инструмент всегда
оказывается лучше обобщённого в решении своей специализированной задачи. В мире Java существует небольшое количество
парсеров JSON, написанных вручную древними эльфийскими мастерами, и Александр Мыльцев (один из разработчиков
Parboiled2) проверил, насколько сильно Parboiled проигрывает в производительности этим артефактам.
[Результаты][bench-elv] оказались достаточно оптимистичными, особенно в случае с Parboiled2.

[bench-elv]: http://myltsev.name/ScalaDays2014/#/

      Тест-кейс                           │ Время, мс │
    ──────────────────────────────────────┼───────────┼─────────────────────────────────
      Parboiled1JsonParser                │     85.64 │ ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇
      Parboiled2JsonParser                │     13.17 │ ▇▇▇▇
      Json4SNative                        │      8.06 │ ██▍
      Argonaut                            │      7.01 │ ▇▇
      Json4SJackson                       │      4.09 │ ▇


## Parboiled против регулярных выражений

Благодаря использованию статических оптимизаций, Parboiled2 способен работать значительно быстрее регулярных выражений
(как минимум тех, что идут в комплекте с библиотекой классов Java). Вот немного подтверждающих данных
из [списка рассылки][bench-re]:

      Тест-кейс                           │ Время, мс │
    ──────────────────────────────────────┼───────────┼───────────────────────────────────
      Parboiled2 (warmup)                 │   1621.21 │ ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇
      Parboiled2                          │    409.16 │ ▇▇▇▇▇▇▇▇
      Parboiled2 w/ better types (warmup) │    488.92 │ ▇▇▇▇▇▇▇▇▇▇
      Parboiled2 w/ better types          │    134.68 │ ▇▇▇
      Regex (warmup)                      │    621.95 │ ▇▇▇▇▇▇▇▇▇▇▇▇
      Regex                               │    620.38 │ ▇▇▇▇▇▇▇▇▇▇▇▇

[bench-re]: https://groups.google.com/forum/#!msg/parboiled-user/XATcJRLTXjA/XSmf3n6gZSwJ


## Parboiled против Scala Parser Combinators

В списке рассылки можно найти и [другой тест производительности][bench-spc], который неплохо согласуется с первым
(про JSON) и содержит данные для сравнения со Scala Parser Combinators. Всё очень и очень печально.

      Тест-кейс                           │ Время, мс │
    ──────────────────────────────────────┼───────────┼─────────────────────────────────
      Parboiled1JsonParser                |     73.81 | ▇
      Parboiled2JsonParser                |     10.49 | ▎
      ParserCombinators                   |   2385.78 | ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇

[bench-spc]: https://groups.google.com/forum/#!topic/parboiled-user/bGtdGvllGgU


# Чего Parboiled не может

Большинство статей про комбинаторы парсеров начинается с изматывающих объяснений того, что такое PEG, с чем его есть и
почему его надо бояться. Для того чтобы парсить конфиги, досконально разбираться в этом не обязательно, но знать об
ограничениях данного типа грамматик всё равно стоит. Итак, Parboiled принципиально не умеет:

 - Разбирать леворекурсивные грамматики. Это не под силу всем нисходящим парсерам (top-down parsers), к коим относятся
   и PEG. Однако, леворекурсивную грамматику можно [адаптировать][lrg-adapt].

[lrg-adapt]: http://neerc.ifmo.ru/wiki/index.php?title=Устранение_левой_рекурсии

 - Разбирать грамматики на отступах (indentation-based grammars), например Python или YAML. Не получается это сделать
   из-за того, что сгенерированный парсер является однопроходным, без отдельного лексера. Разбор отступов же
   выполняется на этапе лексического анализа. У этой проблемы есть простое решение: напишите препроцессор, который
   расставит виртуальные маркеры до (`INDENT`) и после (`DEDENT`) выхода в отступ. В Parboiled1 имеются для этого
   [стандарные инструменты][pb1-ibg], но для Parboiled2 подобную процедуру пока что придётся выполнять самостоятельно.

[pb1-ibg]: https://github.com/sirthias/parboiled/wiki/Indentation-Based-Grammars

 - Использовать потоковый ввод (streaming input). PEG используют поиск с возвратом, он же [бэктрекинг][backtracking].
   Теоретически, этот недостаток можно устранить при помощи буферизации потока, но ничто не мешает написать такую
   грамматику, в которой происходит возврат к самому началу. Поэтому, чтобы эта идея заработала на практике, необходимо
   научиться определять по грамматике границы чанков, между которыми возврат невозможен. Матиас весьма
   [заинтересован][mattias-wish] в разработке этой фичи, так что возможно ее появление в следующих релизах.

[backtracking]: https://en.wikipedia.org/wiki/Backtracking
[mattias-wish]: https://groups.google.com/d/msg/parboiled-user/b7PH49fiFco/gGt46xe3Ae4J

В следующей части я расскажу о том, как в Parboiled описывается пользовательская грамматика, а ещё мы напишем простой
распознаватель для древовидного формата конфигурационных файлов.
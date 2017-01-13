Про ScalaCheck
==============

**Часть 1. Зачем ScalaCheck?**.

ScalaCheck — это [комбинáторная](combinator-library) библиотека, значительно
облегчающая написание модульных тестов на Scala. В ней используется подход
property-based тестирования, впервые реализованного в библиотеке
[QuickCheck](quickcheck-intro) для языка Haskell. Существует множество
реализаций QuickCheck: есть реализации для [Java](quickcheck-java),
[C](quickcheck-c), а так же [других](quickcheck-wiki) языков и платформ.

Эта серия статей во многом похожа на мою [предыдущую](parboiled), посвященную
Parboiled, поэтому структура этой серии статей будет похожей. Здесь вы узнаете
о подходе, именуемом property-based testing, который на русский можно перевести
как «свойство-ориентированное тестирование». Использование данного подхода
позволяет значительно сократить время на разработку тестов. В этой и последующих
статьях мы научимся смотреть на мир сквозь свойства, пощупаем генераторы.
Заинтересовало? Прошу под кат.

[combinator-library]: https://en.wikipedia.org/wiki/Combinator_library
[quickcheck-wiki]: https://en.wikipedia.org/wiki/QuickCheck
[quickcheck-intro]: http://www.stuartgunter.org/intro-to-quickcheck/
[quickcheck-c]: https://github.com/silentbicycle/theft
[quickcheck-java]: https://github.com/pholser/junit-quickcheck/
[parboiled]: https://habrahabr.ru/post/270233/

<cut text="Читать про ScalaCheck →">

**Структура цикла**

 - Зачем ScalaCheck?
 - Генераторы
 - Свойства
 - Минимизация
 - Интеграция и настройки


# Введение

Модульное тестирование является одним из важнейших подходов в разработке
программного обеспечения. Даже если ваша программа при компиляции проходит
проверки на уровне системы
типов, это еще не означает, что в ней отсутствуют логические ошибки.
Это значит, что каким бы мощным ни был ваш язык программирования, без
тестирования кода не обойтись. Однако, стоимость
тестирования весьма высока: помимо потраченных человеко-часов требуется
тратить нечеловеко-yсилия на рутинное написание модульных тестов.
Из-за этого многие заказчики экономят на тестировании, чем многие
программисты пользуются с превеликой радостью: модульные тесты писать
скучно (но нужно!).
Cогласитесь, покрывать огромное количество случаев и писать однотипные
синтаксические конструкции изо дня в день — удовольствие сомнительное.

К сожалению, даже типобезопасный код на функциональных языках нужно тестировать.
И хотя уже само применение функциональной парадигмы значительно облегчает модульное
тестирование — отсутствие побочных эффектов позволяет рассматривать меньше
краевых случаев и совсем забыть о тестировании состояния, — количество тестирующего кода
это значительно не сокращает. Кроме того, несмотря на то, что ручное тестирование
даст вам некоторую уверенность в вашем коде, оно не может гарантировать, что со временем
не обнаружится какой-нибудь частный случай, который вы упустили из виду.

Подход ScalaCheck позволяет перейти на следующий уровень абстракции и избежать
значительной части этой рутины, одновременно повысив читаемость тестирующего кода
и сократив его объем (что положительно скажется на его ремонтопригодности).
Покрытие вашего кода тестами также заметно возрастет.
И чтобы добиться этого, нужно только познакомиться с несколькими новыми
концепциями.


# Свойство-ориентированное тестирование

## Свойствах в общем

Итак, давайте для начала разберемся в том, что является свойством. Если в двух
словах, то свойство — это логическое выражение. На него можно смотреть как на
некоторый закон или правило, можно как на некий атрибут черного ящика. Нас не
волнует состояние тестируемой функции, нас волнуют входные и выходные данные,
спецификация. Вот простейший пример свойства, для понимающих *«греческий»*:

    ∀x ∈ ℝ:  x ≠ 0 ⟹ x² > 0

Если вы не понимаете греческий, не отчаивайтесь, дальше все будет
изложено на английском.

    // Автор прекрасно понимает, что тип Double никогда не являлся ℝ,
    // однако, для упрощения изложения, этим бесспорно важным свойством
    // придется пожертвовать.
    forall { x: Double => (x > 0) ==> x * x > 0 }

По-русски это читается так: «Для любого вещественного x больше 0, x² всегда
больше 0».

Как видите, свойство представляет собой более высокий уровень абстракции, чем
традиционный assertion-based тест в JUnit. Но на основе этого достаточно
абстрактного описания свойств, ScalaCheck выполняет вполне конкретные тесты,
сопоставимые по качеству с теми, которые бы вы писали руками.

## Это не теории

JUnit4 обладает одной уникальной функцией, которая называется *theories*. Они
бы ещё гипотезами их назвали. Теории работают весьма похожим образом, но в
отличии от ScalaCheck, они не позволяют генерировать входные данные
произвольным образом, а так же выполнять минимизацию (shrinking).

> Далее я буду использовать, возможно, не очень корректный термин «минимизация»
> в качестве вполне понятного термина 'shrinking'. Сужение, уж простите,
> звучит как диагноз.

Итак, что представляют из себя теории? Теория проверяет утверждение о том, что
некоторое условие всегда верно для набора заданных данных (они же *data points*).
Количество наборов данных неограниченно. Для того чтобы ваш метод стал теорией, его
нужно соответствующим образом проаннотировать: добавить `@Theory`. Входные
данные аннотируются как `@DataPoint`. Раннер запускает тест каждый раз для
каждого из дата-поинтов. Вот небольшой пример, нагло позаимствованный из
документации JUnit:

    @RunWith(Theories.class)
    public class UserTest {

       // Первый тестовый набор — хороший.
       @DataPoint
       public static String GOOD_USERNAME = "optimus";

       // Второй тестовый набор — плохой, на нем тест и упадет.
       @DataPoint
       public static String USERNAME_WITH_SLASH = "optimus/prime";

       @Theory
       public void filenameIncludesUsername(String username) {
           assumeThat(username, not(containsString("/")));
           assertThat(new User(username).configFileName(),
                      containsString(username));
       }
    }

Механизм похож на тот что использует ScalaCheck. Существует лишь два
существенных отличия:

 - ScalaCheck сам генерирует тесты;
 - когда тест падает, из всех *случайным образом сгенерированных* наборов данных
   ScalaCheck выбирает частный случай.

Поэтому, согласитесь, что в сравнении с теориями ScalaCheck выигрывает
как по объему кода, так и по количеству тестов. Узнать больше о теориях, вы
можете [здесь](more-on-theories).

[junit-theories]: https://github.com/junit-team/junit4/wiki/Theories
[more-on-theories]: http://web.archive.org/web/20110608210825/http://shareandenjoy.saff.net/tdd-specifications.pdf


## Плюсы и минусы property-based подхода

ScalaCheck создавался не для того, чтобы полностью вытеснить ScalaTest или
пресловутый JUnit, а для того, чтобы внести в процесс модульного тестирования
дополнительные преимущества:

+ Лаконичность: меньше кода — выше покрытие в сравнении со стандартным
  (assertion-based) подходом
+ Высокоуровневость: мы фокусируемся на входных данных вообще, а не на частных случаях.
+ Минимизация: когда что-то сломалось, нам помогут найти,
  где именно (или мы поможем себе сами).
+ Наличие сущностей, которые проще тестировать как единое целое, нежели
  тестировать их покомпонентно.

Свойство-ориентированное тестирование — не серебряная пуля. У этого подхода есть и
недостатки:

- Медленный код убивает процесс тестирования. (Хотя возможно, это и плюс. Моей
  практике известны случаи, когда использование ScalaCheck заставляло задуматься
  об оптимизации. Иногда можно подождать.)
- Ложное ощущение безопасности. Мы думаем, и уверенны что покрыли тестами, все
  что могли, хотя зачастую это вовсе не так.
- Нечувствительность к граничным условиям.
- Равномерное случайное распределение тестовых наборов. В общем-то, это хорошо,
  но не всегда удобно.

## Когда использовать?

ScalaCheck можно использовать так же, как и любой другой фреймворк для
тестирования. Да, придется думать несколько иначе, однако ScalaCheck позволит
вам написать практически любой тест. Просто не всегда это эффективно, удобно и
читаемо. Однако, есть области где ScalaCheck будет более эффективен:

+ код, крайне чувствительный к входным данным;
+ конечные автоматы или любые системы, зависящие от состояния;
+ парсеры (то для чего использую ScalaCheck лично я);
+ разнообразные [преобразователи данных][data-processors]:
  + валидаторы;
  + классификаторы;
  + агрегаторы;
  + сортировщики, и т.д.
+ Spark RDD, а также маперы и редьюсеры для Hadoop.

[data-processors]: https://en.wikipedia.org/wiki/Data_processing


# ScalaCheck

## Особенности библиотеки

ScalaCheck — это:

- компактная библиотека (менее двадцати файлов с кодом);
- отсутствие дополнительных зависимостей;
- поддержка тестов с внутренним состоянием (stateful testing);
- отказ от использования `java.util.Random` в качестве генератора
  псевдослучайных чисел (более того, ScalaCheck внимательно следит за тем,
  чтобы случайные тестовые наборы не повторялись,
  используя для этого [поиск с возвратом][backtracking]).
- работоспособность на scala-js и Dotty.

> Внутри `java.util.Random` используется [линейный конгруэнтный][lcg] метод
> генерации псевдослучайных последовательностей (далее — LCG). Подробнее вы можете
> прочитать в официальной [документации][jur]. LCG не обеспечивают достаточного
> качества генерации псевдослучайных чисел, и используются в большинстве
> библиотек исключительно за счет простоты и высокой производительности.
> ScalaCheck использует свой генератор, который гораздо лучше ведет себя в
> серьезных статистических применениях. Подробнее о генераторе вы можете узнать
> [здесь][ssrng].

[stateful-scalacheck]: http://scalacheck.org/files/scaladays2014/#1
[lcg]: https://en.wikipedia.org/wiki/Linear_congruential_generator#Advantages_and_disadvantages
[backtracking]: https://en.wikipedia.org/wiki/Backtracking
[jur]: https://docs.oracle.com/javase/8/docs/api/java/util/Random.html
[ssrng]: http://burtleburtle.net/bob/rand/smallprng.html


## Подготовительные работы

После того, как мы определились с тем, нужно ли нам свойство-ориентированное
тестирование и ScalaCheck в частности, давайте приступим к подготовительным
работам. Добавьте следующую зависимость в ваш проект (я рассчитываю что вы,
уважаемый читатель, уже перешли на Scala 2.12):

    <!-- Пользователи sbt и gradle, скорее всего, знают о Maven. -->
    <dependency>
        <groupId>org.scalacheck</groupId>
        <artifactId>scalacheck_2.12</artifactId>
        <version>1.13.4</version>
    </dependency>

Так же предполагается, что вы используете последнюю версию. Существует
[проблема](deadlock-problem) при использовании устаревших версий библиотеки
в связке с Scala 2.12. Будьте осторожны.

[deadlock-problem]: https://github.com/rickynils/scalacheck/issues/290

Как уже было сказано ранее, ScalaCheck построен на двух основных концепциях:
на свойствах и генераторах. Свойства хорошо освещены во множестве блогов, в том
числе и русскоязычных. Поэтому больше внимания я постараюсь уделить генераторам.
В этой части мы **бегло** ознакомимся как со свойствами, так и с генераторами.

## Немного о свойствах

Свойство представляет собой минимальный тестируемый модуль. Представлено
экземпляром класса `org.scalacheck.Prop`. Простейший пример:

    import org.scalacheck.Prop

    val propStringLengthAfterConcat = Prop forAll { s: String =>
      val len = s.length
      (s + s).length == len + len
    }

    // Только при наличии конкретного типа исключения, свойство будет считаться
    // успешным
    val propDivByZero = Prop.throws(classOf[ArithmeticException] {1/0})

    // Для любого целочисленного списка, при доступе к элементу индекс которого
    // на единицу больше его длины, всегда будет выброшено
    // исключение IndexOutOfBoundsException
    val propListIndexOutOfBounds = Prop forAll { xs: List[Int] =>
      Prop.throws(classOf[IndexOutOfBoundsException]) {
        xs(xs.length + 1)
      }

## Немного о генераторах

На практике генераторы приходится писать не реже, чем свойства. Для того чтобы
воспользоваться ими, вам следует проимпортировать `org.scalacheck.Gen`.

    import org.scalacheck.Gen

    // Генераторы, которые случайно выбирает значение из равномерно
    // распределенного диапазона.
    val binaryDigit = Gen.choose(0, 1)
    val octDigit    = Gen.choose(0, 7)

    // Генератор, который с равной вероятностью выбирает значение
    // из предоставленного списка.
    val vowel = Gen.oneOf('a', 'e', 'i', 'o', 'u')

Также в ScalaCheck имеется набор готовых генераторов для стандартных типов:

    // Генератор, возвращающий случайную строчную букву.
    val alphaLower = Gen.alphaLowerChar

    // Генератор, возвращающий случайный идентификатор (строку случайной
    // длины, первый символ которой всегда строчная буква, а дальнейшие
    // могут быть только цифрами или буквами).
    val identifier = Gen.identifier

    // Генератор, возвращающий случайное положительное число типа Long.
    val natural = Gen.posNum[Long]

Вы также можете объединять имеющиеся генераторы, применяя к ним `map` и for
comprehension:

    val personGen = for {
      charValue <- Gen.oneOf("Jason", "Oliver", "Jessica", "Olivia")
      ageValue  <- Gen.posNum[Int]
    } yield Person (name = nameValue, age = ageValue)

И свойства и генераторы мы рассмотрим подробнее в следующих статьях. Следующая
статья серии будет посвящена генераторам. Спасибо что дочитали, оставайтесь
на связи.
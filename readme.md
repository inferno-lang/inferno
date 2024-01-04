# Inferno
Inferno это идея и прототип языка программирования под JVM экосистему с полной совместимостью, с кодовыми базами на других JVM языках (в частности с Java).

## Основные принципы

Принципы описанные ниже, могут быть весьма спорными (относительно других языков), но это и является, "вишенкой на торте" этого языка.

1. Простота, нет синтаксического "сахара". Минимум ключевых слов, пунктуаторов и синтаксических конструкций и их вариаций.
3. Нет перегрузки операторов. [Повторяя ошибки предков — Перегрузка операторов](https://erokhin.net/articles/operator-overloading)
4. Разделение на "сущности" и "классы".
5. Нет именованных параметров (за исключением аннотаций). [Ошибка на миллиард — Именованные параметры](https://erokhin.net/articles/named-parameters)
6. Нет функций расширения (extension function). [Почему экстеншены зло](https://erokhin.net/articles/extension-functions)
7. Подсказки для компилятора являются аннотацией CompileControl. (никаких кейвордов по типу inline, tailrec и пр.)
8. Простой и единообразный синтаксис.
9. Специальные expression receivers которые могут упростить контроль над флоу в коде.

## Техническая составляющая

Часть языка пишется на Java с использованием ANTLR, соответственно, сгенерированные исходники ANTLR на Java, генератор байт кода с использованием ASM, аналогично на Java, все остальное на Inferno.

Используемые внутри библиотеки:
Cactoos, ANTLR, ASM

Разрабатывается на OpenJDK 21. Генерирую код, совместимый с JVM версии 11 (LTS). Для сборки используется система сборки Gradle (8.3)

## Синтаксис

### Простой пример с сущностью

```r
namespace objects

entity Books ^ BooksEnvelope, Closeable
ctor(field db: Database)

fetch -> List[Book] {
  <- db.exec("select id from books").map(id => Book.new(id, db))
  ; Или используя expression receiver
  !return <- db.exec("select id from books").map(id => Book.new(id, db))
}
```

Инстанцировать объекты (в т.ч и анонимные классы), можно с помощью new "метода"

```r
demo -> Nothing {
  books <- Books.new(DatabaseSingletoneFabric.instance())
}
```

### Expression receivers (Ресиверы выражений)

Все ресиверы выражений реализуются на стороне компилятора, и не могут быть добавлены пользователем, принимают только вычисляемое выражение, вот несколько примеров.

```r
exprReceiverDemo -> Nothing {
  !guard   <- true = true   ; Принимает boolean выражение и бросает GuardViolationException если значение false.
  !require <- arr.get(0)    ; Буквально требует значение или кладет на лопатки JVM с RequireContractException
  !return  <- arr.get(0)    ; Явно возвращает управление из метода
  !recover <- arr.get(0)    ; Подавляет исключение (Exception) и передает управление дальше.
  !defer   <- db.close()    ; Откладывает безусловное исполнение блока до конца исполнения пользовательского кода в методе.
  _        <- db.info()     ; Игнорирует, отправляет вникуда значение выражения (подобное Blackhole#consume в JMH).

  ; Так как принимает вычисляемое выражение, то можно использовать блоки, к примеру:

  !recover <- {
    val <- array.get(0)
    str <- String.new(val)
  }
}
```

Отдельный пример с defer блоком, если работали с Golang то уже понятно о чем речь.
Defer expression receiver работает очень просто, откладывает исполнение вычисляемого выражения до конца функции, даже если произойдет исключение.

Если в выражении для defer ресивера будет выброшено исключение, то будет выброшен `DeferDiedException`

```r
fetch -> Nothing {
  sql <- db.connect("connection string")
  !defer <- db.close()

  sql.exec("select * from animals")
  panic("Выбрасываем исключение, но коннект с бд будет закрыт.")
}
```

### Основные выражения

В языке всего две конструкции являющиеся выражением это when и тернарный оператор:

```r
test -> Nothing {
  count <- state.count = 0 ? 10 : state.count + 1

  msg <- when (readln().trim().normalize()) {
    ""      => "Name can't be empty"
    "admin" => "Admin name is restricted"
    "root"  => "Illegal name"
    other   => panic()
  }
}
```

### Массивы и листы

В языке можно создать синтаксически List (реализация ArrayList) и обычный jvm массив

```r
collections -> Nothing {
  sarr  <- ["Hello", " ", "world"]     ; Массив строк, выведенный тип является String[].
  slist <- [| "Hello", " ", "world" |] ; Лист строк, выведенный тип является List[String] (реализация ArrayList[String])
}
```

### Многопоточное программирование

Язык так же предоставляет простые методы по работе с многопоточным программированием, никакой раскраски кода [отсылка](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). Реализация полностью соотносится к прямому использованию CompletableFuture из JDK (CF в дальнейшем).

```r
go -> Nothing {
  async(2)                        ; CompletableFuture.completedFuture(2)
  async(() => {})                 ; CompletableFuture.runAsync(() => {})
  async(() => 2)                  ; CompletableFuture.supplyAsync(() => 2)
  asyncAll(() => 2,  () => {}, 2) ; CompletableFuture.allOf(CompletableFuture.supplyAsync(() => 2), CompletableFuture.runAsync(), CompletableFuture.completedFuture)
  asyncAny(() => 2,  () => {}, 2) ; CompletableFuture.anyOf(CompletableFuture.supplyAsync(() => 2), CompletableFuture.runAsync(), CompletableFuture.completedFuture)

  ; Простой пример асинхронной задачи

  async(() => db.exec("select * from books").map(Book::new))
    .exceptionaly(...)
    .xxx()
}
```

### Помогая компилятору

В языке так же можно менять семантику конечного кода (промежуточного - JVM Bytecode) с помощью аннотации `CompileControl`

```r
@CompileControl(
  compatible <- Ver1_0,
  preview    <- true,
  tailrec    <- true,
  inline     <- true,
  assertions <- false,
  trace      <- true
)
fetch -> List[Book] {
  <- db.exec("select id from books").map(Book::new)
}
```

### Модификаторы доступа и аттрибуты для JVM

В языке есть возможность явно "экспоузить" метод, поле, класс с каким-либо модификатором или аттрибутом. К примеру:

```r
@Expose(attrs <- [Private, Static, Native, Synchronized, Strictfp, Final, Virtual])
fetch -> List[Book] {
  !return <- db.exec("select id from books").map(Book::new)
  <- db.exec("select id from books").map(Book::new)
}

; Так же есть более краткая запись без явного доступа к attrs:

@Expose(Protected)
field cache <- LogCache.new(RamCache.new())

@Expose([Private, Static, Native, Synchronized, Strictfp, Final, Virtual])
fetch -> List[Book] {
  !return <- db.exec("select id from books").map(Book::new)
  <- db.exec("select id from books").map(Book::new)
}
```

### Анонимные классы

В языке так же есть анонимные классы, но ее дизайн пока что экспериментальный и возможно, в конечном итоге их не будет.

```r
Thread.new(Runnable.new() {
  run -> Nothing {
    println(3)
  }
}).start()
```

### Обобщения, параметризованные типы

Обобщения (в дальнейшем дженерики), реализованы очень просто, примером ярким для меня послужил Golang.

```r
class Collection

; Декларация дженерика на уровне метода
get[T](idx: Int) -> T {
  <- todo("using of placeboo stdlib version")
}

; Декларация дженерика на уровне класса
class GenericCollection[T]

get(idx: Int) -> T {
  <- todo("using of placeboo stdlib version")
}
```

### Комментарии и документация

Документация как и комментарий декларируется с `;` с отличием что для документации их надо три `;;;`

Пример:

```r
;;; Декоратор для загрузки книг из базы данных
;;;
;;; @param What какой-то параметр
;;; @returns Коллекцию книг
;;; @see     Books#fetch
;;; @since   v1.0
fetch(what: String) -> List[Book] {
  db.exec("select id from books").map(Book::new) ; Грузим книжечки и преобразуем в Book
}
```

### Различия между сущностью и классом:
1. По умолчанию методы и конструктора публичные.
2. Все поля иммутабельные и приватные.
3. У каждой сущности есть свой уникальный ID.
4. Равенство и хешкод базируется на уникальном ID.
5. Не наследуемые.

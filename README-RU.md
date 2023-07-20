# Inferno
Inferno это идея и прототип языка программирования под JVM экосистему с полной совместимостью, с кодовыми базами на других JVM языках.

## Основные принципы

Принципы описанные ниже, могут быть весьма спорными, но это и является, "вишенкой на торте" этого языка.

1. Нет переменных и полей.
2. Нет операторов.
3. Нет обнуляемых значений и нуллов. (за исключением при интеропе с другими JVM языками, которые их допускают)
4. Полностью интроспектируемый. (Классы, методы, параметры и параметризированные параметры)
5. Простой и единообразный синтаксис.
6. Все является объектом.
7. Минимум вербозности. Насколько это возможно, типы выводит компилятор. Минимум пунктуаторов и операторов

## Немного примеров и сравнений с Java

### Классы

#### Самый обычный класс, имеющий айдентити по умолчанию

```Ruby
.SomeClass
```

#### Класс, принимающий аргумент (тип выводится на основании того, как используется, и как сам SomeClass используют)

```Ruby
.SomeClass otherClass
```

#### Параметризированный класс принимающий два параметризированных аргумента (один из которых имеет constraint)

```Ruby
.SomeClass['T 'R^CharSequence] some: T str: R
```

#### Интерфейс и абстрактный класс (используются аттрибуты)

```Ruby
.SomeInterface [interface]
```

И абстрактный класс,

```Ruby
.SomeAbstClass [abstract]
```

#### Наследование

Пример где Child класс наследуется от класса/интерфейса Parent и класса/интерфейса ILogger

*Множественное наследование не поддерживается*

```Ruby
.Child ^ Parent ILogger
```

#### Статический/виртуальный класс

```Ruby
.Static [static]
```

```Ruby
.Virtual [virtual]
```

### Для сравнения с джавой

```Java
public class Some {
}
```

```Ruby
.Some [virtual identiyless]
```

```java
public final class Test {
  public void test() {
    new Impl<String, String>("1", "3").noop()
  }
}
```

```Ruby
.Test [identiyless]
  #test
    Impl::new '1' '3' ||> ::noop
```

```Java
public final class SomeIdentity {
  @Override
  int hashCode() {
    ...
  }

  @Override
  boolean equals() {
    ...
  }
}
```

```Ruby
.SomeIdentity
```

```Java
public final class SomeIdentity2 extends AbstractList {
  @Override
  int hashCode() {
    ...
  }

  @Override
  boolean equals() {
    ...
  }
}
```

```Ruby
.SomeIdentity2 ^ AbstractList
```

```Java
public final class Impl<T, R extends CharSequence & Serializable> implements ILogger, List<T> extends Abst<T, R> {
  private final T some;
  private final R str;

  public Impl(T some, R str) {
    super(some, str);
    this.some = some;
    this.str = str;
  }

  public void create(Object id, Object stream) {
    return new Ignore().noop()
  }
}
```

```Ruby
.Impl['T 'R^CharSequence Serializable] some str ^ ILogger List[T] Abst[T R] some str
  #create id stream
    Ignore::new::noop
```
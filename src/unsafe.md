% Небезопасный код

Главная сила Rust — в мощных статических гарантиях правильности поведения
программы во время исполнения. Но проверки безопасности очень осторожны: на
самом деле, существуют безопасные программы, правильность которых компилятор
доказать не в силах. Чтобы писать такие программы, нужен способ немного ослабить
ограничения. Для этого в Rust есть ключевое слово `unsafe`. Код, использующий
`unsafe`, ограничен меньше, чем обычный код.

Давайте рассмотрим синтаксис, а затем поговорим о семантике. `unsafe`
используется в четырёх контекстах. Первый — это объявление того, что функция
небезопасна:

```rust
unsafe fn beregis_avtomobilya() {
    // страшные вещи
}
```

Например, все функции, вызываемые через [FFI][ffi], должны быть помечены как
небезопасные. Другое использование `unsafe` — это отметка небезопасного блока:

[ffi]: ffi.html

```rust
unsafe {
    // страшные вещи
}
```

Третье — небезопасные типажи:

```rust
unsafe trait Scary { }
```

И четвёртое — реализация (`impl`) таких типажей:

```rust
# unsafe trait Scary { }
unsafe impl Scary for i32 {}
```

Важно явно выделить код, ошибки в котором могут вызвать большие проблемы. Если
программа на Rust падает с "segmentation fault", можете быть уверены —
проблема в участке, помеченном как небезопасный.

# Что значит "безопасный"?

В контексте Rust "безопасный" значит "не делает ничего небезопасного". Также
важно знать, что некоторое поведение скорее всего нежелательно, но явно
_не_ считается небезопасным:

* Deadlock'и
* Утечка памяти или других ресурсов
* Выход без вызова деструкторов
* Целочисленное переполнение

Rust не может предотвратить все виды проблем в программах. Код с ошибками может
и будет написан на Rust. Вышеперечисленные вещи неприятны, но они не считаются
именно что небезопасными.

В дополнение к этому, ниже представлен список неопределённого поведения
(undefined behavior) в Rust. Избегайте этих вещей, даже когда пишете
небезопасный код:

* Гонка данных
* Разыменование нулевого или висячего указателя
* Чтение [неинициализированной][undef] памяти
* Нарушение [правил о совпадении указателей][aliasing] с помощью сырых
  указателей
* `&mut T` и `&T` следуют модели LLVM [noalias][noalias], кроме случаев, когда
  `&T` содержит `UnsafeCell<U>`. Небезопасный код не должен нарушать эти
  гарантии совпадения указателей.
* Изменение неизменяемого значения или ссылки без использования `UnsafeCell<U>`
* Получение неопределённого поведения с помощью intrinsic-операций компилятора:
    * Индексация вне границ объекта с помощью `std::ptr::offset` (`offset`
      intrinsic), кроме разрешённого случая "один байт за концом объекта".
    * Использование `std::ptr::copy_nonoverlapping_memory` (intrinsic-операции
      `memcpy32`/`memcpy64`) с пересекающимися буферами
* Неправильные значения примитивных типов, даже в скрытых полях:
    * Нулевые или висячие ссылки или упаковки (boxes)
    * Любое значение логического типа, кроме `false` (0) или `true` (1)
    * Вариант перечисления, не включённый в его определение
    * Суррогатное значение `char` или значение `char`, превыщающее `char::MAX`
    * Последовательности байт, не являющиеся UTF-8, в `str`
* Размотка стека в код на Rust из чужого кода (через границы FFI), или размотка
  из кода на Rust в чужой код

[noalias]: http://llvm.org/docs/LangRef.html#noalias
[undef]: http://llvm.org/docs/LangRef.html#undefined-values
[aliasing]: http://llvm.org/docs/LangRef.html#pointer-aliasing-rules

# Сверхспособности небезопасного кода

В небезопасном блоке или функции, Rust разрешает три ситуации, которые обычно
запрещены. Всего три. Вот они:

1. Доступ к или изменение [статической изменяемой переменной][static].
2. Разыменование сырого указателя.
3. Вызов небезопасных функций. Это самая мощная возможность.

Это всё. Важно отметить, что `unsafe`, например, не "выключает проверку
заимствования". Объявление какого-то кода небезопасным не изменяет его
семантику; небезопасность не означает принятие компилятором любого кода. Но она
позволяет писать вещи, которые _нарушают_ некоторые из правил.

Вы также встретите ключевое слово `unsafe`, когда будете реализовывать интерфейс
к чужому коду не на Rust. Идиоматичным считается написание безопасных обёрток
вокруг небезопасных библиотек.

Давайте поговорим о трёх упомянутых возможностях, доступных в небезопасном коде.

## Доступ или изменение `static mut`

Rust позволяет пользоваться глобальным изменяемым состоянием с помощью `static
mut`. Это может вызвать гонку по данным, и в сущности небезопасно. Подробнее
смотрите раздел о [static][static].

[static]: const-and-static.html#static

## Разыменование сырого указателя

Сырые указатели поддерживают произвольную арифметику указателей, и могут вызвать
целый ряд проблем безопасности памяти и безопасности в целом. В каком-то смысле,
возможность разыменовать произвольный указатель — одна из самых опасных вещей,
которые вы можете сделать. Подробнее смотрите раздел о
[сырых указателях][rawpointers].

[rawpointers]: raw-pointers.html

## Вызов небезопасных функций

Эта возможность затрагивает то, откуда можно делать вызов небезопасного кода:
небезопасные функции могут вызываться только из небезопасных блоков.

Мощь и полезность этой возможности сложно переоценить. Rust предоставляет
некоторые [intrinsic-операции][intrinsics] компилятора в виде небезопасных
функций, а некоторые небезопасные функции обходят проверки безопасности для
достижения большей скорости исполнения.

В заключение, повторимся: хотя вы и _можете_ делать в небезопасных участках
почти что угодно, это не значит, что стоит это делать. Компилятор будет
предполагать выполнение оговоренных инвариантов, так что будьте осторожны!

[intrinsics]: intrinsics.html

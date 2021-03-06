% Вернемся к Метапеременным и Развертыванию

Когда парсер начинает захватывать токены в метапеременные, *он уже не может остановиться или откатиться*. Это означает, что паттерн во втором правиле макроса, приведенного ниже,  *никогда не совпадет с образцом*, что бы ни подавали на вход:

```ignore
macro_rules! dead_rule {
    ($e:expr) => { ... };
    ($i:ident +) => { ... };
}
```

Представим что случится, если этот макрос вызвать как  `dead_rule!(x+)`.  Интерпретатор начнет с первого правила и попытается распарсить вход как выражение.  Первый токен  (`x`) подходит как выражение.  Второй *тоже* подходит как выражение, представляя собой узел двоичного сложения.

В таком случае, выдав это выражение на вход, без правого оператора сложения, вы ожидаете, что парсер пропустит первое правило и перейдет ко второму.  Вместо этого, парсер вызовет панику и прервет компиляцию, ссылаясь на ошибку в синтаксисе.

Таким образом, важно понимать, что вы должны при написании правил в паттернах указывать наиболее общую форму, а не какую-либо конкретную.

Для защиты от дальнейших изменений синтаксиса, затрагивающих интерпретацию входа макроса, `macro_rules!` ограничивает то, что может следовать за различными метапеременными. Полный список для Rust 1.3 следующий:

* `item`: что угодно.
* `block`: что угодно.
* `stmt`: `=>` `,` `;`
* `pat`: `=>` `,` `=` `if` `in`
* `expr`: `=>` `,` `;`
* `ty`: `,` `=>` `:` `=` `>` `;` `as`
* `ident`: что угодно.
* `path`: `,` `=>` `:` `=` `>` `;` `as`
* `meta`: что угодно.
* `tt`: что угодно.

В дополнение, `macro_rules!` обычно запрещает повторению идти за другим повторением, даже если по содержанию они не конфликтуют.

Еще одна особенность подстановки, которая обычно удивляет людей, это то, что подстановка  *не* основана на токенах, несмотря на то, что *выглядит* именно так.  Вот простая демонстрация этого:

```rust
macro_rules! capture_expr_then_stringify {
    ($e:expr) => {
        stringify!($e)
    };
}

fn main() {
    println!("{:?}", stringify!(dummy(2 * (1 + (3)))));
    println!("{:?}", capture_expr_then_stringify!(dummy(2 * (1 + (3)))));
}
```

Помните, что `stringify!` - это встроенное расширение синтаксиса, которое берет все токены, которые ей дали на вход, и объединяет их в одну большую строку.

Вывод будет следующий:

```text
"dummy ( 2 * ( 1 + ( 3 ) ) )"
"dummy(2 * (1 + (3)))"
```

Заметьте, что, *несмотря на* одинаковые входные данные, выводы разные.  Это происходит из-за того, что первый вызов преобразует в строку выражение из деревьев токенов, в то время как второй преобразует в строку  *узел выражения AST*.

Для визуализации разницы - вот, с чем на входе, макрос `stringify!` вызывается в первом случае:

```text
«dummy» «(   )»
   ╭───────┴───────╮
    «2» «*» «(   )»
       ╭───────┴───────╮
        «1» «+» «(   )»
                 ╭─┴─╮
                  «3»
```

…а вот, с чем, он вызывается во втором:

```text
« »
 │ ┌─────────────┐
 └╴│ Call        │
   │ fn: dummy   │   ┌─────────┐
   │ args: ◌     │╶─╴│ BinOp   │
   └─────────────┘   │ op: Mul │
                   ┌╴│ lhs: ◌  │
        ┌────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
        │ LitInt │╶┘ └─────────┘ └╴│ BinOp   │
        │ val: 2 │                 │ op: Add │
        └────────┘               ┌╴│ lhs: ◌  │
                      ┌────────┐ │ │ rhs: ◌  │╶┐ ┌────────┐
                      │ LitInt │╶┘ └─────────┘ └╴│ LitInt │
                      │ val: 1 │                 │ val: 3 │
                      └────────┘                 └────────┘
```

Как вы можете заметить, здесь именно *одно* дерево токенов, которое содержит AST, который был распарсен из входа вызова `capture_expr_then_stringify!`.  Отсюда то, что вы здесь видите в выводе - не преобразованные в строку токены, а преобразованные в строку *узлы AST*.

Эта особенность имеет и другие последствия. Представьте следующее:

```rust
macro_rules! capture_then_match_tokens {
    ($e:expr) => {match_tokens!($e)};
}

macro_rules! match_tokens {
    ($a:tt + $b:tt) => {"got an addition"};
    (($i:ident)) => {"got an identifier"};
    ($($other:tt)*) => {"got something else"};
}

fn main() {
    println!("{}\n{}\n{}\n",
        match_tokens!((caravan)),
        match_tokens!(3 + 6),
        match_tokens!(5));
    println!("{}\n{}\n{}",
        capture_then_match_tokens!((caravan)),
        capture_then_match_tokens!(3 + 6),
        capture_then_match_tokens!(5));
}
```

Вывод:

```text
got an identifier
got an addition
got something else

got something else
got something else
got something else
```

После разбора входа в узлы AST, подставленный результат становиться *неразрывным*; *т.е.* вы не можете проверить содержимое или найти совпадение с образцом внутри него никогда больше.

Вот *еще* пример, который может особенно смутить:

```rust
macro_rules! capture_then_what_is {
    (#[$m:meta]) => {what_is!(#[$m])};
}

macro_rules! what_is {
    (#[no_mangle]) => {"no_mangle attribute"};
    (#[inline]) => {"inline attribute"};
    ($($tts:tt)*) => {concat!("something else (", stringify!($($tts)*), ")")};
}

fn main() {
    println!(
        "{}\n{}\n{}\n{}",
        what_is!(#[no_mangle]),
        what_is!(#[inline]),
        capture_then_what_is!(#[no_mangle]),
        capture_then_what_is!(#[inline]),
    );
}
```

Вывод:

```text
no_mangle attribute
inline attribute
something else (# [ no_mangle ])
something else (# [ inline ])
```

Единственный способ избежать этого - использовать захват в метапеременные, применяя  `tt` или `ident` типы. Если вы будете использовать захват в метапеременные другого типа, единственное, что вы сможете сделать с результатом - пробросить его прямо на вывод.

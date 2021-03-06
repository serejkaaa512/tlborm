% Замена на повторение

```ignore
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}
```

Этот паттерн представляет случай, когда совпавшая последовательность
отбрасывается, а переменная, представлявшая ее, заменяется на какой-то
повторяющийся паттерн, который подходит по длине.

Например, представим создание экземпляра кортежа, в котором не больше 12
элементов (ограничение для Rust 1.2), и каждому присвоено значение по умолчанию.

```rust
macro_rules! tuple_default {
    ($($tup_tys:ty),*) => {
        (
            $(
                replace_expr!(
                    ($tup_tys)
                    Default::default()
                ),
            )*
        )
    };
}
# 
# macro_rules! replace_expr {
#     ($_t:tt $sub:expr) => {$sub};
# }
# 
# assert_eq!(tuple_default!(i32, bool, String), (0, false, String::new()));
```

> **<abbr title="Только для этого примера">ТДЭП</abbr>**: мы *могли бы* просто
использовать `$tup_tys::default()`.

Здесь, мы на самом деле не *используем* совпадение по типам. Вместо этого, мы
выбрасываем их и заменяем одним, повторяющимся выражением. Говоря по-другому,
нам все равно, *какие* типы используются, нам важно только *сколько* их.

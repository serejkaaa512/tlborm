% Внутренние правила

```rust
#[macro_export]
macro_rules! foo {
    (@as_expr $e:expr) => {$e};

    ($($tts:tt)*) => {
        foo!(@as_expr $($tts)*)
    };
}
# 
# fn main() {
#     assert_eq!(foo!(42), 42);
# }
```

Из-за того, что макросы не подчиняются стандартным правилам приватности или
 доступа через модули, любой публичный макрос *должен* поставляться вместе со всеми
макросами, от которых он зависит. Это можем привести к разрастанию пространства
имен макросов, или даже к конфликтам с макросами из других контейнеров.  Это
может также приводить в замешательство пользователей, которые попытаются
*выборочно* импортировать макросы: они должны последовательно импортировать
*все* макросы, включая те, которые могут быть не задокументированны публично.

Хорошим решением является следующее - скрывать публичные макросы *внутри*
экспортируемого макроса. Пример выше показывает как обычный макрос `as_expr!`
может быть перемещен *внутрь* публично экспортируемого макроса, который
использует его.

Причина, по которой используется `@` - начиная с Rust 1.2, токен `@` *не*
используется в префиксной позиции; таким образом, он не может ни с чем
конфликтовать. Другие символы или уникальные префиксы могут использоваться по
желанию, но использование `@` получило широкое распространение, поэтому
оно может помочь читателям в понимании кода.

> **Помните**: токен `@` раньше использовался в префиксной позиции для
обозначения указателя для сборщика мусора, когда язык
использовал символы для обозначения указателей. *Сейчас* его единственная
цель - связать имя с паттерном.  Для этого, однако, он используется как
*инфиксный* оператор, что не конфликтует с его использованием здесь.

В дополнение, внутренние правила часто идут *до* любых "основных" правил, чтобы
избежать проблем с тем, что `macro_rules!` может попытаться некорректно
разобрать внутренний вызов как что-то, чем оно не может быть, например,
выражение.

Если при экспорте по крайней мере один внутренний макрос невозможно не
экспортировать (*например*, у вас много макросов, которые зависят от общего
набора правил), вы можете использовать паттерн, чтобы объединить *все*
внутренние макросы в один супер-макрос.

```ignore
macro_rules! crate_name_util {
    (@as_expr $e:expr) => {$e};
    (@as_item $i:item) => {$i};
    (@count_tts) => {0usize};
    // ...
}
```

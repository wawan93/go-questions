### array
Коллекция элементов одного типа. `var a [5]int` - массив из 5 элементов типа `int`. 
**Размер массива - это часть типа.** Нельзя передать `[3]int` туда, где требуется `[2]int` и наоборот.

Можно менять элементы и получать их так `arr[i]` (`arr[i] = 'x'`). Если условие `0<i<=len(arr)` **не** выполняется, получим `out of range` панику.
### slicing
Массив (и другой слайс) можно нарезать таким образом `arr[low:high]`, где `low>=0` и `high<len(arr)`. Получим в результате слайс.
### слайс
Это уже коллекция с динамической длиной и capacity. 
### слайс под капотом
```go
type string struct {
	arr unsafe.Pointer
	len int
	cap int
}
```

У слайса есть `len` и `cap`, такие что `len<=cap`, `cap=len(arr)`

[[Нюанс про unsafe.Pointer для слайсов в Golang]]
### append
Добавить элемент в слайс можно функцией `append`. При этом len увеличится на количество добавляемых элементов. Если элементов будет больше чем `cap`, то вызовется внутренняя функция `grow`, будет выделена новая память в 2 раза (с нюансом) большего размера, слайс скопируется в эту новую память, а потом добавятся новые элементы.

[[Нюанс про grow для слайсов в Golang]]
### copy
Есть ещё функция `copy(dst, src)`. Важно, что dst идет первым параметром, а не вторым, как было бы логичнее. И ещё важно, что len обоих слайсов должны совпадать, потому что скопируются только `min(len(dst), len(src))` элементов.
### Как удалить элемент из слайса:
```go
func main() {
    slc := []int{1,2,3,4,5}
    idx := 2
    slc = append(slc[:idx], slc[idx+1:]...)
    fmt.Println(slc)
}
```
```
[1 2 4 5]
```
Более эффективный вариант, если не важна сортировка:
```go
func main() {
    slc := []int{1,2,3,4,5}
    idx := 2
    slc[idx] = slc[len(slc)-1]
    slc = slc[:len(slc)-1]
    fmt.Println(slc)
}
```
```
[1 2 5 4]
```

---
[Arrays, slices (and strings): The mechanics of 'append'](https://go.dev/blog/slices)
[GoLang Slice в деталях, простым языком Николай Тузов — Golang](https://www.youtube.com/watch?v=10LW7NROfOQ)
[Dave Chaney](https://dave.cheney.net/2018/07/12/slices-from-the-ground-up)


## Вопросы с собеседований

1. В чём разница между массивом (`array`) и срезом (`slice`) в Go?
	- **Array** — это значение фиксированного размера, известного во время компиляции. При присваивании или передаче в функцию копируется весь массив.
	- **Slice** — динамический, ссылочный тип, содержащий указатель на «бэк‑энд» массив, длину и вместимость. При копировании среза копируются только эти три поля; сами данные остаются общими.
2. Zero value, `len` и `cap`
	- Нулевое значение среза — `nil` (не указывает ни на какой массив), `len(s)==0 && cap(s)==0`. У массива - аллоцированный массив с заданным количеством элементов, заполненный zero value для того типа, какого типа элементы массива
	- Функции `len()` и `cap()` возвращают текущую длину и вместимость среза/массива. Для массива `cap==len` всегда, для среза — потенциальный максимум до расширения.

## Задачи с собеседований

```go
func main() {
    var x []int
    x = append(x, 1)
    x = append(x, 2)
    x = append(x, 3)
    y := x
    x = append(x, 4)
    y = append(y, 5)
    x[0] = 0

    fmt.Println(x)
    fmt.Println(y)
}
```
Выведет
```
[0 2 3 5]
[0 2 3 5]
```
Обе переменные в итоге указывают на один и тот же подлежащий массив, и последний `x[0] = 0` меняет данные сразу для `x` и `y`

```go
func main() {
    x := []int{1,2,3,4}
    y := x
    x = append(x, 5)
    y = append(y, 6)
    x[0] = 0
    fmt.Println(x)
    fmt.Println(y)
}
```
Выведет
```
[0 2 3 4 5]
[1 2 3 4 6]
```
При первом `append(x,5)` capacity `x` исчерпана и создаётся новый массив; `y` всё ещё ссылается на старый массив.


```go
func main() {
    x := []int{1,2,3,4,5}
    x = append(x, 6)
    x = append(x, 7)
    a := x[4:]
    y := alterSlice(a)
    fmt.Println(x)
    fmt.Println(y)
}

func alterSlice(a []int) []int {
    a[0] = 10
    a = append(a, 11)
    return a
}
```
Выведет
```
[1 2 3 4 10 6 7]
[10 6 7 11]
```
Срез `a` указывает на часть `x`; изменение `a[0]` затрагивает `x`, а `append` расширяет `a` в той же области до `cap(a)`.


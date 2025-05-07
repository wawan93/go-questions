
`map` — это встроенный тип, реализующий хеш-таблицу. Под капотом хранится структура `hmap` с указателем на массив бакетов, счётчиком элементов и метаданными (load factor, seed и т.д.).
```go
// A header for a Go map.
// Note: the format of the hmap is also encoded in
// cmd/compile/internal/gc/reflect.go. Keep in sync.
type hmap struct {
    count      int            // # live cells == size of map. Must be first (used by len() builtin)
    flags      uint8
    B          uint8          // log₂ of # of buckets (can hold up to loadFactor * 2^B items)
    noverflow  uint16         // approximate number of overflow buckets; see incrnoverflow
    hash0      uint32         // hash seed
    buckets    unsafe.Pointer // pointer to array of 2^B bmap buckets; nil if count==0
    oldbuckets unsafe.Pointer // previous bucket array (half size), non-nil only during growing
    nevacuate  uintptr        // progress counter for incremental evacuation
    extra      *mapextra      // optional fields (overflow slices, iterator state, …)
}

// A bucket for a Go map.
type bmap struct {
    // tophash generally contains the top byte of the hash value
    // for each key in this bucket. If tophash[i] < minTopHash,
    // that slot is marked with an evacuation state instead.
    tophash [abi.MapBucketCount]uint8

    // …далее в памяти располагаются подряд:
    // - bucketCnt ключей (keys)
    // - bucketCnt значений (elems)
    // - указатель на overflow-бакет (если этот переполнен)

    // (в исходниках эти поля не описываются напрямую здесь,
    // а получаются через арифметику указателей на основе
    // dataOffset и t.BucketSize)
}
```

**buckets**
Каждый bucket содержит 8 слотов ключ–значение и массив байтов `[8]topHash`, где хранится старшие 8 бит хеша каждого ключа (или метка “пусто”/“удалено”/“эвакуировано”). Это позволяет быстро отсеивать неподходящие слоты 

**Обработка коллизий**
Если в бакете нет свободного слота (`topHash[i] == emptyAll` или `emptyOne`), создаётся цепочка overflow-bucket через поле `overflow` в каждом бакете. При эвакуации все такие цепочки тоже перераспределяются.

**Zero value** (`var m map[K]V`) — `nil`, не указывает ни на какую таблицу.

**Создание**: через `make` или литерал
```go
m1 := make(map[string]int)
m1["a"] = 1

m2 := map[string]int{"a": 1, "b": 2}
```

**Чтение** из `nil`-`map` безопасно: возвращает `V`–zero (для `int` это `0`).
**Запись** в `nil`-`map` приводит к панике `assignment to entry in nil map`.
**Проверка** наличия ключа:
```go
v, ok := m[key] // ok == true, если ключ был в m
```
**Удаление**:
```go
delete(m, key) // молча игнорирует отсутствие ключа
```

**Ключами** могут быть только сравнимые типы: числа, строки, булевы, указатели, структуры/массивы из сравнимых полей, интерфейсы с динамическим типом-ключом; **нельзя** `slice`, `map`, `func`.

**Итерация** (`for k, v := range m`) идёт в **неопределённом** порядке.

При росте `map` (load-factor > 6.5 или избытка overflow-бакетов) Go выделяет новый массив бакетов (обычно вдвое больший) и **эвакуирует** (переносит) записи из старых бакетов в новые. Эвакуация происходит **инкрементально**: при каждой операции вставки или удаления переносится хотя бы один бакет, а флаги в `tophash` (`evacuatedEmpty`, `evacuatedX`, `evacuatedY`) отмечают статус каждого элемента до завершения переноса.

Чтение и итераторы на время эвакуации корректно работают сразу с двумя таблицами (старой и новой).

**Не безопасен** для конкурентного доступа (из-за возможной эвакуации) без внешней синхронизации (`sync.Mutex`/`sync.RWMutex` или `sync.Map`). Будет паника.

Также из-за возможных эвакуаций нельзя получить ссылку на элемент мапы.
```go
package main

import "fmt"

func main() {
    m := map[string]int{"a": 1}
    x := &m["a"] // panic: invalid operation: cannot take address of m["a"] (map index expression of type int)
    
    // так можно
    x := m["a"]
	y := &x     
}
```

В go1.24 будет немного другая реализация, [swiss map](https://github.com/golang/go/blob/master/src/runtime/map_swiss.go) ([тут подробности](https://dev.to/huizhou92/swisstable-a-high-performance-hash-table-implementation-1knc))

---
1. [Как на самом деле устроен тип Map в Golang?](https://www.youtube.com/watch?v=P_SXTUiA-9Y)
2. [Dave Chaney blog](https://dave.cheney.net/2018/05/29/how-the-go-runtime-implements-maps-efficiently-without-generics)
3. [go blog](https://go.dev/blog/maps)
---

## Частые вопросы с собеседований

1. **Что такое `map` в Go?**  
Встроенный reference-тип, реализующий хеш-таблицу с усреднённым доступом O(1).

2. **Как создать `map`?**  
Через `make(map[K]V)` или литерал `map[K]V{…}`.

3. **Каково нулевое значение `map` и как оно ведёт себя при чтении/записи?**  
Нулевое значение — `nil`. Чтение из `nil`-`map` возвращает `V`–zero, а при попытке записи будет panic.

4. **Как проверить, существует ли ключ в `map`?**  
`v, ok := m[key]`, где `ok == true` при наличии ключа, иначе `false` и `v == zero`.

5. **Как удалить элемент из `map`?**  
`delete(m, key)` — удаляет, даже если ключ отсутствует, ошибок не будет.

6. **Какие типы могут использоваться в качестве ключей?**  
Только **сравнимые** (`comparable`) типы; `slice`, `map` и `func` — нельзя.

7. **Каков порядок итерации по `map`?**  
Неопределённый, может меняться при каждом запуске.

---
## Задачки с собеседований

```go
package main

import "fmt"

func main() {
    var m map[string]int       // m == nil
    fmt.Println(m == nil)      // true?
    fmt.Println(m["foo"])      // 0?
    m["foo"] = 1               // panic?
}
```

**Что выведет?**

```
true
0
panic: assignment to entry in nil map
```

---

```go
package main

import "fmt"

func main() {
    m := map[string]int{"a": 1}
    delete(m, "b")
    fmt.Println(len(m))
}
```

**Что выведет?**

```
1
```

`delete` без ошибок игнорирует отсутствие ключа.

---

```go
package main

import "fmt"

func main() {
    m1 := map[string]int{"x": 1}
    m2 := m1
    m2["x"] = 2
    fmt.Println(m1["x"])
}
```

**Что выведет?**

```
2
```

`map` — ссылочный тип, `m1` и `m2` указывают на один объект.
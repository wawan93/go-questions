`error` — встроенный интерфейс:

```go
type error interface {
	Error() string
}
```

- Любой тип, реализующий `Error() string`, является `error`.
- `fmt.Errorf`, `errors.New` возвращают `error`.
- **Обёртка ошибок**: Go 1.13+ поддерживает `Unwrap`, `Is`, `As`.
- **Паттерны**:
- `errors.Is(err, target)`
- `errors.As(err, &target)`
- Обёртка: `%w` в `fmt.Errorf`

### Частые вопросы про error

- Чем `errors.New` отличается от `fmt.Errorf`?
- Как создать пользовательскую ошибку с контекстом?
- Что такое wrapping (`%w`)?
- Как использовать `errors.Is` и `errors.As`?

### Задача "error" на собеседовании

```go

var ErrNotFound = errors.New("not found")

func check() error {
	return fmt.Errorf("checking: %w", ErrNotFound)
}

func main() {
	err := check()

	if err == ErrNotFound {
		fmt.Println("direct match")
	} else if errors.Is(err, ErrNotFound) {
		fmt.Println("wrapped match")
	}
}

```

**Ответ:** wrapped match
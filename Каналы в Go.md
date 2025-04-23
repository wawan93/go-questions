- A send to a nil channel blocks forever
- A receive from a nil channel blocks forever
- A send to a closed channel panics
- A receive from a closed channel returns the zero value immediately

---
[Как на самом деле устроены каналы в Golang | Николай Тузов](https://www.youtube.com/watch?v=ZTJcaP4G4JM)
[effective go](https://go.dev/doc/effective_go#channels)
[мастрид про аксиомы от Dave Chaney](https://dave.cheney.net/2014/03/19/channel-axioms)

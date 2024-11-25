## Процессы, потоки, горутины


## Планировщик


## Барьеры памяти


## Мьютексы


Инкрементация не работает корректно.
```golang
package main

import (
	"fmt"
	"sync"
)

const gorutinesNumber = 1000

func main() {
	wg := sync.WaitGroup{}
	wg.Add(gorutinesNumber)

	value := 0
	for i := 0; i < gorutinesNumber; i++ {
		go func() {
			defer wg.Done()
			value++
		}()
	}
	wg.Wait()
	fmt.Println(value)
}

```

Инкремент: value++
Аналогично:
oldValue := value
newValue := oldValue + 1
value = newValue

=> другая горутина между этими инструкциями может встроиться (например, перезапись oldValue локальным значением другой горутины)
=> аномалия: потеренное обновление

Мьютекс - примитив синхронизации, обеспечивающий взаимное исключение исполенения критических участков кода.

```golang

type Mutex struct {...}

func (m *Mutex) Lock()            // захватить мьютекс
func (m *Mutex) Unlock()          // освободить мьютекс
func (m *Mutex) TryLock() bool    // попробовать захватить мьютекс

```
Пример использования мьютекса
```golang
package main

import (
	"fmt"
	"sync"
)

const gorutinesNumber = 1000

func main() {
	mutex := sync.Mutex{}
	wg := sync.WaitGroup{}
	wg.Add(gorutinesNumber)

	value := 0
	for i := 0; i < gorutinesNumber; i++ {
		go func() {
			defer wg.Done()

			mutex.Lock() // остальные горутины подождут на входе в критическую секцию
			value++ // инкремент внутри захвата примитива синхронизации (критическая секция - в этот промежуток кода может попасть только одна горутина)
			mutex.Unlock()
		}()
	}
	wg.Wait()
	fmt.Println(value)
}
```

### Состояния горутин:
Waiting (ожидает наступления события), Running (выполняется на каком-то ядре процессора), Ready (готова к исполнению)

Когда горутина блокируется на мьютексе, она переходит из состояния Running в Waiting.


### Гарантии при написании собственных мьютексов
Взаимное исключение (safety) - между парными вызывами Lock и Unlock может находиться только одна горутина.
Прогресс (liveness) - если одна или несколько горутин пытаются захватить мьютекс, то один из вызовов Lock должен завершиться успешно (горутины не должны бесконечно долго мешать друг другу)

### Сломанный мьютекс

```golang
package main

import (
	"fmt"
	"sync"
)

const (
	unlock = false
	locked = true
)

type BrokenMutex struct {
	state bool
}

// data race и нет взаимного исключения

func (m *BrokenMutex) Lock() {
	for m.state { // пока кто-то владеет мьютексом
		// iteration by iteration
	}
	m.state = locked
}

func (m *BrokenMutex) Unlock() {
	m.state = unlock
}

const gorutinesNumber = 1000

func main() {
	var mutex BrokenMutex
	wg := sync.WaitGroup{}
	wg.Add(gorutinesNumber)

	value := 0
	for i := 0; i < gorutinesNumber; i++ {
		go func() {
			defer wg.Done()

			mutex.Lock() // из-за оптимизации компилятор может переупорядочивать код -> нужны атомарные операции, так как они внутри себя используют барьеры памяти
			value++
			mutex.Unlock()
		}()
	}
	wg.Wait()
	fmt.Println(value)
}
```


### Пример атомика

```golang

type Bool struct {...}

func (m *Bool) CompareAndSwap(old, new bool) (swapped bool)
func (m *Bool) Load() bool // загрузить из памяти атомарно
func (m *Bool) Store(val bool) // сохранить в память атомарно
func (m *Bool) Spaw(new bool) (old bool)

```
### Сломанный мьютекс с атомиками (теперь код не переупорядочивается, но нет safety)

```golang
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

const (
	unlocked = false
	locked   = true
)

type BrokenMutex struct {
	state atomic.Bool
}

// нет взаимного исключения

func (m *BrokenMutex) Lock() {
	for m.state.Load() { // одна горутина может загрузить текущее значение
		// iteration by iteration
	}
	// и потом здесь прерваться, а потом другая заходит в критическую область и в этом же месте прерывается
	m.state.Store(locked) // и потом они обе меняют это значение
}

func (m *BrokenMutex) Unlock() {
	m.state.Store(unlocked)
}

const gorutinesNumber = 1000

func main() {
	var mutex BrokenMutex
	wg := sync.WaitGroup{}
	wg.Add(gorutinesNumber)

	value := 0
	for i := 0; i < gorutinesNumber; i++ {
		go func() {
			defer wg.Done()

			mutex.Lock() // из-за оптимизации компилятор может переупорядочивать код -> нужны атомарные операции, так как они внутри себя используют барьеры памяти
			value++
			mutex.Unlock()
		}()
	}
	wg.Wait()
	fmt.Println(value)
}

```
![изображение](https://github.com/user-attachments/assets/f4332e4e-1754-465b-9a64-a31b96f403f5)

Следовательно, нужны RMW (Read-Modify-Write) операции для атомарного чтения-записи.

### CAS (COMPARE-AND-SWAP)

```golang
if *addr == old {
	*addr = new
	return true
}
return false
```

### Мьютекс, основанный на spin lock

```golang
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

const (
	unlocked = false
	locked   = true
)

type Mutex struct {
	state atomic.Bool
}

func (m *Mutex) Lock() {
	for !m.state.CompareAndSwap(unlocked, locked) { // если состояние действительно unlocked, то будет переведено в locked и вернется true
		// BUSY WAITING цикл активного ожидания (ждем, пока другая горутина не отпустит спинлок)
	}
}

func (m *Mutex) Unlock() {
	m.state.Store(unlocked)
}

const gorutinesNumber = 1000

func main() {
	var mutex Mutex
	wg := sync.WaitGroup{}
	wg.Add(gorutinesNumber)

	value := 0
	for i := 0; i < gorutinesNumber; i++ {
		go func() {
			defer wg.Done()

			mutex.Lock()
			value++
			mutex.Unlock()
		}()
	}
	wg.Wait()
	fmt.Println(value)
}

```

### Как освободиться от активного ожидания
1. time.Sleep(...)
   Мьютексы могут освободить раньше, но мы об этом не узнаем, а таже непонятно, сколько нужно спать.
2. PAUSE
   Улучшает спинлуп.
   Гипертрединг за счет логических ядер в рамках ОС. В ЛЯ есть два набора разных регистров, и ядро может между ними переключаться. PAUSE условно говорит ядру переключиться на другие регистры и начать выполнять другой поток.

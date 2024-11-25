## Процессы, потоки, горутины


## Планировщик


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






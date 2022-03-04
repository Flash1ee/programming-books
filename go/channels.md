# Конспект по каналам
Каналы - средства синхронизации в Го, с помощью которых взаимодействуют горутины.  
Горутины - легковесные потоки(4 кб одна горутина, поток ОС - минимум 1 Мб). 

## Типы каналов
- Однонаправленный
- Двунаправленный


```go
package main

func main() {
  c1 := make(chan int)
  c2 := make(<-chan int) // только на чтение
  c3 := make(chan<- int) // только на запись
}
```

Из доки по go:
`<-` применяется к крайнему левому `chan`
```
The <- operator associates with the leftmost chan possible:

chan<- chan int    // same as chan<- (chan int)
chan<- <-chan int  // same as chan<- (<-chan int)
<-chan <-chan int  // same as <-chan (<-chan int)
chan (<-chan int)
```


## Взаимодействие с каналами 
| Операция           | Nil канал      | Closed канал    | Not-Closed и Non-Nil канал  |
|--------------------|----------------|-----------------|-----------------------------|
| Закрыть канал      | panic          | panic           | succeed to close            |
| Отправить в канал  | block          | panic           | block or succeed to send    |
| Получить из канала | block for ever | never block | block or succeed to receive |

## Примитивы синхронизации, пакет `sync`
[Оф. дока](https://pkg.go.dev/sync)
### Map (sync.Map)
```go
type Map struct {
	// contains filtered or unexported fields
}
```
Зачем использовать, если есть простая map?  
Тип `sync.Map` оптимизирован для двух случаев использования:
1. когда запись для данного ключа записывается только один раз, но читается много раз, как в кэшах, которые только увеличиваются;
2. когда несколько горутин читают, записывают и перезаписывать записи для непересекающихся наборов ключей. В этих двух случаях использование `sync.Map` может значительно уменьшить количество конфликтов за блокировку по сравнению с map Go в паре с отдельным Mutex или RWMutex.


### Mutex
```go
type Mutex struct {
	// contains filtered or unexported fields
}
func (m *Mutex) Unlock()
func (m *Mutex) Lock()
```
- Mutex нельзя копировать после первого использования!  

### RWMutex
```go
type RWMutex struct {
	// contains filtered or unexported fields
}
func (rw *RWMutex) Lock() // Блокировка на запись
func (rw *RWMutex) RLock()  // Блокировка на чтение
func (rw *RWMutex) RLocker() Locker // Locker на запись, поддерживает методы Lock и Unllock()
func (rw *RWMutex) RUlock() Locker // Locker на чтение, поддерживает методы Rlock b RUnlock()
func (rw *RWMutex) Unlock() // Разблокировка на чтение
```
Похож на `sync.Mutex`, но поддерживает возможность создавать блокировки только на чтение.  
Блокировка может удерживаться произвольным числом читателей или одним писателем.   
- Если сделать `Unlock` на незаблокированный mutex, произойдет `panic`!!  
```go
package main

import (
	"sync"
)

func main() {
	mu := sync.RWMutex{}
	mu.Lock()
	mu.Unlock()
	mu.Unlock()
}
fatal error: sync: Unlock of unlocked RWMutex
```
- Lock уводит горутину в сон, пока не произойдет Unlock => если этого не произойдет, то будет `deadlock`;  

### Once
```go
type Once struct {
	// contains filtered or unexported fields
}
func (o *Once) Do(f func()) // Вызов функции f, причем. только 1 раз
```
Once - объект, который будет выполнять ровно одно действие.  
После первого использования нельзя копировать!.  
Пример:  
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}
// Вывод: Only once
```
### Pool
Нужен для повторного использования объектов без нового выделения памяти.  
После первого использования нельзя копировать!.  

```go
type Pool struct {

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
	// contains filtered or unexported fields
}
func (p *Pool) Put(x interface{})
func (p *Pool) Get() interface{}
```
Метод `New` используется, если в пуле нет значений. Если он не определен, то возвращается `nil`;  
Пример:  
```go
package main

import (
	"bytes"
	"io"
	"os"
	"sync"
	"time"
)

var bufPool = sync.Pool{
	New: func() interface{} {
		// The Pool's New function should generally only return pointer
		// types, since a pointer can be put into the return interface
		// value without an allocation:
		return new(bytes.Buffer)
	},
}

// timeNow is a fake version of time.Now for tests.
func timeNow() time.Time {
	return time.Unix(1136214245, 0)
}

func Log(w io.Writer, key, val string) {
	b := bufPool.Get().(*bytes.Buffer)
	b.Reset()
	// Replace this with time.Now() in a real logger.
	b.WriteString(timeNow().UTC().Format(time.RFC3339))
	b.WriteByte(' ')
	b.WriteString(key)
	b.WriteByte('=')
	b.WriteString(val)
	w.Write(b.Bytes())
	bufPool.Put(b)
}

func main() {
	Log(os.Stdout, "path", "/search?q=flowers")
}
```
### WaitGroup
```go
type WaitGroup struct {
	// contains filtered or unexported fields
}
func (wg *WaitGroup) Add(delta int)
func (wg *WaitGroup) Done()
func (wg *WaitGroup) Wait()
```
WaitGroup - примитив синхронизации, который позволяет контролировать завершение горутин.    
Основная горутина вызывает `Add`, чтобы установить количество ожидаемых горутин. Затем запускается каждая горутина и по завершении вызывает `Done`. В то же время, `Wait` используется для блокировки(ожидания), пока не закончатся все горутины.

`WaitGroup` не должна быть скопирована после первого использования.  

Пример:  
```go
package main

import (
	"sync"
)

type httpPkg struct{}

func (httpPkg) Get(url string) {}

var http httpPkg

func main() {
	var wg sync.WaitGroup
	var urls = []string{
		"http://www.golang.org/",
		"http://www.google.com/",
		"http://www.somestupidname.com/",
	}
	for _, url := range urls {
		// Increment the WaitGroup counter.
		wg.Add(1)
		// Launch a goroutine to fetch the URL.
		go func(url string) {
			// Decrement the counter when the goroutine completes.
			defer wg.Done()
			// Fetch the URL.
			http.Get(url)
		}(url)
	}
	// Wait for all HTTP fetches to complete.
	wg.Wait()
}
```
## Примитивы синхронизации, пакет `sync/atomic`
[Оф. дока](https://pkg.go.dev/sync/atomic). 
Пакет atomic предоставляет низкоуровневые примитивы атомарной памяти, полезные для реализации алгоритмов синхронизации.  

- `mutex vs atomic` [c++ разбор и сравнение](https://coffeebeforearch.github.io/2020/08/04/atomic-vs-mutex.html). 
Mutex и atomic используются работы с одними и теми же данными в разных потоках. Основное отличие - mutex использует блокировки на уровне приложениия(в ЯП реализованы механизмы блокировок), atomic - использует механизмы на уровне процессора.  
Примеры инкремента на уровне ассебмлерного кода:  

Mutex
```asm
       │20:┌─→mov   %r12,%rdi
 22.62 │   │→ callq pthread_mutex_lock@plt
  2.51 │   │  test  %eax,%eax
       │   │↓ jne   60
 49.50 │   │  incq  0x0(%rbp)
  0.75 │   │  mov   %r12,%rdi
 11.33 │   │→ callq pthread_mutex_unlock@plt
       │   ├──dec   %ebx
 13.29 │   └──jne   20
```
Atomic
```asm
 98.89 │10:   lock   incq (%rdx) //делает операцию неделимой
  0.03 │      dec    %eax
  1.08 │    ↑ jne    10
```

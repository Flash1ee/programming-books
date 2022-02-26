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

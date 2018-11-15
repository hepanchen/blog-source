---
title: "Golang的id锁实现"
date: 2018-11-15T18:04:18+08:00
draft: true
---

![](golang的id锁实现-be145.png)

# 需求
不同的协程进行分组，只有分组相同的协程需要并发控制。（非原创）

---

## 方案1 mutex+cond+map

```go
package kmutex

import (
   "fmt"
   "sync"
)

type KMutex interface {
   Lock(key interface{})
   UnLock(key interface{})
}

var empty struct{}
/*
* map+cond的实现，缺点容易引起goroutine的惊群现象，key比较少，
* 等待key的routine比较多时，可能会有性能问题
*/
type kMutex struct {
   l *sync.Mutex       //使用指针是因为mutex不能复制，否则起不到锁的效果
   cond *sync.Cond
   mtxMap map[interface{}]struct{}    //使用空struct大小为0节约空间
}

func NewKmutex() KMutex {
   l := new(sync.Mutex)
   return &kMutex{l: l, cond:sync.NewCond(l),
      mtxMap:make(map[interface{}]struct{}),}
}

func(k kMutex)locked(key interface{}) (ok bool){
   _, ok = k.mtxMap[key]
   return
}

func (k kMutex)Lock(key interface{})  {
   k.l.Lock()
   for k.locked(key) {
      k.cond.Wait()
   }
   k.mtxMap[key] = empty
   k.l.Unlock()
}

func (k kMutex)UnLock(key interface{})  {
   k.l.Lock()
   if _, ok := k.mtxMap[key]; ok {
      delete(k.mtxMap, key)
      k.cond.Broadcast()
   }else {
      fmt.Println(k.mtxMap)
      panic("unlock a unlocked lock")
   }
   k.l.Unlock()
}
```
---

## 方案2 sync.map
```go
package kmutex

import (
   "fmt"
   "sync"
)

type KMutex interface {
   Lock(key interface{})
   UnLock(key interface{})
}

var empty struct{}
/*
* sync.map 实现 依然会有大量协程被唤醒，但不是同时唤醒
* 没有惊群影响大，不过应该获得锁的线程可能不能获得锁,需要多次尝试
*/
type mapKMutex struct {
   s *sync.Map
}

func NewMapKmutex() KMutex {
   return &mapKMutex{
      s: &sync.Map{},
   }
}

func (mk mapKMutex)Lock(key interface{}) {
   m := &sync.Mutex{}
   mLock, _ := mk.s.LoadOrStore(key, m)
   mLock.(*sync.Mutex).Lock()
   if mLock.(*sync.Mutex) != m {
      mLock.(*sync.Mutex).Unlock()//如果获取了别人存入的锁，重试
      mk.Lock(key)
      return
   }
   return
}

func (mk mapKMutex)UnLock(key interface{}) {
   if mLock, ok := mk.s.Load(key); !ok {
      panic("unlock an unlocked lock")
   } else {
      mk.s.Delete(key)
      mLock.(*sync.Mutex).Unlock()
   }
}

```
---
## 测试
```go
func main(){
m := kmutex.NewKmutex()
//m := kmutex.NewMapKmutex()
sum := 0
incr := func(key int) {
  for i := 0;i<1000;i++ {
    m.Lock(key)
    sum++
    m.UnLock(key)
  }
}
for i:=0;i<10000;i++ {
  go incr(1)
}

for ; ;  {
  time.Sleep(time.Second)
  fmt.Println(sum)
}
}
```


* 经过测试发现当协程持有不同的key的时候，会有并发问题，得到的结果不是预期值。
* 当持有相同的key时，结果符合预期。
* 在组内竞争比较大的情况，使用sync.map的方案比使用cond的方案性能好很多。

## 参考资料
https://medium.com/@petrlozhkin/kmutex-lock-mutex-by-unique-id-408467659c24

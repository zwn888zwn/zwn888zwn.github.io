---
layout    : post
title     : "go RWLock写优先"
date      : 2021-08-04
lastupdate: 2021-08-04
categories: go
---

 go RWLock

从源码注释和实验结果证明，go中读写锁是写优先的。即在已获取读锁的情况下，尝试获取写锁，后续的读锁也会一起阻塞，直到写锁释放。

![](/assets/img/2021-07-15-go-overide-method-zh/16690820013851.jpg)

测试读写锁代码：

```go
func TestRWLock(t *testing.T) {
	var lock sync.RWMutex
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		fmt.Println("try rlock 1")
		lock.RLock()
		defer lock.RUnlock()
		fmt.Println("get rlock 1")

		func() {
			time.Sleep(10 * time.Second)
			fmt.Println("try rlock 2")
			lock.RLock()
			defer lock.RUnlock()
			fmt.Println("get rlock 2")
		}()
		
		wg.Done()
	}()

	go func() {
		time.Sleep(4 * time.Second)
		fmt.Println("try wlock")
		lock.Lock()
		defer lock.Unlock()
		fmt.Println("get wlock")
		wg.Done()
	}()

	wg.Wait()
	fmt.Println("finish")
}
```

输出结果：

```
try rlock 1
get rlock 1
try wlock
try rlock 2
```




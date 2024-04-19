---
layout    : post
title     : "go怎么处理千万级数据"
date      : 2024-04-18
lastupdate: 2024-04-18
categories: go
---

----

## 读文件方法对比

### 将整个文件读入内存

readFullFile(file)

一次将整个文件读入内存并处理文件，这将消耗更多内存，如果文件过大会显着增加时间。

```go
func main() {
	// 读取整个文件内容
	content, err := ioutil.ReadFile("example.txt")
	if err != nil {
		log.Fatal(err)
	}

	// 将文件内容按行分割成字符串数组
	lines := strings.Split(string(content), "\n")

	// 遍历字符串数组，输出每一行内容
	for _, line := range lines {
		fmt.Println(line)
	}
}
```

由于我们的文件大小太大，即16 GB，我们无法将整个文件加载到内存中。但是第一个选项对我们来说也不可行，因为我们希望在几秒钟内处理文件。

```go
Benchmark/Sequential_000_workers_0000_batchSize-8                      1        197251508333 ns/op      17140040864 B/op        43544146 allocs/op
PASS
ok      github.com/snassr/blog-0010-processinglargefilesingo    197.898s

```



### 逐行buffer读取文件

bufioScan(file)

```go
func main() {
    filename := "your_file.txt" // 替换为实际文件名
    file, err := os.Open(filename)
    if err != nil {
        fmt.Println("Error opening file:", err)
        return
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        line := scanner.Text()
        fmt.Println(line)
    }

    if err := scanner.Err(); err != nil {
        fmt.Println("Error reading file:", err)
    }
}
```

`Scanner`会将整行文本加载到内存中，然后逐行处理。如果文件中有特别长的行，它可能会导致内存占用过高。`Scanner`的缓冲区大小为64KB，如果一行超过这个大小，它将无法正确处理。

```go
Benchmark/Sequential_000_workers_0000_batchSize-8                      1        16439025625 ns/op       12646394264 B/op        65274099 allocs/op
PASS
ok      github.com/snassr/blog-0010-processinglargefilesingo    16.576s
```



#### 调大缓冲区（较好）

bufioBigBuf(file)

```go
const maxCapacity = 1024 * 1024  // 缓冲大小，1MB，可读取任何 1MB 的行。
buf := make([]byte, maxCapacity) // 
scanner.Buffer(buf, maxCapacity)
```

bufio默认缓冲区大小为4KB，调大缓冲区后，减少了系统调用次数，耗时缩短了些

```go
Benchmark/Sequential_000_workers_0000_batchSize-8                      1        16792737333 ns/op       12646410976 B/op    65274179 allocs/op
PASS
ok      github.com/snassr/blog-0010-processinglargefilesingo    15.889s
```



### mmap读文件

mmapRead(file)

```go
// Provide the filename & let mmap do the magic
	r, err := mmap.Open("text.txt")
	if err != nil {
		panic(err)
	}

	//Read the entire file in memory at an offset of 0 => the entire thing
	p := make([]byte, r.Len())
	_, err = r.ReadAt(p, 0)
	if err != nil {
		panic_(err)
	}

	// print the file
	fmt.Println(string( p ))
```

修改p一次读取大小为64MB的block分段读取，读取时间和占用内存要比一次读入整个文件要好，但都比原生IO要高

```go
Benchmark/Sequential_000_workers_0000_batchSize-8                      1        38172455750 ns/op       17152662584 B/op    43544770 allocs/op
PASS
ok      github.com/snassr/blog-0010-processinglargefilesingo    38.900s
```

查看源码得知，`mmap.Open`一次性把整个文件mmap，尝试一次只mmap若干个页，并处理好边角，执行数据处理逻辑，最后Munmap，循环。为了避免复制内容过多（mmapReadV2(file)-mmapReadV3(file)的优化），每次只从字符数组拷贝一个string出来处理，但顺序读取耗时还是比原生IO多

```
Benchmark/Sequential_000_workers_0000_batchSize-8                      1        25932430125 ns/op       12646436360 B/op        65274409 allocs/op
PASS
ok      github.com/snassr/blog-0010-processinglargefilesingo    26.341s
```





[Go mmap 文件内存映射_golang mmap-CSDN博客](https://blog.csdn.net/baidu_32452525/article/details/131313142)

[阅读笔记：零拷贝及一些引申内容 - 掘金](https://juejin.cn/post/6854573213452599310#heading-5)

[Exploring MMAP In Go](https://ghvsted.com/blog/exploring-mmap-in-go/)

[Linux 零拷贝技术-mmap与sendFile - hongdada - 博客园](https://www.cnblogs.com/hongdada/p/16926179.html)

## pipeline方式处理

上面讲的是顺序读文件效率的比较。再结合流水线的方式，最大化程度利用cpu，边读边解析

![image-20240415163805788](/assets/img/2024-04-18-process-large-data-in-go-zh/image-20240415163805788.png)

**data file** →reader →processor(s) →combiner → **result**



## 测试数据

测试代码[processinglargefilesingo/main_test.go at main · zwn888zwn/processinglargefilesingo](https://github.com/zwn888zwn/processinglargefilesingo/blob/main/main_test.go#L136)

测试数据直接借用github里的data，解压后4个G，2000W行



## 参考链接

[snassr/blog-0010-processinglargefilesingo](https://github.com/snassr/blog-0010-processinglargefilesingo)

[硬核，图解bufio包系列之读取原理](https://mp.weixin.qq.com/s?__biz=MzkyMjE5Mzg3Nw==&mid=2247486295&idx=3&sn=a253da7689f8d12bcc1c315d866c63a5&chksm=c1f951edf68ed8fb2883332a029ece284534c9a7b3d6f2292167466b2805a2001844aa0d1d54#rd)



[Processing Large Files with Go (Golang) | by snassr | Medium](https://medium.com/@snassr/processing-large-files-in-go-golang-6ea87effbfe2)

[bufio — 缓存 IO · Go语言标准库](https://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter01/01.4.html)



[bufio包系列之一个误用bufio读取的示例-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2211939?areaId=106001)

[Reading 16GB File in Seconds, Golang | by Ohm Patel | The Startup | Medium](https://medium.com/swlh/processing-16gb-file-in-seconds-go-lang-3982c235dfa2)

[Go 如何按行读取（大）文件？尝试 bufio 包提供的几种方式 - 掘金](https://juejin.cn/post/7338611185190977599#heading-7)

[读入数据 · Go语言中文文档](https://www.topgoer.com/项目/千万数据过滤/读入数据.html)

---
title: Go profile
date: 2019-08-23
private:
---
# Go profile

### Go profiler 指标
使用Go 内置的profiler我们能获取以下的样本信息：

    CPU profiles
    Heap profiles
    block profile、traces等

### go profiling常用的分析工具
我了解的有以下几种工具
1. `go test` 基准测试文件：比如使用命令`go test . -bench=. -cpuprofile prof.cpu` 生成采样文件后，再通过命令 `go tool pprof [binary] prof.cpu` 来进行分析。(有点类似php的xhprof 的调用关系图，调用关系复杂就不直观了)

2. `import _ net/http/pprof`：如果我们的应用是一个web服务，我们可以在http服务启动的代码文件(eg: main.go)添加 import _ net/http/pprof，这样我们的服务 便能自动开启profile功能，有助于我们直接分析采样结果。

3. `runtime` 工具：通过在代码里面调用 `runtime.StartCPUProfile`或者`runtime.WriteHeapProfile` 生成分析工具
4. 官方的`pkg/profile` 这个下文细说

更多调试的使用，可以阅读The Go Blog的 Profiling Go Programs

## go profiler 工具
本节参考：https://wjp2013.github.io/go/go-tools-basic/

准备下要分析的代码:

    import (
        "time"
        "github.com/pkg/profile"
    )

    func joinSlice() []string {
        var arr []string
        for i := 0; i < 10000; i++ {
            arr = append(arr, "arr")
        }
        return arr
    }

    func main() {
        stopper := profile.Start(profile.CPUProfile, profile.ProfilePath("."))

        defer stopper.Stop()

        joinSlice()

        time.Sleep(time.Second)
    }

用 go tool 工具链输出 pdf 格式文件：

    go build -o cpu cpu.go
    ./cpu
    go tool pprof --pdf cpu cpu.pprof > cpu.pdf

## go-torch 火焰
火焰图是一个非常直观的查找性能瓶颈点

### 安装go-torch
go-torch 输出的数据可以用FlameGraph 脚本实现可视化(FlameGraph 是profile数据的可视化层工具)

我们先配置FlameGraph的脚本

    git clone https://github.com/brendangregg/FlameGraph.git
    cp flamegraph.pl ~/bin
    # 查看帮助命令
    flamegraph.pl -h
    USAGE: /usr/local/bin/flamegraph.pl [options] infile > outfile.svg

安装go-torch很简单(用于go-torch展示profile的输出)

    go get -v github.com/uber/go-torch
    $ go-torch -h
    Usage:
    go-torch [options] [binary] <profile source>

### 调优代码实例
先准备实例代码：
1. 下载demo [地址1](https://github.com/domac/playflame/tree/slow) / [地址2](https://github.com/ahuigo/playflame/tree/slow) 
2. 运行server 端：$ go run main.go -printStats

### pprof profile 生成调用关系图(不直观)
接下来我们用go-wrk （或者ab、siege）压测advanced 接口

    go-wrk  -n=100000 -c=500  http://localhost:9090/advance

在上面的压测过程中，我们再新建一个终端窗口输入以下命令，生成我们的profile文件：

    $ go tool pprof --seconds 10 http://localhost:9090/debug/pprof/profile

命令中，我们设置了10秒的采样时间，当看到(pprof)的时候，我们输入 web, 表示从浏览器打开

    Fetching profile from http://localhost:9090/debug/pprof/profile?seconds=25
    Please wait... (25s)
    Saved profile in /Users/ahui/pprof/pprof.localhost:9090.samples.cpu.014.pb.gz
    Entering interactive mode (type "help" for commands)
    (pprof) web

这样我们可以得到一个完整的程序调用性能采样profile的输出,如下图：
![](/img/go/profile/pprof-simple.png)

调用图太不直观了，我们需要简单的火焰图

### Flame 图
先压测：

    $ go-wrk  -n=100000 -c=500  http://localhost:9090/advance

压测过程中用 go-torch来生成采样报告, 30s后出报告:

    $ go-torch -u http://localhost:9090 -t 30
    INFO[08:47:10] Run pprof command: go tool pprof -raw -seconds 30 http://localhost:9090/debug/pprof/profile
    INFO[08:47:41] Writing svg to torch.svg

火焰图的y轴表示cpu调用方法的先后，x轴表示在每个采样调用时间内，方法所占的时间百分比，越宽代表占据cpu时间越多. 

![](/img/go/profile/flame1.png)
![](/img/go/profile/flame2.png)


### bench cpu
我们来压测下stats 这个目录

    $ cd stats
    $ go test -bench . -benchmem -cpuprofile prof.cpu
    BenchmarkAddTagsToName-4   	 1000000	      2138 ns/op	     487 B/op	      18 allocs/op

注意2138ns/op ，说明很慢

    $ go tool pprof stats.test  prof.cpu
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) top10
      flat  flat%   sum%        cum   cum%
      90ms 10.34% 10.34%       90ms 10.34%  regexp/syntax.(*Inst).MatchRunePos
      90ms 10.34% 20.69%      240ms 27.59%  runtime.mallocgc
      80ms  9.20% 29.89%      240ms 27.59%  regexp.(*Regexp).tryBacktrack
      50ms  5.75% 35.63%       50ms  5.75%  runtime.heapBitsSetType
      50ms  5.75% 41.38%       50ms  5.75%  runtime.nextFreeFast
      50ms  5.75% 47.13%      860ms 98.85%  stats.addTagsToName
      40ms  4.60% 51.72%      580ms 66.67%  regexp.(*Regexp).ReplaceAllString
      40ms  4.60% 56.32%       40ms  4.60%  regexp.(*inputString).step
      30ms  3.45% 59.77%      380ms 43.68%  regexp.(*Regexp).backtrack
      30ms  3.45% 63.22%       50ms  5.75%  regexp.(*bitState).push

从排行榜看到，大概regexp很大关系，但这不好看出真正问题，需要再用别的招数

我们在(pprof)后，输入list addTagsToName， 分析基准测试文件中具体的方法

    (pprof) list addTagsToName
    Total: 870ms
    ROUTINE ======================== stats.addTagsToName in /Users/ahui/go/src/github.com/ahuigo/playflame/stats/reporter.go
        50ms      860ms (flat, cum) 98.85% of Total
            .          .     34:}
            .          .     35:
            .          .     36:func addTagsToName(name string, tags map[string]string) string {
            .          .     37:	var keyOrder []string
            .          .     38:	if _, ok := tags["host"]; ok {
        20ms       40ms     39:		keyOrder = append(keyOrder, "host")
            .          .     40:	}
            .       50ms     41:	keyOrder = append(keyOrder, "endpoint", "os", "browser")
            .          .     42:
        10ms       10ms     43:	parts := []string{name}
            .          .     44:	for _, k := range keyOrder {
            .       20ms     45:		v, ok := tags[k]
            .          .     46:		if !ok || v == "" {
            .          .     47:			parts = append(parts, "no-"+k)
            .          .     48:			continue
            .          .     49:		}
        20ms      710ms     50:		parts = append(parts, clean(v))
            .          .     51:	}
            .          .     52:
            .       30ms     53:	return strings.Join(parts, ".")
            .          .     54:}
            .          .     55:
            .          .     56:var specialChars = regexp.MustCompile(`[{}/\\:\s.]`)
            .          .     57:
            .          .     58:func clean(value string) string {

可以看到两个时间: selfTime(自身的时间) cumTime. 说明clean 占用时间最多

    (pprof) list clean
    Total: 870ms
    ROUTINE ======================== stats.clean in /Users/hilojack/go/src/github.com/ahuigo/playflame/stats/reporter.go
        10ms      590ms (flat, cum) 67.82% of Total
            .          .     54:}
            .          .     55:
            .          .     56:var specialChars = regexp.MustCompile(`[{}/\\:\s.]`)
            .          .     57:
            .          .     58:func clean(value string) string {
        10ms      590ms     59:	return specialChars.ReplaceAllString(value, "-")
            .          .     60:}

既然正则最慢，那我们还是用普通字符替换吧！

    func clean(value string) string {
        newStr := make([]byte, len(value))
        for i := 0; i < len(value); i++ {
            switch c := value[i]; c {
            case '{', '}', '/', '\\', ':', ' ', '\t', '.':
                newStr[i] = '-'
            default:
                newStr[i] = c
            }
        }
        return string(newStr)
    }

再压测，性能提升到765ns/op 了

    $ go test -bench . -benchmem -cpuprofile prof.cpu
    BenchmarkAddTagsToName-4   	 2000000	       765 ns/op	     400 B/op	      14 allocs/op

注意我们的内存分配次数14allocs/op, 下面我们再优化下mem

### bench mem
生成memProfile 

    go test -bench . -benchmem -memprofile prof.mem
    14 allocs/op

分析mem:

    $ go tool pprof --alloc_objects  stats.test prof.mem
    Entering interactive mode (type "help" for commands)
    (pprof) top
    Showing nodes accounting for 27739275, 100% of 27739275 total
          flat  flat%   sum%        cum   cum%
      14992237 54.05% 54.05%   27739275   100%  stats.addTagsToName
       9732244 35.08% 89.13%    9732244 35.08%  stats.clean
       3014794 10.87%   100%    3014794 10.87%  strings.(*Builder).grow

    (pprof) list addTagsToName
    Total: 27739275
    ROUTINE ======================== stats.addTagsToName in /Users/hilojack/go/src/github.com/ahuigo/playflame/stats/reporter.go
      14992237   27739275 (flat, cum)   100% of Total
             .          .     34:func addTagsToName(name string, tags map[string]string) string {
             .          .     35:	var keyOrder []string
             .          .     36:	if _, ok := tags["host"]; ok {
       2621480    2621480     37:		keyOrder = append(keyOrder, "host")
             .          .     38:	}
       2916530    2916530     39:	keyOrder = append(keyOrder, "endpoint", "os", "browser")
             .          .     40:
             .          .     41:	parts := []string{name}
             .          .     42:	for _, k := range keyOrder {
             .          .     43:		v, ok := tags[k]
             .          .     44:		if !ok || v == "" {
             .          .     45:			parts = append(parts, "no-"+k)
             .          .     46:			continue
             .          .     47:		}
       9454227   19186471     48:		parts = append(parts, clean(v))
             .          .     49:	}
             .          .     50:
             .    3014794     51:	return strings.Join(parts, ".")
             .          .     52:}

原方案是采用slice存放字符串元素，最后通过string.join()来拼接， 我们多次调用了append方法，而在go里面slice其实如果容量不够的话，就会触发分配. 我们可优化为预先分配：

    parts := make([]string, 1, 50)

不过，strings 做join 比较慢，我们可以采用:

1. buffer 优化代替string 拼接：
2. 进一步，利用buffer 池，减少内存分配

代码参考: go-lib/master/stats/reporter.go

    var bufPool = sync.Pool{
        New: func() interface{} {
            return &bytes.Buffer{}
        },
    }

    func addTagsToName(name string, tags map[string]string) string {
        ....
        buf := bufPool.Get().(*bytes.Buffer)
        defer bufPool.Put(buf)
        buf.Reset()
        buf.WriteString(name)
        ....
        return buf.String()
    }

# 踩坑记： go 服务内存暴涨
https://www.v2ex.com/t/666257#reply94

# 参考
- [Go代码调优利器-火焰图](https://lihaoquan.me/2017/1/1/Profiling-and-Optimizing-Go-using-go-torch.html) 
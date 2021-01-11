# Sliding Window


滑动窗口算法相比较固定窗口算法的改进是解决了固定窗口切换时可能会产生两倍于阈值流量请求的缺点。

示例：

```Go
package main

import (
  "log"
  "sync"
  "time"
)


type counterSlidingWindow struct {
  windowSize int64 //整个滑动窗口的大小，单位秒
  splitNum int64 // 切分窗口的数目大小，每个窗口对应一个桶存储数据。
  currentBucket int // 当前的桶
  limit int // 滑动窗口内限流大小
  Bucket []int // 存放每个窗口内的计数
  startTime int64 // 滑动窗口开始时间
}

func NewSlidingWindow(windowSize int64, limit int, splitNum int64) *counterSlidingWindow {
  return &counterSlidingWindow{
    windowSize: windowSize,
    limit: limit,
    splitNum: splitNum,
    currentBucket: 0,
    Bucket:      make([]int, splitNum),
    startTime: time.Now().Unix(),
  }
}



func (c *counterSlidingWindow) tryAcquire() bool {
  currentTime := time.Now().Unix()
  // 计算请求时间是否大于当前滑动窗口的最大时间
  // 算出当前时间和开始时间减去窗口大小的值，作用为计算超出当前滑动窗口的时间
  t := currentTime - c.windowSize - c.startTime
  // 如果小于或等于0则代表未超出当前滑动窗口的时间。
  if t < 0 {
    t = 0
  }
  // 用t除以滑动窗口的份数，计算出需要滑动的数量。
  windowsNum := t / (c.windowSize / c.splitNum)
  c.slideWindow(windowsNum)
  count := 0
  for i := 0; i < int(c.splitNum); i++ {
    count += c.Bucket[i]
  }
  log.Printf("当前滑动窗口总数为: %d", count)
  if count > c.limit {
    //log.Println("开始限流")
    return false
  }
  index := c.getCurrentBucket()
  log.Println("当前的bucket", index)
  c.Bucket[index]++
  return true
}

func (c *counterSlidingWindow) getCurrentBucket() int {
  currentTime := time.Now().Unix()
  t := int(currentTime / (c.windowSize / c.splitNum) % c.splitNum)
  return t
}

func (c *counterSlidingWindow) slideWindow(windowsNum int64)  {
  var (
    // 滑动窗口默认设置为 splitNum
    slideNum int64 = c.splitNum
  )
  if windowsNum == 0 {
    return
  }
  // 如果windowsNum小于splitNum，则将slideNum设置为windowsNum
  // 如果windowsNum大于splitNum，就代表需要滑动的窗口大于一轮了，所以直接清空当前所有滑动窗口
  if windowsNum < c.splitNum {
    slideNum = windowsNum
  }
  log.Println("当前要滑动的窗口数", slideNum)
  for i := 0; i < int(slideNum); i++ {
    // 根据splitNum取余，获取当前的bucket
    c.currentBucket = (c.currentBucket +1) % int(c.splitNum)
    log.Printf("当前清空的位置: %d, 当前的大小: %d",c.currentBucket, c.Bucket[c.currentBucket])
    c.Bucket[c.currentBucket] = 0
  }
  c.startTime = c.startTime + windowsNum * (c.windowSize / c.splitNum)
}

func main()  {
  c := NewSlidingWindow(10, 5, 5)
  c.tryAcquire()
  wg := sync.WaitGroup{}
  wg.Add(1)
  go func() {
    defer func() {
      wg.Done()
    }()
    for i:=0; i<= 100; i++ {
      for j:=0; j<1; j++ {
        if c.tryAcquire() {
        }
      }
      //time.Sleep(time.Millisecond * 1429)
      time.Sleep(time.Second *6)
    }
  }()
  go func() {
    defer func() {
      wg.Done()
    }()
    for i:=0; i<= 100; i++ {
      for j:=0; j<1; j++ {
        if c.tryAcquire() {
        }
      }
      time.Sleep(time.Millisecond * 1429)
      //time.Sleep(time.Second *6)
    }
  }()
  wg.Wait()

}


```

输出：

```Go
2021/01/11 21:51:05 当前滑动窗口总数为: 0
2021/01/11 21:51:05 当前的bucket 2
2021/01/11 21:51:05 当前滑动窗口总数为: 1
2021/01/11 21:51:05 当前滑动窗口总数为: 1
2021/01/11 21:51:05 当前的bucket 2
2021/01/11 21:51:05 当前的bucket 2
2021/01/11 21:51:06 当前滑动窗口总数为: 3
2021/01/11 21:51:06 当前的bucket 3
2021/01/11 21:51:07 当前滑动窗口总数为: 4
2021/01/11 21:51:07 当前的bucket 3
2021/01/11 21:51:09 当前滑动窗口总数为: 5
2021/01/11 21:51:09 当前的bucket 4
2021/01/11 21:51:10 当前滑动窗口总数为: 6
2021/01/11 21:51:11 当前滑动窗口总数为: 6
2021/01/11 21:51:12 当前滑动窗口总数为: 6
2021/01/11 21:51:13 当前滑动窗口总数为: 6
2021/01/11 21:51:15 当前滑动窗口总数为: 6
2021/01/11 21:51:16 当前滑动窗口总数为: 6
2021/01/11 21:51:17 当前要滑动的窗口数 1
2021/01/11 21:51:17 当前清空的位置: 1, 当前的大小: 0
2021/01/11 21:51:17 当前滑动窗口总数为: 6
2021/01/11 21:51:17 当前滑动窗口总数为: 6
2021/01/11 21:51:19 当前要滑动的窗口数 1
2021/01/11 21:51:19 当前清空的位置: 2, 当前的大小: 3
2021/01/11 21:51:19 当前滑动窗口总数为: 3
2021/01/11 21:51:19 当前的bucket 4
2021/01/11 21:51:20 当前滑动窗口总数为: 4
2021/01/11 21:51:20 当前的bucket 0
2021/01/11 21:51:22 当前要滑动的窗口数 1
2021/01/11 21:51:22 当前清空的位置: 3, 当前的大小: 2
2021/01/11 21:51:22 当前滑动窗口总数为: 3
2021/01/11 21:51:22 当前的bucket 1
2021/01/11 21:51:23 当前要滑动的窗口数 1
2021/01/11 21:51:23 当前清空的位置: 4, 当前的大小: 2
```

注意：这只是测试的例子，还要考虑并发操作下考虑加锁或者使用`atmoic` 。

# 参考资料

[一文搞懂高频面试题之限流算法，从算法原理到实现，再到对比分析](https://zhuanlan.zhihu.com/p/228412634)

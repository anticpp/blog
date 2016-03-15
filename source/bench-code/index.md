---
title: bench_code
date: 2016-03-15 10:47:34
---

# python-threading.py
```python
#!/bin/env python

import threading
import time
import sys

tasks = []
c_running = 0
concurrency = 100

if len(sys.argv)>1:
    concurrency = int(sys.argv[1])

print "start testing, concurrency " + str(concurrency) + ": "

def worker(i):
    global c_running
    c_running = c_running + 1;

    start = int(time.time())
    while True:
        now = int(time.time())
        if now-start>600:
            break;
        time.sleep(2)
    c_running = c_running - 1;

def info():
    while True:
        time.sleep(0.5)
        print "RUNNING " + str(c_running) 

def schedule(n):
    global tasks
    for i in range(n):
        time.sleep( 0.001 )
        threading.Thread( target=worker, args=(i, ) ).start()

threading.Thread( target=schedule, args=(concurrency, )).start()
threading.Thread( target=info ).start()
```

# python-gevent.py
```python
#!/bin/env python


import gevent
import time
import sys

tasks = []
c_running = 0
concurrency = 100

if len(sys.argv)>1:
    concurrency = int(sys.argv[1])

print "start testing, concurrency " + str(concurrency) + ": "

def worker(i):
    global c_running
    c_running = c_running + 1;

    start = int(time.time())
    while True:
        now = int(time.time())
        if now-start>600:
            break;
        gevent.sleep(2)
    c_running = c_running - 1;

def info():
    while True:
        gevent.sleep(0.5)
        print "RUNNING " + str(c_running) 

def schedule(n):
    global tasks
    for i in range(n):
        gevent.sleep( 0.001 )
        tasks.append( gevent.spawn(worker, i) )

tasks.append( gevent.spawn(schedule, concurrency) )
tasks.append( gevent.spawn(info) )
gevent.joinall( tasks )
```

# go-coroutine.go
```go
package main

import "fmt"
import "time"
import "os"
import "strconv"

func worker(begin chan int, quit chan int) {
    boom := time.After(600 * time.Second)
    begin<-0
    for {
        select {
        case <-boom:
            quit<-0
            return
        default:
            time.Sleep(1 * time.Second)
        }
    }
}

func schedule(n int, begin chan int, quit chan int) {
    for i:=0; i<n; i++ {
        time.Sleep(1 * time.Millisecond)
        go worker(begin, quit)
    }
}

func info(begin chan int, quit chan int) {
    running := 0
    tick := time.Tick(500 * time.Millisecond)
    for {
        select {
        case <-begin:
            running++
        case <-quit:
            running--
        case <-tick:
            fmt.Println("RUNNING ", running)
        }
    }
}

func main() {

    concurrency := 1000
    if len(os.Args)>1 {
        concurrency, _ = strconv.Atoi( os.Args[1] )
    }
    begin := make(chan int)
    quit := make(chan int)
    go schedule(concurrency, begin, quit)
    go info(begin, quit)

    for {
        time.Sleep(10 * time.Second)
    }
}
```
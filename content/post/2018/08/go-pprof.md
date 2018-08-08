+++
title= "Go Pprof"
date= 2018-08-09T00:09:39+07:00
draft= false
og_image = "2018/08/cat_inside.png" 
categories = ["programming"]
tags = [ 
  "golang",
  "pprof",
  ]
+++

## การทำ Profiling สำหรับ Go
จดไว้หน่อยกันลืมไม่ค่อยได้ทำด้วย ใครทำเป็นเก่งๆสอนหน่อยนะครับผมทำผิดตรงไหน comment แจ้งกันได้นะครับเรื่องนี้เพิ่งศึกษาเหมือนกัน

## ทำ Profiling เพื่ออะไร
* CPU profiling เพื่อวัดการใช้งานของ CPU ว่าใน code ของเรา ใช้ CPU มากที่บรรทัดไหน
* Memory profiling เพื่อวัดการใช้งาน Memory ว่า code ของ allocate memory มากที่จุดไหน
* Block profiling คล้ายๆ CPU profiling แต่เป็นการหาว่า goroutine มันต้องหยุดเพื่อรอ share resource มากน้อยแค่ไหนเพื่อหาว่า คอขวดของ ระบบอยู่ที่ไหน

## วิธีการเก็บ profiling ของโปรแกรมของเรา

มีวิธีเก็บ หลักๆ 3 วิธี

* go test
* เก็บผ่าน http
* ให้โปรแกรม save ลง file 

## go test

```
$ go test -run none -bench . -benchtime 3s -benchmem -cpuprofile cpu.pprof
$ go tool pprof cpu.out
(pprof) list algOne
(pprof) web list algOne
or
(pprof) web
```

## เก็บผ่าน http

ใน code เพิ่ม 

```golang
import _ "net/http/pprof"
```

ถ้า code ไม่มีการใช้งาน http server เราจะต้อง เพิ่ม net/http ให้ code เราโดยเพิ่ม code ดังนี้

```golang
go func() {
	log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

ถ้ามีการเปิดใช้งาน net/http server อยู่แล้วก็ ListenAndServ ตามปรกติได้เลย

```golang
package main

import (
	"encoding/json"
	"log"
	"net/http"
	_ "net/http/pprof"
)

func main() {
	http.HandleFunc("/sendjson", sendJSON)

	log.Println("listener : Started : Listening on: http://localhost:4000")
	http.ListenAndServe(":4000", nil)
}
```

## ให้โปรแกรม save ลง file

ใน code เพิ่ม

```golang
package main

import "github.com/pkg/profile"

func main() {
        // CPU profiling by default
        defer profile.Start().Stop()
	//Default จะ profiling CPU ถ้าอยากเก็บ memory จะต้อง
	//defer profile.Start(profile.MemProfile).Stop()

        ...
}
```

## วิธีนำ file ที่ได้ไปวิเคราะห์

คำสั่ง pprof ต้องการ binary ของ code ที่เราจะวิเคราะห์ ฉะนั้นเราจะต้อง build code ของเราเป็น binary ด้วย
จากนั้น ติดตั้ง pprof

```
go get -u github.com/google/pprof
```

* การเรียกจาก file ที่ save ไว้

file ที่ได้จากการสั่ง go test หรือใช้ package "github.com/pkg/profile" สามารถวิเคราะห์ได้โดย

```
$~/go/bin/pprof -http=:6060 your_program_bin cpu.pprof
```

* การเรียกจาก http 

```
$~/go/bin/pprof -http=:6060 your_program_bin http://127.0.0.1:8080/debug/pprof/profile
```

## วิธีใช้ go tool trace

* เพิ่ม code เพื่อเก็บ trace

```golang
package main

import (
	"os"
	"runtime/trace"
)

func main() {
	f, err := os.Create("trace.out")
	if err != nil {
		panic(err)
	}
	defer f.Close()

	err = trace.Start(f)
	if err != nil {
		panic(err)
	}
	defer trace.Stop()

  // Your program here
}
```

* เรียกใช้งาน

```
$ go tool trace trace.out
```

ข้อมูลเพิ่มเติม https://making.pusher.com/go-tool-trace/

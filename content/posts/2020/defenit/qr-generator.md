---
title: "Defenit: QR Generator"
date: 2020-06-09T01:16:06-04:00
draft: false
toc: false
images:
tags: 
  - misc
---

When you first connect to the server, it asks you for a name, 
and then starts spitting out ones and zeros in a grid format, and
gives you a prompt `>>` but times out pretty quickly. I decided to
take a chance and assume they wanted the QR code output. Here's
what the service spit out, after the name prompt and some comments about levels:

```
< QR >
1 1 1 1 1 1 1 0 1 0 1 0 1 0 0 0 0 1 0 0 1 0 1 1 1 1 1 1 1
1 0 0 0 0 0 1 0 1 0 0 1 1 0 0 0 1 0 0 0 1 0 1 0 0 0 0 0 1
1 0 1 1 1 0 1 0 0 1 1 1 1 1 0 1 0 0 0 1 0 0 1 0 1 1 1 0 1
1 0 1 1 1 0 1 0 0 0 1 1 1 0 1 0 0 0 0 1 1 0 1 0 1 1 1 0 1
1 0 1 1 1 0 1 0 1 0 0 0 0 1 0 0 0 1 1 1 0 0 1 0 1 1 1 0 1
1 0 0 0 0 0 1 0 1 0 0 1 0 1 0 1 0 0 0 0 0 0 1 0 0 0 0 0 1
1 1 1 1 1 1 1 0 1 0 1 0 1 0 1 0 1 0 1 0 1 0 1 1 1 1 1 1 1
0 0 0 0 0 0 0 0 0 0 1 0 1 1 0 1 1 0 1 0 0 0 0 0 0 0 0 0 0
1 1 1 1 0 1 1 0 0 0 0 1 1 1 0 1 1 0 1 0 0 1 0 1 1 0 0 1 1
1 1 0 1 0 0 0 1 0 1 0 0 0 1 1 0 1 1 0 0 1 1 0 1 1 1 0 0 0
1 1 1 0 0 0 1 0 0 1 0 1 1 0 1 0 1 0 1 1 0 0 1 0 0 1 1 0 0
0 0 1 1 0 1 0 0 1 0 1 1 0 0 0 0 1 0 1 1 0 0 0 0 1 0 1 1 0
1 0 0 0 0 1 1 1 0 1 1 0 1 0 0 1 0 1 1 0 1 0 0 1 0 1 1 0 1
0 0 0 1 1 1 0 0 1 0 1 1 1 1 0 0 1 0 1 1 1 1 0 0 1 0 1 1 1
1 1 1 0 0 1 1 0 1 1 1 0 0 1 0 1 1 0 1 0 0 1 0 1 1 0 1 0 0
1 0 1 1 1 0 0 0 1 0 1 0 0 1 1 1 0 0 1 0 1 1 1 0 0 0 1 0 1
1 0 1 0 0 0 1 1 1 1 0 1 1 1 0 1 1 1 1 0 0 1 0 1 0 1 0 1 0
0 0 1 1 0 0 0 1 0 1 1 1 1 0 0 1 0 1 0 1 1 1 1 1 1 0 0 1 0
0 0 1 1 0 0 1 0 0 1 1 0 1 0 1 0 0 1 0 0 0 0 0 1 0 0 1 1 0
0 1 0 0 0 1 0 1 1 1 0 1 1 0 0 1 1 1 0 1 1 0 1 1 1 0 0 1 0
1 1 1 1 0 0 1 0 1 0 0 1 0 1 0 1 0 0 1 1 1 1 1 1 1 1 1 1 1
0 0 0 0 0 0 0 0 0 1 0 1 1 1 1 0 1 0 1 0 1 0 0 0 1 1 1 1 1
1 1 1 1 1 1 1 0 0 0 1 0 0 0 1 1 0 1 0 1 1 0 1 0 1 0 1 1 0
1 0 0 0 0 0 1 0 1 1 1 1 0 1 1 1 1 1 0 0 1 0 0 0 1 0 0 1 1
1 0 1 1 1 0 1 0 0 1 1 1 0 1 0 0 0 1 1 0 1 1 1 1 1 1 0 0 1
1 0 1 1 1 0 1 0 1 1 0 1 1 1 1 1 0 1 1 0 0 1 1 1 1 0 0 1 0
1 0 1 1 1 0 1 0 1 1 1 0 1 0 1 0 0 1 1 0 0 0 0 0 0 0 0 0 0
1 0 0 0 0 0 1 0 1 1 0 1 0 1 1 0 0 1 0 0 0 1 0 1 1 0 1 1 1
1 1 1 1 1 1 1 0 1 1 1 1 0 1 1 0 0 1 1 0 0 1 1 0 0 1 1 1 0

>> 
```

Eventually I realized it needed to handle varying sizes of QR codes and
handle reading until the prompt. 

Here is the my code:

```qr.go
package main

import (
    "fmt"
    "image"
    "image/color"
    "log"
    "net"
    "strings"

    "github.com/makiuchi-d/gozxing"
    "github.com/makiuchi-d/gozxing/qrcode"
)

func main() {
    con, err := net.Dial("tcp", "qr-generator.ctf.defenit.kr:9000")
    if err != nil {
        panic(err)
    }
    defer con.Close()
    qrmark := "< QR >"
    buf := make([]byte, 1024*16)
    nBytes, err := con.Read(buf)
    nBytes, err = con.Read(buf)
    nBytes, err = con.Write([]byte("E\n"))
    codestr := ""
    for {
        nBytes, err = con.Read(buf)
        codestr = codestr + string(buf[:nBytes])
        fmt.Println(codestr)
        if strings.Index(codestr, "Defenit{") >= 0 {
            break
        }
        if strings.Index(codestr, qrmark) < 0 || strings.Index(codestr, ">>") < 0 {
            continue
        }
        codestr = codestr[strings.Index(codestr, qrmark):]

        lines := strings.Split(codestr, "\n")
        width := len(strings.Split(lines[1], " "))
        img := image.NewRGBA(image.Rectangle{image.Point{0, 0}, image.Point{width, width}})
        fmt.Println("qr code width:", width)
        for x, line := range lines[1:] {
            arr := strings.Split(line, " ")
            if len(arr) == width {
                for y, val := range arr {
                    if val == "1" {
                        img.Set(x, y, color.Black)
                    }
                }
            } else {
                break
            }
        }

        bmp, _ := gozxing.NewBinaryBitmapFromImage(img)
        qrReader := qrcode.NewQRCodeReader()
        text, err := qrReader.Decode(bmp, nil)
        if err != nil {
            log.Fatal(err)
        }
        fmt.Println("qrcode value:", text)
        con.Write([]byte(fmt.Sprintf("%s\n", text)))
        codestr = ""
    }
}
```

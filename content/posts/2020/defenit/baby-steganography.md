---
title: "Defenit: Baby Steganography"
date: 2020-06-11T00:33:40-04:00
draft: false
toc: false
images:
tags: [steganography, audio]
---
They give you a file without an extension, and hint that the "sub bit" contains
some hidden data. [problem file](/ctf-writeups/defenit/problem)

When I opened the file in hex fiend, I could see the header of
`RIFF‡ı∏WAVEfmt`, indicating that it was a wav file. So I looked for a wav
 library, pulled out the samples, and tried to figure out where it might
be hiding.

There were two channels, left and right, and 16 bits for samples. I tried
looking in the LSB of each channel, both channels, putting the bits together
into bytes, but didn't find anything interesting. It was my teammate that tried the LSB of
each byte, which ended up being correct.

Here's the final code, with some comments:

```baby-stega.go
package main

import (
	"fmt"
	"io"
	"os"

	wav "github.com/youpy/go-wav"
)

func main() {
	file, _ := os.Open("problem")
	defer file.Close()
	reader := wav.NewReader(file)

	arr := make([]byte, 4096)
	var buildingByte byte
	var bitPosition int
	for {
		// this code was originally broken up like this to read samples
		// and get the left and right sample LSB
		n, err := reader.Read(arr)
		if err == io.EOF {
			break
		}
		for _, b := range arr[:n] {
			// when we've filled the byte (8 bits)
			if bitPosition == 8 {
				// skip common bytes we don't care about
				// ... discovered after running
				if buildingByte != 0 && buildingByte != '#' {
					fmt.Printf("%c", buildingByte)
				}
				// reset byte and position
				buildingByte = 0
				bitPosition = 0
			}
			// shift and or the bit into position
			buildingByte |= ((b & 0x1) << (7 - bitPosition))
			bitPosition++
		}
	}
}
```

and the output:

```
Defenit{Y0u_knOw_tH3_@uD10_5t39@No9rAphy?!}
```

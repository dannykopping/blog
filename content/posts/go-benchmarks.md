---
title: 'Golang quickie: benchmark output'
description: Benchmark output fun with Go
date: 2022-06-09
draft: false
tags: [quickie, go, benchmarks, performance]
---

## Benchmarks in Go

Go provides a convenient way for writing & running benchmarks, but by default doesn't show you much useful info.

Here's an example:

```golang
package main

import (
	"math/rand"
	"testing"
)

var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")

func randSeq(n int) string {
	b := make([]rune, n)
	for i := range b {
		b[i] = letters[rand.Intn(len(letters))]
	}
	return string(b)
}

func BenchmarkRandInt(b *testing.B) {
	for i := 0; i < b.N; i++ {
		randSeq(i)
	}
}
```

Here's the output:

```bash
$ go test -bench=.               
goos: linux
goarch: amd64
pkg: github.com/dannykopping/blog
cpu: Intel(R) Core(TM) i7-10510U CPU @ 1.80GHz
BenchmarkRandInt-8         10000            148245 ns/op
PASS
ok      github.com/dannykopping/blog    1.486s
```

Not much info there, just the iterations (`10000`) and nanosecond timing per operation (`ns/op`) detail.

### Allocations

```golang
func BenchmarkRandInt(b *testing.B) {
	b.ReportAllocs() // <- adding this

	for i := 0; i < b.N; i++ {
		randSeq(i)
	}
}
```

If we add [`b.ReportAllocs()`](https://pkg.go.dev/testing#B.ReportAllocs) to our benchmark, we'll now see this output:

```bash
$ go test -bench=.
goos: linux
goarch: amd64
pkg: github.com/dannykopping/blog
cpu: Intel(R) Core(TM) i7-10510U CPU @ 1.80GHz
BenchmarkRandInt-8         10000            144155 ns/op           26858 B/op          2 allocs/op
PASS
ok      github.com/dannykopping/blog    1.445s
```

We can also use the `-benchmem` flag to get the same output.

We now have two additional entries per line, namely bytes per operation (`B/op`) and heap allocations per op (`allocs/op`).

This shows us that our program needs to allocate twice per operation, which can be improved upon. Maybe we shouldn't
cast that slice of runes to a string...

```golang
func randSeq(n int) []rune { // <- changed return type
	b := make([]rune, n)
	for i := range b {
		b[i] = letters[rand.Intn(len(letters))]
	}

	return b // <- no more cast
}
```

```bash
$ go test -bench=.
goos: linux
goarch: amd64
pkg: github.com/dannykopping/blog
cpu: Intel(R) Core(TM) i7-10510U CPU @ 1.80GHz
BenchmarkRandInt-8         10000            119485 ns/op           21540 B/op          1 allocs/op
PASS
ok      github.com/dannykopping/blog    1.198s
```

Great, down to 1 `allocs/op`.

We can also see that we're processing `21540 B/op`, but can we see how much data we're processing per second?

### Speed

We can track the amount of data being processed in our benchmarks by adding [`b.SetBytes(n int64)`](https://pkg.go.dev/testing#B.SetBytes).

```golang
func BenchmarkRandInt(b *testing.B) {
	b.ReportAllocs()

	for i := 0; i < b.N; i++ {
		b.SetBytes(int64(len(randSeq(i)))) // <- setting this
	}
}
```

```bash
 $ go test -bench=.
goos: linux
goarch: amd64
pkg: github.com/dannykopping/blog
cpu: Intel(R) Core(TM) i7-10510U CPU @ 1.80GHz
BenchmarkRandInt-8         10000            113312 ns/op          88.24 MB/s       21540 B/op          1 allocs/op
PASS
ok      github.com/dannykopping/blog    1.136s
```

OK, nice. `88.24 MB/s` of bytes being processed.

### Custom Metrics

In the output above, we saw `BenchmarkRandInt-8         10000` with `10000` indicating test iterations.

We can add a custom metric which will report the same figure with [`b.ReportMetric(n float64, unit string)`](https://pkg.go.dev/testing#B.ReportMetric).

```golang
func BenchmarkRandInt(b *testing.B) {
	b.ReportAllocs()

	for i := 0; i < b.N; i++ {
		b.SetBytes(int64(len(randSeq(i))))
        b.ReportMetric(float64(i+1), "runs")
	}
}
```

```bash
$ go test -bench=. -benchtime=100x                 
goos: linux
goarch: amd64
pkg: github.com/dannykopping/blog
cpu: Intel(R) Core(TM) i7-10510U CPU @ 1.80GHz
BenchmarkRandInt-8           100              1526 ns/op          64.86 MB/s           100.0 runs            206 B/op          1 allocs/op
PASS
ok      github.com/dannykopping/blog    0.002s
```

Using the `-benchtime=100x` flag to limit the iterations to 100, we can now see `100.0 runs` in the output!
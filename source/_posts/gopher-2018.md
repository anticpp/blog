---
title: gopher-2018
date: 2018-05-11 23:28:16
tags:
---

记录一下参加gopher 2018的东西

## Go在Grab地理服务中的实践

- geohash
	基于坐标的hash，快速查找附近的司机

- goreplay
	流量抓取

- go-torch
	火焰图

## Error handling in Go 2 

- github.com/mpvl/errc
- github.com/mpvl/errd
- try & handle
- refer: Go: the Good, the Bad and the Ugly

## Why we design a fast key-value database

- motivation
- levelDB, RocksDB
- LSM Trees/B+ Trees
- Separate keys from values.
- Key stored in `LSM` Tree
- Value stored in value log.
- CGO evil

## IPFS

- DHT

## Composition in Go

- Interface decouple: What you do makes who you are.
- https://gogoing.

## Bazel build //:go

- Bazel
- vgo

## CGO

- Soooooooooooo complex

## arm64的go工具链

- `go tool`
- go assembler,/..................
- `go build -x test.go`

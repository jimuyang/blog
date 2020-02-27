---
title: Go Modules那些事儿
date: 2020-02-22 16:19:34
tags:
- Golang
---

# 简介 
A repository contains one or more modules. A module is a collection of related Go packages that are released together. A Go repository typically contains only one module, located at the root of the repository. A file named go.mod there declares the module path: the import path prefix for all packages within the module. The module contains the packages in the directory containing its go.mod file as well as subdirectories of that directory, up to the next subdirectory containing another go.mod file (if any).


# GO111MODULE

Using Go 1.13, GO111MODULE's default (auto) changes:
* behaves like GO111MODULE=on anywhere there is a go.mod OR anywhere outside the GOPATH even if there is no go.mod.   
So you can keep all your repositories in your GOPATH with Go 1.13.
* behaves like GO111MODULE=off in the GOPATH with no go.mod.

> https://dev.to/maelvls/why-is-go111module-everywhere-and-everything-about-go-modules-24k

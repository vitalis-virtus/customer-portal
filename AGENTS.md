# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

This is a new, mostly empty Go module (`github.com/vitalis-virtus/customer-portal`, Go 1.25). `main.go` currently contains only an empty `main` function. There is no build tooling, test suite, or architecture yet — update this file as real structure is added.

## Commands

- Build: `go build ./...`
- Run: `go run .`
- Test: `go test ./...`
- Format: `gofmt -l .` (check) / `gofmt -w .` (apply)
- Vet: `go vet ./...`

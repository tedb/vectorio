vectorio: Scatter/Gather I/O for Go
===

vectorio is a library for Go (aka "golang") to implement the writev(2), and in the future, readv(2) system calls.  This enables a list of byte slices to be efficiently written to a file without first copying them to an intermediate buffer.  Go's built-in [bufio](http://golang.org/pkg/bufio/) library is the standard approach to buffered I/O, but vectorio can be a replacement to achieve the same goal but without copying data.

This is not an official Google product.  Authored by Ted Behling as a personal project.

[![Build Status](https://travis-ci.org/google/vectorio.svg)](https://travis-ci.org/google/vectorio)

Example
---
        package main
        
        import (
        	"fmt"
        	"io/ioutil"
        	"syscall"
        
        	"github.com/google/vectorio"
        )
        
        func main() {
        	// Create a temp file for demo purposes
        	f, err := ioutil.TempFile("", "vectorio")
        	if err != nil {
        		panic(err)
        	}
        	defer f.Close()
        
        	data1 := []byte("foobarbaz_buf\n")
        	data2 := []byte("barbazfoo_buf\n")
        
        	// This demonstrates the "buffered" form of the library, similar to bufio.
        	// w implements io.Writer.
        	w, err := vectorio.NewBufferedWritev(f)
        	if err != nil {
        		panic(err)
        	}
        
        	// The simple method is to just write a byte slice;
        	// this is converted to a syscall.Iovec and queued
        	// for writing
        	w.Write(data2)
        
        	// The user can also gain more control with WriteIovec,
        	// although this does not have a significant advantage over Write.
        	// This is what Write does on your behalf.
        	w.WriteIovec(syscall.Iovec{&data1[0], uint64(len(data1))})
        
        	// Flush must be called after writes are complete, to empty out
        	// the buffer of pending Iovec.
        	// Returns the total number of bytes written, as reported by the underlying syscall.
        	nw1, err := w.Flush()
        	if err != nil {
        		panic(err)
        	}
        
        	// One can also write a slice of byte slices ([][]byte) all at once.
        	// Note, this usage does *not* implement io.Writer, but if you have a slice
        	// of byte slices to write, this is a way to do that without looping.
        
        	multiple := [][]byte{
        		[]byte("foobarbaz_slice\n"),
        		[]byte("foobazbar_slice\n"),
        	}
        
        	// we return the number of bytes written, as reported by the underlying syscall.
        	nw2, err := vectorio.Writev(f, multiple)
        	if err != nil {
        		panic(err)
        	}
        
        	fmt.Println("Wrote", nw1+nw2, "bytes to file", f.Name())
        }

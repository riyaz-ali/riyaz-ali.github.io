---
title: Berkeley Packet Filter in Golang
date: 2021-01-24
author: riyaz-ali
background: https://source.unsplash.com/4m7gmLNr3M0
description: |
    In this article, I explore how we can use the
    classic Berkeley Packet Filter (cBPF) in Golang.
aliases:
- /bpf-filters-in-golang.html
tags:
- Golang
- Networking
- HowTo
keywords:
- Golang
- Packet Filter
- BPF
- Networking
---

Recently I stumbled upon a use case where I was looking to setup a Wireguard (udp) tunnel across two nodes behind different NATs. I wanted to implement something like 
[UDP Hole Punching](https://en.wikipedia.org/wiki/UDP_hole_punching) and came across [this mailing list message](https://lists.zx2c4.com/pipermail/wireguard/2016-August/000372.html)
from 2016 which linked to the [`contrib/nat-hole-punching`](https://git.zx2c4.com/wireguard-tools/tree/contrib/nat-hole-punching) source in Wireguard.

I'll cover the whole _hole punching_ mechanism, perhaps, in a different article. But for context, the mentioned source used [`raw` linux sockets](https://linux.die.net/man/7/raw)
and [_classic_ Berkeley Packet Filter](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter) to inject custom packets and do hole punching. 

The source is written in C but I wanted my implementation in Golang. So, I started looking and soon came across [`golang.org/x/net/bpf`](https://pkg.go.dev/golang.org/x/net/bpf)
which provides an implementation of raw `bpf` assembly codes and [a virtual machine](https://pkg.go.dev/golang.org/x/net/bpf#VM) written in Golang. I wanted to use the assembly codes but not necessarily depend on the `bpf.VM` (that would've defeated the purpose of filters in the first place 😅).

Instead, I started looking for ways in which I could use [`setsockopt`](https://linux.die.net/man/2/setsockopt) with `SO_ATTACH_FILTER` to attach the filter directly to the native
file descriptor. I came across [`ipv4#PacketConn.SetBPF`](https://pkg.go.dev/golang.org/x/net/ipv4#PacketConn.SetBPF) method from [`golang.org/x/net/ipv4`](https://pkg.go.dev/golang.org/x/net/ipv4) which takes in a `[]bpf.RawInstructions` and applies it onto the underlying socket, although the socket in this case wasn't exactly the `raw` socket I was working with.

To circumvent this, I started looking further into the source, till I found [`sockOpt.setAttachFilter`](https://github.com/golang/net/blob/5f4716e94777e714bc2fb3e3a44599cb40817aac/ipv4/sys_bpf.go#L17-L24) and, consequently, [`setsockopt`'s implementation on unix](https://github.com/golang/net/blob/5f4716e94777e714bc2fb3e3a44599cb40817aac/internal/socket/sys_unix.go#L20-L23). The `setsockopt` used [`syscall.Syscall6`](https://pkg.go.dev/syscall#Syscall6) with 
`syscall.SYS_SETSOCKOPT` to implement the system call and passed through the given arguments as `unsafe.Pointer` to the syscall. 

### Putting it all together

After all this I came with something close the following code snippet to enable using berkeley filter with sockets in Golang 🚀

```Golang
package filter

import (
	"golang.org/x/net/bpf"
	"golang.org/x/sys/unix"

	"syscall"
	"unsafe"
)

// Filter represents a classic BPF filter program that can be applied to a socket
type Filter []bpf.Instruction

// ApplyTo applies the current filter onto the provided file descriptor
func (filter Filter) ApplyTo(fd int) (err error) {
	var assembled []bpf.RawInstruction
	if assembled, err = bpf.Assemble(filter); err != nil {
		return err
	}

	var program = unix.SockFprog{
        Len: uint16(len(assembled)), 
        Filter: (*unix.SockFilter)(unsafe.Pointer(&assembled[0])),
    }
	var b = (*[unix.SizeofSockFprog]byte)(unsafe.Pointer(&program))[:unix.SizeofSockFprog]

	if _, _, errno := syscall.Syscall6(syscall.SYS_SETSOCKOPT,
		uintptr(fd), uintptr(syscall.SOL_SOCKET), uintptr(syscall.SO_ATTACH_FILTER),
		uintptr(unsafe.Pointer(&b[0])), uintptr(len(b)), 0); errno != 0 {
		return errno
	}

	return nil
}
```

Using this is simple too! You just have to define the filter program using `bpf` assembly codes, like:

```Golang
// filter packet by checking if they are destined to local port 8080
var filter = Filter{
    bpf.LoadAbsolute{Off: 22, Size: 2},  // load the destination port
    bpf.JumpIf{Val: 8080, SkipFalse: 1}, // if Val != 8080 skip next instruction

    bpf.RetConstant{Val: 0xffff}, // return 0xffff bytes (or less) from packet
    bpf.RetConstant{Val: 0x0},    // return 0 bytes, effectively ignore this packet
}
```

and call `ApplyTo` on the socket's file descriptor.

```Golang
// import "syscall"

// open a raw socket
fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_RAW, syscall.IPPROTO_UDP)
if err != nil {
    // ... define error handling
}

// then apply the filter
err = filter.ApplyTo(fd)
if err != nil {
    // ... define error handling
}
```

And that's it! the kernel would filter out packets based on specified filter program and next time you do a `syscall.Read` on your `fd` you'd only see the packets you're interested in 🎉 🚀
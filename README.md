# mkwinsyscall

Drop-in replacement fork of **mkwinsyscall**, a tool that generates windows system call bodies based on their prototypes.

The official documentation can be found there: https://pkg.go.dev/golang.org/x/sys/windows/mkwinsyscall

## Why this fork ?

I made this fork to include my custom resolution library named [**MyProc**](https://github.com/atsika/myproc). It allows you to resolve module handles (GetModuleHandle) and function (or proc) addresses (GetProcAddress) manually, without using the Windows API. It includes the API hashing technique (using fnv1a hashing algo) to hide which functions are used.

## Usage

This is a drop-in replacement of **mkwinsyscall**, so you just need to replace the URL from the `go generate` command.

For example: 

```go
//go:generate go run golang.org/x/sys/windows/mkwinsyscall -output winsyscalls_windows.go definitions.go

//sys RtlCopyMemory(dest uintptr, src uintptr, dwSize uint32) = ntdll.RtlCopyMemory
//sys HeapAlloc(hHeap windows.Handle, dwFlags uint32, dwBytes uintptr) (lpMem uintptr, err error) = kernel32.HeapAlloc
```

Become:

```go
//go:generate go run github.com/atsika/mkwinsyscall -output winsyscalls_windows.go definitions.go

//sys RtlCopyMemory(dest uintptr, src uintptr, dwSize uint32) = ntdll.RtlCopyMemory
//sys HeapAlloc(hHeap windows.Handle, dwFlags uint32, dwBytes uintptr) (lpMem uintptr, err error) = kernel32.HeapAlloc
```

## Output

The below example shows the output of the functions resolved using **MyProc** by hash.

```go
var (
	modkernel32 = pelib.NewDLL(uint32(0xa3e6f6c3)) // kernel32.dll
	modntdll    = pelib.NewDLL(uint32(0xa62a3b3b)) // ntdll.dll

	procHeapAlloc     = pelib.NewProc(modkernel32, uint32(0x2cbb1074)) // HeapAlloc
	procRtlCopyMemory = pelib.NewProc(modntdll, uint32(0x4779076f))    // RtlCopyMemory
)

func HeapAlloc(hHeap windows.Handle, dwFlags uint32, dwBytes uintptr) (lpMem uintptr, err error) {
	r0, _, e1 := syscall.SyscallN(procHeapAlloc.Addr(), uintptr(hHeap), uintptr(dwFlags), uintptr(dwBytes))
	lpMem = uintptr(r0)
	if lpMem == 0 {
		err = errnoErr(e1)
	}
	return
}

func RtlCopyMemory(dest uintptr, src uintptr, dwSize uint32) {
	syscall.SyscallN(procRtlCopyMemory.Addr(), uintptr(dest), uintptr(src), uintptr(dwSize))
	return
}
```

## Limitations

> :warning: This fork doesn't make the call to `LoadLibrary` that is done under-the-hood by `NewDLL` because of the API hashing technique.
> This means that if a DLL is not loaded and the lookup isn't made by string, the function `pelib.NewDLL` will fail and return `nil`. The same will happen for `pelib.NewProc`.
> 
> To be sure and avoid your program crashing, check if the returned value is different from `nil` before using it.

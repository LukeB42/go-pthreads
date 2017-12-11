go-pthreads
===========

This is a binding of C's pthreads to Google Go. **This library is not a
replacement for goroutines.** This library is designed to help bind C libraries
with blocking function calls to Go in a go-friendly manner. If this is not your
use case, this library probably won't help you.

Use Case
--------

If a goroutine exists that calls a function that will block potentially
indefinitely, that goroutine cannot be stopped until the blocking function
returns and the goroutine checks an "exit" channel, and exits of its own will.

In every day go programming, this condition should not exist, as all inter-
thread communication, as well as reading and writing, should be done using
channels. However, in C, many libraries (and, in my specific case, networking
libraries) implement blocking functions (recv), and the mixture of a blocking
function and a multi-channel select caused many implementation problems and
"hacks" to get the "blocking" function to return periodically, so the exit
channel could be checked, and the routine could exit if it had to.

Example
-------

```go
package main

import (
	pthread "./go-pthreads"
	"fmt"
	"io"
	"log"
	"net"
	"time"
)

func main() {
	fmt.Println("Binding to localhost:8000")
	thread1 := pthread.Create(spinner, 70*time.Millisecond)
	thread2 := pthread.Create(listener, nil)

	input := ""
	fmt.Scanln(&input) // Hit return to exit.
	thread1.Kill()
	thread2.Kill()
}

// Must accept an interface type as this is intended to be a pthread
func listener(args interface{}) {
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Print(err) // e.g., connection aborted
			continue
		}
		go handleConn(conn) // handle connections in coroutines
	}

}

func handleConn(c net.Conn) {
	defer c.Close()
	for {
		_, err := io.WriteString(c, time.Now().Format("15:04:05\n"))
		if err != nil {
			return // e.g., client disconnected
		}
		time.Sleep(1 * time.Second)
	}
}

func spinner(args interface{}) {
	delay := args.(time.Duration)
	for {
		for _, r := range `-\|/` {
			fmt.Printf("\r%c", r)
			time.Sleep(delay)
		}
	}
}
```

Output:

```
$ ./clock_threaded
Binding to localhost:8000
/
```

From another shell

```
$ nc localhost 8000
10:31:26
10:31:27
10:31:28
^C

```


Pros/Cons
---------

Pros:

* Provides a mechanism to kill a blocked thread (thread.Kill())
* Provides thread status without any logic in the child (thread.Running())

Cons:

* Does not implement pthread_cleanup_push/pop
* Runs in a dedicated thread (most of the time; sometimes this is a pro)
* Does not integrate with the go scheduler (as a consequence of the new thread)
* Harder to debug (crashes in C code don't produce stack traces)


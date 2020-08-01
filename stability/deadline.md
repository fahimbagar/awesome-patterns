# Deadline Pattern

Package deadline implements the deadline (also known as "timeout") resiliency pattern for Go.

## Implementation

Below is the implementation of a very simple deadline to illustrate the purpose
of the deadline design pattern.

### Deadline Constructor

`Deadline` is a simple struct that implements timeout duration checking for maximum execution time.

```go
package deadline

import (
	"errors"
	"time"
)

// ErrTimedOut is the error returned from Run when the deadline expires.
var ErrTimedOut = errors.New("timed out waiting for function to finish")

// Deadline implements the deadline/timeout resiliency pattern.
type Deadline struct {
	timeout time.Duration
}

// New constructs a new Deadline with the given timeout.
func New(timeout time.Duration) *Deadline {
	return &Deadline{
		timeout: timeout,
	}
}
```

### Deadline Implementation

`Run` runs the given function, passing it a stopper channel. If the deadline passes before the function finishes executing, `Run` returns `ErrTimeOut` to the caller and closes the stopper channel so that the work function can attempt to exit gracefully. 

It does not (and cannot) simply kill the running function, so if it doesn't respect the stopper channel then it may keep running after the deadline passes. If the function finishes before the deadline, then the return value of the function is returned from `Run`.


**Note:** Deadline can be replaced by Context, check the [Related Content](#related-contents). 
```go
func (d *Deadline) Run(work func(<-chan struct{}) error) error {
	result := make(chan error)
	stopper := make(chan struct{})

	go func() {
		value := work(stopper)
		select {
		case result <- value:
		case <-stopper:
		}
	}()

	select {
	case ret := <-result:
		return ret
	case <-time.After(d.timeout):
		close(stopper)
		return ErrTimedOut
	}
}
```

### Deadline Usage
```go
func ExampleDeadline() {
	dl := New(1 * time.Second)

	err := dl.Run(func(stopper <-chan struct{}) error {
		// do something possibly slow
		// check stopper function and give up if timed out
		return nil
	})

	switch err {
	case ErrTimedOut:
		// execution took too long, oops
	default:
		// some other error
	}
}
```

# Related Contents

- [Deadline in Golang using Context](https://blog.golang.org/context)

- [Example using Context](https://golang.org/src/context/example_test.go)
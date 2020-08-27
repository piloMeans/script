# Fake README

[Origin README](./README_origin.md)

# Why I get this

Because the code structure of the `script` project is cool.

Basic use `source` => `filter` => `sink` to monitor the linux script

And Pipe is used to transfer data between them.

### Pipe

`Pipe` in this project, is a struct, contains two field `Reader` and `err`. Reader is intersting, which is implemented to **auto** close if the reader reaches EOF.

```go
type Pipe struct {
	Reader ReadAutoCloser
	err error
}
type ReadAutoCloser struct{
	r io.ReadCloser
}

func (a ReadAutoCloser) Read(buf []byte) (n int, err error) {
	if a.r == nil {
		return 0, io.EOF
	}
	n, err = a.r.Read(buf)
	if err == io.EOF {
		a.Close()
	}
	return n, err
}
```
# Fake README

[Origin README](./README_origin.md)

# Why I get this

Because the code structure of the `script` project is cool.

Basic use `source` => `filter` => `sink` to monitor the linux script

And Pipe is used to transfer data between them.

## Pipe

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

# Example 

## Least_Freq example

In this prog, it get the least freq string of the stdin.

```go
func main(){
	script.Stdin().Freq().Last(1).Stdout()
}
```

The logic is simple, read the `Stdin` and Use `Freq` to analysis the input, the `Last` get last one, and `Stdout` output the result.

#### Stdin()

The script is connected data with `Pipe`, so the `Stdin()` actually wrap the stdin as a pipe.

```go
func Stdin() *Pipe {
	// the pipe.WithReader acturally set the Read field of the Pipe
	return NewPipe().WithReader(os.Stdin)
}
```

#### Freq()

`Freq()`  read the data of the Stdin Pipe, and use map to do stastic and slice do sort work.

```go
// here i'd ignore some err check and other detail
func (p *Pipe) Freq() *Pipe {
	freq := map[string]int{}  // create empty map

	// Pipe.EachLine use scanner to travel the Pipe Reader and do process work
	p.EachLine(func(line string, out *strings.Builder)){    
		freq[line]++
	}

	type frequency struct {
		line string
		count int
	}

	freqs := make([]frequency, 0, len(freq)) // make a empty slice for sort
	for line, count := range freq {   // read the map
		freqs = append(freqs, frequency{line, count})  // add the elem to slice
	}

	// sort the freqs
	// count first, the line (string order)
	sort.Slice(freqs, func(i, j int) bool {   
		if freqs[i].count == freqs[j].count {
			return freqs[i].line < freqs[j].line
		}
		return freqs[i].count > freqs[j].count
	})

	// final, output the res to construct a pipe
	var output strings.Builder
	for _, items := range freqs {
		output.WriteString(fmt.Sprintf("%d %s", items.count, items.line))
		output.WriteRune('\n')
	}
	return Echo(output.String())
}

// Echo construct a Pipe from a string, use the strings.NewReader()
func Echo(s string) (p *Pipe) {
	return NewPipe().WithReader(strings.NewReader(s))
}
```

#### Last()

Now, we get a pipe with Freq result, so Last would only output the last several elements.

```go
func (p *Pipe) Last(lines int) (*Pipe) {
	defer p.Close()
	
	scanner := bufio.NewScanner(p.Reader)
	input := ring.New(lines)  // use a ring to record result, cool
	for scanner.Scan() {
		input.Value = scanner.Text()
		input = input.Next()           // make sure the input always the next of the last
	}
	output := strings.Builder()

	// for each ring's element, check whether the value is string
	// because the input always the next of the last:
	// if the ring is full, all element contains string, so result ok
	// if the ring is not full, the elements, between "input" and first element, 
	// do not contain string, so skip, the result is ok 
	input.Do(func(p interface{}){
		value, ok := p.(string)
		if ok {
			output.WriteString(value)
			output.WriteRun('\n')
		}
	})

	return Echo(output.String())
}
```
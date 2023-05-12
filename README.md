# code-style-guideline

Rules for writing clear, idiomatic Go code.

## Formatting

All Go code in the Bequant projects must be formatted with gofmt before pushing to repository.

## Naming
### Variables
Use 'userID' rather than 'userId'. Same for abbreviations; 'HTTP' is preferred over 'Http' or 'http'.

Constants should never be named using uppercase or snakecase.

Do:
```go
const WebsocketEventPostEdited = 1
```
Don't:
```go
const WEBSOCKET_EVENT_POST_EDITED = 1
```
Please use constants widely for repeated constant values.

### Functions
It is common rule for functions and methods to be named starting with verb.

Do:
```go
func convertType() {}
```
Don't:
```go
func toType() {}
```

### Getters
It's neither idiomatic nor necessary to put Get into the getter's name.
Do:
```go
obj.Property()
```
Don't:
```go
obj.GetProperty()
```
### Interfaces
By convention, one-method interfaces are named by the method name plus an -er (-or) suffix or similar modification 
to construct an agent noun: Reader, Writer, Formatter, CloseNotifier etc.

Do:
```go
type Cleaner interface {}
```
Don't:
```go
type ICleanAdapter interface{}
```
### Mixed caps
The convention in Go is to use MixedCaps or mixedCaps rather than underscores to write multiword names. 

## Control structures defining
If function or method return single result and outer function body must return, it's common to see if clause composed to single line.
Do:
```go
if err := file.Chmod(0664); err != nil {
    return err
}
```
Don't:
```go
err := file.Chmod(0664)
if err != nil {
    return err
}
```
Also you'll find that when an if statement doesn't flow into the next statement—that is, the body ends in break, continue, goto, or return—the unnecessary else is omitted.
## Loops handling
Iteration over structure that contains pointer may lead to bugs. To avoid them redeclare value inside the loop scope.

Do:
```go
array := []*string{}
for _, val := range array {
	val := val
	// process the 'val' variable
}
```
Don't:
```go
array := []*string{}
for _, val := range array {
// process th 'val' variable
}
```
## Labeling (GOTO)
There's no point to use them. Just don't.
## Type asserting, type switching
Golang code will panic if type assertion doesn't protected with type check.

Do:
```go
newType, ok := oldType.(string)
// check if 'ok' is ok here
```
Don't:
```go
newType := oldType.(string)
// possible panic
```
Also if assertion may not result single type it's better solution to use type switching.

Do:
```go
switch t := t.(type) {
case int64:
	// process int
case float64:
	// process float
default:
	// you can return error here
}
```
Don't:
```go
newType, ok := t.(int64)
if !ok {
    newType, ok := t.(float64)
	if !ok {
		// etc.
}

}
```
## Deferring
Go's defer statement schedules a function call (the deferred function) to be run immediately before the function executing the defer returns. It's an unusual but effective way to deal with situations such as resources that must be released regardless of which path a function takes to return. The canonical examples are unlocking a mutex or closing a file. 
One of the most often us cases for deferring function call is mutex releasing, closing open files.

Do:
```go
func someFunc() {
	mx.Lock()
	defer mx.Unlock()
}
```
Don't:
```go
func someFunc() {
mx.Lock()
// some further code with body interruption
if err != nil {
	mx.Unlock()
	return err
}
// some further code leading to successful completion
mx.Unlock()
return nil
}
```
Please note that deferring function must not be invoked inside of loops because that may lead to stack overflow.

## Data handling

### Slices allocating
If possible to avoid memory leaking by the nature of method 'append' initiate new slice with specific capacity 

Do:
```go
newSlice := make([]string, 0, len(inputData))
```
Don't:
```go
newSlice := make([]string, 0)
```

If resulting slice capacity is unpredictable it's better to reinitialize the slice instead appending new data to it.

Do:
```go
c := "c"
ab := []string{"a", "b"}
newSlice := make([]string, 0, len(ab)+1) // len=0, cap=3
newSlice = append(newSlice, ab...)
newSlice = append(newSlice, c) // zero allocations
```
Don't:
```go
c := "c"
ab := []string{"a", "b"} // len=2, cap=2
newSlice = append(ab, c) // append creates to many memory allocations if slice capacity limit reached. len=3, cap=4
}
```

Returning part of slice from function may lead to memory leak.

Do:
```go
return oldSlice[1:6] // underlaying array of for example 1000 elements won't be collected by GC
```
Don't:
```go
newSlice := oldSlice[1:6]
return newSlice
```

Do not use pointers to slices. Slices are already reference types which point to an underlying array. If you want a function to modify a slice, then return that slice from the function, rather than passing a pointer.

### Buffering
Doing potentially expensive allocations of bytes.Buffer and []byte use of Buffer.Grow, Buffer.ReadFrom, oi.ReadAll, io.Copy.

Do:
```go
buf := bytes.NewBuffer(nil)
_, err = io.Copy(buf, resp.Body)
```
Don't:
```go
body, error := ioutil.ReadAll(response.Body)
```

### Concatenating
A strings.Builder is used to efficiently build a string using Write methods. It minimizes memory copying. The zero value is ready to use.

Do:
```go
// ZERO-VALUE:
//
// It's ready to use from the get-go.
// You don't need to initialize it.
var sb strings.Builder

for i := 0; i < 1000; i++ {
    sb.WriteString("a")
}
```
Don't:
```go
var res string

for i := 0; i < 1000; i++ {
    res += "a"
}
```

## Map handling
For composite map keys it's more efficient to use struct key rather then builded string.

Do:
```go
type structKey struct {
	a string
	b string
}
myMap[structKey{"a", "b"}] = value
```
Don't:
```go
myMap["a"+"b"] = value
```

## Goroutine handling
Goroutine usage with variables closure may lead to bugs. To avoid them pass variable inside the goroutine scope.

Do:
```go
var a string
go func(passedA string) {
	// process 'passedA' variable 
}(a)
```
Don't:
```go
var a string
go func() {
	// process 'a' variable using closure
}()
```

For goroutines that expected to return error please use 'errgroup' library to avoid bugs producing.

Do:
```go
g := errgroup.Group{}
g.Go(func() error {
    // process goroutine here and return error if it fails
	return err
})
g.Go(func() error {
    return err
})
err := g.Wait()
// error handling
```
Don't:
```go
errChan := make(chan error)
wg := sync.WaitGroup{}
wg.Add(1)
go func() {
	defer wg.Done()
	// some code here
	if err != nil {
		errChan <- err
}
wg.Add(1)
go func() {
    defer wg.Done()
    // some code here
    if err != nil {
    errChan <- err
}
wg.Wait()
for err := range errChan {
	// error handling
}
```

Always prefer synchronous functions by default. Async calls are hard to get right. They have no control over goroutine lifetimes and introduce data races. If you think something needs to be asynchronous, measure it and prove it. Ask these questions:

    - Does it improve performance? If so, by how much?
    - What’s the tradeoff of the happy path vs. slow path?
    - How do I propagate errors?
    - What about back-pressure?
    - What should be my concurrency model?

Do not create one-off goroutines without knowing when/how they exit. They cause problems that are hard to debug, and can often cause performance degradation rather than an improvement.

## Channels handling
To avoid panics and deadlocks while using unbuffered channels always process reading goroutine first.

Do:
```go
var myChan = make(chan string)
go func() {
	val := <- myChan
}()
myChan <- someValue
```
Don't:
```go
var myChan = make(chan string)
go func() {
    myChan <- someValue
}()
val := <- myChan
```

While waiting data from any type of channels try to avoid deadlock if data won't come.

Do:
```go
select {
case val := <- myChan:
	// process 'val'
case <- time.After(someTime):
	// pass or return error to be ensure that goroutines context switching will continue
}
```
Don't:
```go
val := <- myChan
```

While writing to buffered channel always check if its full to avoid deadlocks.

Do:
```go
var myChan = make(chan string, 10)
if len(myChan) < cap(myChan) {
    myChan <- someValue
}
```
Don't:
```go
myChan <- someValue
```

## Errors handling
By conventions error variables should be names with prefix 'Err' or 'err' and contain lowercase message.

Do:
```go
ErrConnLost := errors.New("connection to somewhere was lost")
```
Don't:
```go
ConnError := errors.New("Connection to somewhere was lost")
```

For architecture with many layers info about all layers must be included to error message for easy debugging purpose.

Do:
```go
return fmt.Errorf("handler: %w", err) // or
return fmt.Errorf("middleware: %w", err)
```
Don't:
```go
return err
```

Note that error message must not have 'error' word in it to avoid confusion between logging levels.

Do:
```go
ErrConnLost := errors.New("connection to somewhere was lost")
logger.Warning(ErrConnLost)
```
Don't:
```go
ErrConnLost := errors.New("error: connection to somewhere was lost")
logger.Warning(ErrConnLost) // so is it warning or error?
```

In case where function should gather all bunch of errors before failing please use 'multierror' package.

Do:
```go
var multierr multierror.Error

return multierr.Combine(err, err2, err3, err4, err5, err6) // or

for { // some loop
    multierr.Errors = append(multierr.Errors, err)
}
return multierr.ErrorOrNil()
```

Don't:
```go
return err // returning only first error or

return fmt.Errorf("%w %w %w", err, err2, err3) // wrapping error or contcat error messages
```

## Panic using
For this purpose, there is a built-in function panic that in effect creates a run-time error that will stop the program (but see the next section). 
Use it only if you need to stop your service at starting level if some of the crucial requirements weren't completed. This will help to notify developers team about problem.

Do:
```go
db, err := db.SetupDb()
if err != nil {
    panic(err)
}
```
Don't:
```go
db, err := db.SetupDb()
if err != nil {
    print(err)
}
```

Or use panic to avoid data malforming or business logic corruption.

Do:
```go
err := db.InsertCrucialData(data)
if err != nil {
    panic(err)
}
data.Flush() // deletes not stored data
```
Don't:
```go
err := db.InsertCrucialData(data)
if err != nil {
    print(err)
}
data.Flush() // deletes not stored data
```

## Log leveling
The purpose of logging is to provide observability - it enables the application communicate back to the administrator about what is happening. To communicate effectively logs should be meaningful and concise.
Critical level: This log-level represents the most severe situations when the service is entirely unable to continue operating. After emitting a critical log line, it is expected that the service will terminate.

Error level: This log-level is used when something unexpected has happened to the service, but it does not result in a total loss of service. Log lines using the error level must be actionable, so that the system administrator can investigate and resolve the incident. The error log level may indicate a loss of service for an individual user or request or it may indicate a total failure of a non-critical subsystem within the service.

Warn level: This log level is used to indicate that something unexpected has happened, but the server is able to continue operating and it has not suffered any loss of functionality as a consequence of this failure. System administrators may wish to investigate the cause of log lines at this level, but the need is typically less pressing than for those at error level. System administrators may also wish to monitor the rate of occurrence of individual log-lines at this level as this may be indicative of a wider problem. Log lines at the warning level should be as detailed as possible, since these are often the least clear-cut category of message.

Info level: This log level should be used to record normal, expected application behavior, even if it results in an error for the end user. They are not actionable individually, but the significant changes in the frequency of occurrence of individual log lines at this level may be indicative of a possible problem.

Debug level: This log-level is used for diagnostic information which may be used to debug issues but is not necessary for normal production system logging, nor actionable by system administrators.

Remember! Use logging only inside of dedicated layer of your app.

Do:
```go
// client.go
return err
// logic.go on top of client.go
return fmt.Errorf("logic: %w", err)
// handler.go on top of logic.go
logger.Errorf("handler: %v", err)
```
Don't:
```go
// client.go
logger.Error(err)
return err
// logic.go on top of client.go
logger.Error(err)
return err
// handler.go on top of logic.go
logger.Errorf("handler: %v", err)
```

## Configuring the app
For DevOps convenience make the app be able to parse environment variables along with command
line parameters.

Do:
```go
import (
	"github.com/spf13/viper"
    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Run: func (cmd *cobra.Command, args []string) {
        dbDSN := viper.GetString("db.dsn")
    }
}
func init() {
    rootCmd.PersistentFlags().String("db.dsn", "", "database address")
    viper.BindPFlags(rootCmd.PersistentFlags())
    viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    viper.AutomaticEnv()
}
func main() {
    if err := rootCmd.Execute(); err != nil {
        panic(err)
    }
}

```
Don't:
```go
func main() {
    flag.Parse()
	dbDSN := new(string)
    dbDSN := flag.String("dbaddr", "", "database address")
	if dbDSN == nil {
		*dbDSN = os.GetEnv("DB_DSN")
    }
}
```

# Designing the app
Please design your app according to [12 Factor App](https://12factor.net/)

# App layout
At Bequant commonly used below layout for Golang services:
    
    |- cmd/app // starting point of certain app
    |- internal // private packages that shouldn't be exported
    |- external // bequant packages that were imported
    |- domain // package for business logic object definitions
    |- dto // package for data transfer object definitions (may be merged in 'domain' package)
    |- client // package for sending data over the network
    |- handler // package for server's handlers definition
    |- deploy // folder for proper CI/CD configuration
    |- spec // folder for swagger API specification
    |- logger // package for logging tool definition
    |- metrics // package for metrics collection tool definition
    |- go.mod,go.sum
    |- .gitignore
    |- gitlab-ci.yml
    |- Dockerfile
    |- Makefile
    |- docker-compose.yml
    |- env.example
    |- README.md

# PR preparing
Any PR that can potentially have a performance impact on the Bequant codebase is encouraged to have a performance review. 
For more information, please see this link. The following is a brief list of indicators that should to undergo a performance review:

    - New features that might require benchmarks and/or are missing load-test coverage.
    - PRs touching performance of critical parts of the codebase (e.g. Hub/WebConn).
    - PRs adding or updating SQL queries.
    - Creating goroutines.
    - Doing potentially expensive allocations: bytes.Buffer and []byte:
        Use of Buffer.Grow, Buffer.ReadFrom, io.ReadAll.
        Creating big slices and maps without capacity when size is known in advance.
    - Recursion, unbounded, and deeply nested for loops.
    - Use of locks and/or other synchronization primitives.
    - Regular expressions, especially when creating regexp.MustCompile dynamically every time.
    - Use of the reflect package.

# Google style guide
For all topics that weren't covered by this guideline please look at [Official Google Go Style Guide](https://google.github.io/styleguide/go/index)

## Net/http Library basics

* net/http library allows you to build a basic web API in Go.
* It provides a servemux which is essentially a router or a request forwarder.
* `servemux.HandleFunc` routes request URL patterns to Go functions that execute the actual application logic. These functions are called handlers.
* The handlers take in an `http.ResponseWriter` object and a `*http.Request` pointer.
* `http.ResponseWriter` writes headers, response bodies in the HTTP response
* `*http.Request` reads data from the HTTP request including the method, headers, request data, etc.
* `http.NewServeMux()` allows you to instantiate a new serve mux object.
* `http.ListenAndServe(port str, servemux ServeMux)` method starts the web server and listens for incoming requests.




## Requests
* `r.URL.Get(query_param str)` allows you to fetch URL query parameters from the request object.

# Responses
* ##TODO Write about http.Error() shortcut


**The http.Handler Interface & ServeHTTP method

* In Go's net/http library a handler is what our multiplexer maps URL patterns to.
* If the multiplexer receives an incoming request which URL pattern matches a HandleFunc string, it'll call the handler function thats associated to that. 
* But what's a handler?
* A handler is an object which satisfies the `http.Handler` interface:
* ```
```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

* For an object to be a handler, it must have a `ServeHTTP()` method with that exact signature.

```
type home struct {}

func (h *home) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("This is my homepage"))
}
```

* You can write your handlers as structs with methods attached that implement `ServeHTTP()` method within them like in the example above
* But its long winded to create an object just to implement the `ServeHTTP()` method which is why we just use standalone functions generally.
* The next question arises; If for a function to be considered a type of Handler, it needs to implement a `ServeHTTP` function within but we don't do that.
* That's because we register our handlers with the multiplexers `HandleFunc` method.
* This method implements the `Handlerfunc` adapter function which automatically adds a `ServeHTTP()` method to our function. When executed, this `ServeHTTP()` method then calls the content of the original function. It's a round about way of coercing a normal function into satisfying the `http.Handler` interface.
* The `http.ListenAndServe()` method accepts the port to listen on in the form of a string and a Handler object type (an object that satisfies the `Handler` interface). However, we're passing in the servemux, not a handler.
* The servemux is a special kind of handler with its own `ServeHTTP()` method thus distinguishing it as a Handler and satisfying the Handler interface.
* When our server receives a new HTTP request, it calls the servemux's `ServeHTTP()` method. This looks up the relevant handler based on the request URL path, and in turn calls that handler's `ServeHTTP()` method.
* A Go web application is simply a chained bunch of `ServeHTTP()` methods starting with the servemux's who's role is to match and call other `ServeHTTP()` methods.

**Handler Dependencies & Dependency Injection

* Often our handlers will need dependencies passed in, in order to carry out their function. 
* A dependency is an object(s) that is required for certain logic to execute that is constructed and then passed into that logic rather than that logic also being responsible for the construction of the dependency itself. 
* For example, our handlers may need custom loggers, database connection pool, etc.
* These can be created outside and separate from our handlers and then passed in for our handlers to use.
* If our handlers and dependencies share the same package we can create a `struct` that contains all of our dependency objects (for example) handlers and then turn our handler functions into methods of that struct.
* For example:

```
package main

type application struct {
	errorLog *log.Logger
	infoLog *log.Logger
}
```

And then:
```
func (app *application) exampleHandler(w http.ResponseWriter, r *http.Request) {
	app.infoLog("Hello world")
}
```

* In order for this to work, the handlers and the `application` struct will need to share the same package. 
* But what if the handlers are in one package and the application structure is in another? We can use closures to handle this.
* The high overview is that when our application structure exists in one package and our handlers in another, we can no longer attach our handler function as methods to the application structure because we cant create methods on a structure that exists in another package. For reference that would look like this (which wont work):

```
package config

type Env struct {
	errorLog *log.Logger
	infoLog *log.Logger
}
```

```
package handlers

func (env config.Env) exampleHandler(w http.ResponseWriter, r *http.Request){
	env.infoLog("Hello world")
}
```

* This wont work because we cant access `config.Env` and make a method out of the handler like that.
* So what do we do?
* We can make separate packages between our handlers, config, and main packages and use closures in order to pass in the config struct and save the context.
* Here's the example:

loggers.go
```
package config

import "fmt"

// Defining a common interface for different types of loggers to satisfy.
type Logger interface {
	Log(message string)
}

// This is the main config struct that gets passed to our handlers
type Env struct {
	infoLog Logger
}

// Implementation of logger that logs to the console
type consoleLogger struct {}

func (l *consoleLogger) Log(message string) {
	fmt.Println(message)
}

// Constructor function to construct a new console logger.
func NewConsoleLogger() *consoleLogger {
	return &consoleLogger{}
}
```

handlers.go
```
package handlers

import (
	"api/config"
	"fmt"
	"net/http"
)

// This function gets passed the Env struct and returns the actual function that handles the request logic. This inner function has the context of Env because the outer function gets it passed in. This is the closure
func ExampleHandler(env *config.Env) http.Handler {
	return http.HandleFunc(func(w http.ResponseWriter, r *http.Request)) {
		if r.Method != "GET" {
			http.Error(w, http.StatusText(405), 405)
			return
		}
		env.infoLog.Log("Handling request...")
		return
	}
}
```

main.go
```
package main

import (
	"api/config"
	"api/handlers"
	"net/http"
)

func main() {
	infoLogger := config.NewConsoleLogger() // Calls the constructor function to create an instance of the consoleLogger struct and store it in infoLogger

	env := &config.Env{infoLog: infoLogger} // Store the consoleLogger struct to the instantiated Env struct here. 

	// Call the exampleHandler handler and pass in the env struct object.
	http.Handle("/", handlers.ExampleHandler(env))
}
```

* Essentially closures allow you to capture the env variable cross package boundaries and pass it into handlers.


**Centralized Error Handling

* A good practice would be to add helper methods into their own module responsible error handling common errors such as `500 Internal Server Error`'s
* This separates concerns by isolating and centralizing error handling logic within it's own module and not mixing that logic anywhere else like in the handlers themselves over and over or in the main function.

```
package main
import (
"fmt"
"net/http"
"runtime/debug"
)
// The serverError helper writes an error message and stack trace to the errorLog,
// then sends a generic 500 Internal Server Error response to the user.
func (app *application) serverError(w http.ResponseWriter, err error) {
	trace := fmt.Sprintf("%s\n%s", err.Error(), debug.Stack())
	app.errorLog.Println(trace)http.Error(w, http.StatusText(http.StatusInternalServerError),http.StatusInternalServerError)
}

// The clientError helper sends a specific status code and corresponding description
// to the user. We'll use this later in the book to send responses like 400 "Bad
// Request" when there's a problem with the request that the user sent.
func (app *application) clientError(w http.ResponseWriter, status int) {
	http.Error(w, http.StatusText(status), status)
}

// For consistency, we'll also implement a notFound helper. This is simply a
// convenience wrapper around clientError which sends a 404 Not Found response to
// the user.
func (app *application) notFound(w http.ResponseWriter) {
	app.clientError(w, http.StatusNotFound)
}
```


**Isolating and Encapsulating the routes

* Another good refactor in order to seperate out our concerns and promote isolation and encapsulation between our logic and module responsibilities would be to isolate our routes (currently defined in `main()`) into their own module.

```
cdm/web/routes.go
--------------------

package main

import "net/http"

// The routes() method reurns a servemux object with our application routes attached
func (app *application) routes() *http.ServeMux {

	// Construct a new servemux
	mux := http.NewServeMux()

	 fileServer := http.FileServer(http.Dir("./ui/static/"))
	 mux.handle("/static/", http.StripPrefix("/static", fileServer))

	mux.HandleFunc("/", app.home)
	mux.HandleFunc("/snippet/view", app.snippetView)
	mux.HandleFunc("/snippet/create", app.snippetCreate)

	return mux

}
```

```
cmd/web/main.go
-----------------

package main
...

func main() {
	addr := flag.String("addr", ":4000", "HTTP network address")
	flag.Parse()
	
	infoLog := log.New(os.Stdout, "INFO\t", log.Ldate|log.Ltime)
	errorLog := log.New(os.Stderr, "ERROR\t", log.Ldate|log.Ltime|log.Lshortfile)
	
	app := &application{
	errorLog: errorLog,
	infoLog:
	infoLog,
	}
	
	srv := &http.Server{
		Addr:
		*addr,
		ErrorLog: errorLog,
		Handler: app.routes(),
	}
	
	infoLog.Printf("Starting server on %s", *addr)
	err := srv.ListenAndServe()
	errorLog.Fatal(err)
}

```

* Now our main function is only responsible for parsing the runtime configuration parameters, creating the dependencies for the handlers and running the HTTP server.
* Routes are encapsulated in their own module and concerns are separated.


# 4. Database-driven Responses

* **What we'll learn:
	* Install a database driver to act as a middle-man between MySQL and our Go application
	* Connection to the MySQL DB for our web application using a pool of reusable connections.
	* Creating a standalone, isolated models package so that our database logic is reusable and decoupled from the rest of the application.
	* Learning and using the interfaces and functions in Go's `database/sql` packages.
	* Preventing SQL injection attacks by correctly using placeholder statements
	* Using transactions, so we can execute multiple SQL statements in one atomic action.







# General Go notes
* The internal directory:
	* The directory name `internal` carries a special meaning in Go. Any packages that live under this directory can only be imported by code inside the parent of the `internal` directory which in our case is the project root dir. Or in other words, packages in `internal` cannot be imported by code outside of our project.
	* ^ This prevents other codebases that aren't our own from importing and relying on the packages in our internal directory.
	* http.StripPrefix is a middleware wrapper that strips the given string out of the request URL and returns a handler for the updatedd (stripped) request.
	* HEAD request is an HTTP method that acts as a standard HTTP GET method but only returns the headers and not the body. Useful for large files. Like getting the Content-Length header and Content-Type header in order.
	* An Accept-Rangers header indicates that a client is able to perform partial requests on a particular resource. That is, they are able to request ranges of bytes out of a total of a resource.
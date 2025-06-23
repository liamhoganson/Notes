
## Net/http Library basics
* net/http library allows you to build a basic web API in Go.
* It provides a servemux which is essentially a router or a request forwarder.
* `servemux.HandleFunc` routes request URL patterns to Go functions that execute the actual application logic. These functions are called handlers.
* The handlers take in an `http.ResponseWriter` object and a `*http.Request` pointer.
* `http.ResponseWriter` writes headers, response bodies in the HTTP response
* `*http.Request` reads data from the HTTP request including the method, headers, request data, etc.
* `http.NewServeMux()` allows you to instantiate a new serve mux object.
* `http.ListenAndServe(port str, servemux ServeMux)` method starts the web server and listens for incoming requests.

**Requests**
* `r.URL.Get(query_param str)` allows you to fetch URL query parameters from the request object.

# Responses
* ##TODO Write about http.Error() shortcut


**The `http.Handler` Interface & `ServeHTTP` method

* In Go's net/http library a handler is what our multiplexer maps URL patterns to.
* If the multiplexer receives an incoming request which URL pattern matches a HandleFunc string, it'll call the handler function thats associated to that. 
* But what's a handler?
* A handler is an object which satisfies the `http.Handler` interface:
*```
``````go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
``````

* For an object to be a handler, it must have a `ServeHTTP()` method with that exact signature.

``````go
type home struct {}

func (h *home) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("This is my homepage"))
}
``````

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

``````go
package main

type application struct {
	errorLog *log.Logger
	infoLog *log.Logger
}
``````

And then:
```
func (app *application) exampleHandler(w http.ResponseWriter, r *http.Request) {
	app.infoLog("Hello world")
}
```

* In order for this to work, the handlers and the `application` struct will need to share the same package. 
* But what if the handlers are in one package and the application structure is in another? We can use closures to handle this.
* The high overview is that when our application structure exists in one package and our handlers in another, we can no longer attach our handler function as methods to the application structure because we cant create methods on a structure that exists in another package. For reference that would look like this (which wont work):

``````go
package config

type Env struct {
	errorLog *log.Logger
	infoLog *log.Logger
}
``````

``````go 
package handlers

func (env config.Env) exampleHandler(w http.ResponseWriter, r *http.Request){
	env.infoLog("Hello world")
}
``````

* This wont work because we cant access `config.Env` and make a method out of the handler like that.
* So what do we do?
* We can make separate packages between our handlers, config, and main packages and use closures in order to pass in the config struct and save the context.
* Here's the example:

``````go title:loggers.go
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
``````

``````go title:handlers.go
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
``````

``````go title:main.go
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
``````

* Essentially closures allow you to capture the env variable cross package boundaries and pass it into handlers.

**Centralized Error Handling

* A good practice would be to add helper methods into their own module responsible error handling common errors such as `500 Internal Server Error`'s
* This separates concerns by isolating and centralizing error handling logic within it's own module and not mixing that logic anywhere else like in the handlers themselves over and over or in the main function.


``````go title:helpers.go
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
``````

**Isolating and Encapsulating the routes

* Another good refactor in order to separate out our concerns and promote isolation and encapsulation between our logic and module responsibilities would be to isolate our routes (currently defined in `main()`) into their own module:

``````go title:routes.go

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
``````


``````go title:main.go
package main

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

``````

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



**Creating a database model (or creating a data access layer for out application)

* A note about dependencies in Go projects
	* Go project structures include a `go.mod` and `go.sum` files.
	* The `go.mod` file includes a dependency and its exact version being used for that specific project. If you had another code base on your machine within its own Go project structure, that used the same exact dependency but a different version, then that would be Ok as there would be no conflicts since the versioning is outlined in the `go.mod` file. These are akin to Python virtual environments.
	* It tells exactly what version should be used for `go run` `go test` or `go build` commands.
	* `go.sum` includes the cryptographic hashes for the packages in `go.mod` that are installed on your system
	* The `go mod verify` command compares the hash of the installed package to the hash contained within `go.sum` so you can see if the package has been altered in any way.
	* Someone else can run `go mod download` and they'll get an error if the packages they download mismatch the checksums defined in `go.sum`.
	* `go mod download`- Download the exact version of package as specified by the `go.mod` file.
	* `go mod verify` - Verify those packages to ensure they are installed correctly and not altered.
	* Whenever `go run`, `go test`, `go build` is ran, the exact package version listed in `go.mod` will always be used.
	* `go mod tidy` will automatically remove packages from `go.mod` and `go.sum` if those packages dont have any references within the codebase.

* As a reminder, the `internal` directory in our application is providing ancillary non-application specific code that can be re-used for other purposes; A data access layer that can be used by other applications can be used here

* **Designing a Database Model
	* At this stage in our project, we're going to initialize and store DB dependencies like a DB connection pool
	* This means that we are going to be connecting to and interacting with databases (writing to and reading from)
	* It would be a good design to isolate and encapsulate the logic for accessing this data.
	* Hence the database model, or in other words, the data access layer
	* This should be separate from our main function (outside of dependency initialization) and away from our handler who should be focused on HTTP requests and responses.
	* We can do this by creating a new module in `interal`: `/internal/models/snippets.go`
	* Here's the code:
	
``````go title:snippets.go
package models

import (
    "database/sql"
    "time"
)

// Snippet type to hold the data for a single snippet returned by the DB. Its fields match & coorespond to the fields in the DB.
// Data model that defines the structure of the data being inserted or returned from our DB
type Snippet struct {
    ID int
    Title string
    Content string
    Created time.Time
    Expires time.Time
}

type SnippetModel struct {
    DB *sql.DB
}


// A method of the SnippetModel type. This method will insert data into the DB.
func (m *SnippetModel) Insert(title string, content string, expires int) (int, error) {
    return 0, nil
}

// This will return a snippet based on it's ID
func (m *SnippetModel) Get(id int) (*Snippet, error) {
    return nil, nil
}

// Method that will return a list comprised of our defined Snippet types
func (m *SnippetModel) Latest() ([]*Snippet, error) {
    return nil, nil
}
``````

* We define a new package `models` and import the `sql` package for static typing
* We also define a `SnippetModel` struct which holds a reference to an `sql.DB` connection pool (the one we instantiate in main) and methods on that type for writing to and reading from the DB.
* We've also defined a `Snippet` struct which basically represents the keys in our DB table. Think of this as a data model that defines how data should be passed to and from the DB.
* Within main:

``````go title:main.go
package main

import (
    "database/sql"
    "flag"
    "log"
    "net/http"
    "os"
    "github.com/liamhoganson/StealthATS/internal/models"
    _"github.com/go-sql-driver/mysql"
)


// Configuration settings stored in a struct.
type config struct {
    addr string
    staticDir string // Directory containing static assets.
    dsn string
}

// Application struct to centralize application dependencies
type application struct {
    errorLog *log.Logger
    infoLog *log.Logger
    config *config
    snippets *models.SnippetModel
}

func main() {
    var cfg config // Declare cfg variable as an instance of the config struct.
    flag.StringVar(&cfg.addr, "addr", ":4000", "HTTP Network Address")// Use flag.StringVar to parse the command line flag into the memory address of cfg.addr.
    flag.StringVar(&cfg.staticDir, "static_dir", "./ui/static", "Relative file path for static content.")
    flag.StringVar(&cfg.dsn, "dsn", "web:web@unix(/var/lib/mysql/mysql.sock)/snippetbox?parseTime=true", "MySQL connection string")
    flag.Parse() // Run flag.Parse method to actually parse the CLI args.

    // Loggers
    infoLog := log.New(os.Stdout, "INFO\t", log.Ldate|log.Ltime)
    errorLog := log.New(os.Stderr, "ERROR\t", log.Ldate|log.Ltime|log.Llongfile)

    // Setup DB connection pool
    db, err := openDB(cfg.dsn)
    if err != nil {
        errorLog.Fatal(err)
    }
    defer db.Close()

    // Store application dependencies
    app := application{
        errorLog: errorLog,
        infoLog: infoLog,
        config: &cfg,
        snippets: &models.SnippetModel{DB: db},
    }

    srv := &http.Server{
        Addr: cfg.addr,
        ErrorLog: errorLog,
        Handler: app.routes(),
    }

    infoLog.Printf("Starting server on http://localhost%s", cfg.addr)
    err = srv.ListenAndServe()
    errorLog.Fatal(err)
}

// Wrapper function around sql.Open() and returns a DB connection pool
func openDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }
    if err = db.Ping(); err != nil {
        return nil, err
    }
    return db, nil
}
``````

* Here we've added a snippets field to our application dependencies struct which is of type `*models.SnippetModel` (The struct we've defined which holds our data access methods)
* We've also imported our `internal/models/snippets.go` module
* We instantiate our dependencies including the DB connection pool by calling `openDB`
* And we instantiate the the application struct dependency object and store it in the `app` variable. 
* Within that we've instantiated the `SnippetModel` struct in the snippets field within `app`: `snippets: &models.SnippetModel{DB: db}`
* Our handlers have access to the `SnippetModel` data access methods via dependency injection (injecting in that object to our application object) and can call any method on `snippets`
* And the data access methods have access to the DB connection pool through dependency injection as well via the `DB` field within `SnippetModel` object.
* And `Snippet`
* Because the data access code is defined as methods on an object, we can create an interface and mock a `SnippetModel` object for unit testing purposes.
* Side Note: An interface in Go can define multiple method signatures.

* **Benefits of this approach:
	* There's a clean separation of concerns with this design. Our data access layer isn't concerned with our HTTP handlers and business logic layer. 
	* Our HTTP handlers dont need to write any actual data access layer code themselves. Instead, we use dependency injection to pass in the the `SnippetModel` object and its associated methods into the application object and it's methods which are our handlers.

* **Section 4.6: Executing SQL Statements
* Go's `sql.DB` provides the following methods among others:
	* `DB.Query()` is used for SQL `SELECT` queries which return multiple rows
	* `DB.QueryRow()` is used for SQL `SELECT` queries which return a single row
	* `DB.Query()` is used for are used for statements that don't return data (usually used for mutation statements like `INSERT` and `DELETE`)
* We'll update the `SnippetModel.Insert()` method to the following:

``````go title:main.go
// A method of the SnippetModel type. This method will insert data into the DB.
func (m *SnippetModel) Insert(title string, content string, expires int) (int, error) {
    // We can split the statement into multi-line string using back ticks instead of quoutes.
    stmt := `INSERT INTO snippets (title, content, created, expires)
    VALUES(?, ?, UTC_TIMESTAMP(), DATE_ADD(UTC_TIMESTAMP(), INTERVAL ? DAY))`

    // Here we are using the embedded DB connection pool object from `SnippetModel`'s DB field and calling the `Exec` method
    // The first parameter to this method is the SQL statement we defined above, followed by the title, content, and expiry 
    // values. This method returns an sql.Result type, which contains infomration about the execution itself.
    result, err := m.DB.Exec(stmt, title, content, expires)
    if err != nil {
        return 0, err
    }
}
``````

* `sql.Exec()` method returns a `Result` type which is an interface that contains two methods:
	* `LastInsertID()` - This will return an integer value which should the row we've just added ID value. This is usually done by a DB's auto increment feature. Not supported by all DB's. 
	* `RowsAffected()` -  Again, not every DB will support this method, but this returns an integer value which represents the number of rows affected by a `INSERT`, `UPDATE` or `DELETE` statement.
* It's also common practice to ignore the return value of `sql.Exec()` like so: `_, err := m.DB.Exec(stmt, title, content, expires)`
* SQL Prepared Statements:
	* The above SQL query statement is what's called a prepared statement.
	* The first part of the snippet: `INSERT INTO snippets (title, content, created, expires)` is a standard INSERT statement 
	* The second part: `VALUES(?, ?, UTC_TIMESTAMP(), DATE_ADD(UTC_TIMESTAMP(), INTERVAL ? DAY)`
	* Is a section wherein placeholder values are formatted into the entire statement. The `?` values are placeholder values which get populated with the data from our application we send to the DB.
	* The entire statement is pre-compiled and stored on the DB which is good for us because statements like this are commonly used by our application to interact with the DB. By having the DB store it, it saves resources because it wont need to recompile it each time.
	* The statement is pre-compiled and parsed and the given values to replace our placeholders are sent separately which also helps prevent SQL injection attacks.
	* It helps preventing SQL attacks because the prepared statement is an SQL query that's *pre-compiled.* Meaning, the SQL code itself is parsed and the machine code derived from it on the DB itself, any data following the query to over-ride the placeholder fields will not be interpreted as SQL code EVEN if it's textually SQL code. For example, let's compare using a standard query from our application via string formatting that is NOT a prepared statement:
	
``````go title:main.go
package main
import "fmt"

func main() {
	data := 1
	sql_query = fmt.Sprintf(`SELECT * FROM users WHERE id=%d;`, data) 
}
`````` 

* The above code has a huge security vulnerability: When the string is formatted it returns: `SELECT * FROM users WHERE id=1` 
* That's fine for that specific variable (1), but there's a potential (especially when dealing with user input) that data is valid SQL code like so:

``````go title:main.go
package main
import "fmt"

func main() {
	data := '1; DROP table users;'
	sql_query = fmt.Sprintf(`SELECT * FROM users WHERE id=%d;`, data) 
}
``````

* Now that SQL query becomes: `SELECT * FROM users WHERE id=1; DROP TABLE users;`
* That query statement is what gets sent to the Database and our DB will interpret that as valid SQL code and execute it. That will delete the entire `users` table from our DB. This is an example of SQL injection.
* Whereas with a prepared statement: We define a prepared statement which is the valid SQL code we define and that gets sent to the DB in a single request, the DB compiles and stores that code and is prepared to execute that SQL statement with populated values sent in from an additional request. 
* Now this time:

``````go title:main.go
package main
import "fmt"

func main() {
	data := '1; DROP table users;'
	sql_query = (`SELECT * FROM users WHERE id=?;`) 
}
``````

* When this statement is sent to the DB as a prepared statement, the DB interprets the statement and stores it
* Next, the data `1; DROP table users;` gets sent and the DB uses that data for the placeholder value. 
* This will just result in an error now because its literally interpreting the id field as `1; DROP table users;` which is not a valid field.
* It knows not to parse `1; DROP table users;` and treat it just as textual data without compiling it and executing its instructions.
* The SQL statement and raw data are sent separately. So long as the original statement isn't  derived from an un-trusted source, injection cannot occur.


* **4.7 Single-Record SQL Queries
	* We'll run the following SQL query on the database in order to retrieve a single record based on it's ID:
	
	``````go title:snippets.go
	SELECT id, title, content, created, expires FROM snippets
	WHERE expires > UTC_TIMESTAMP() AND id = ?
	``````

	 * This query will return a snippet based off it's primary key (ID) and only if it's not already expired. If it doesnt exist, it will return nothing.
	 * We're also using the same placeholder pattern as before and sending the query and the data separate, making this a prepared statement.
	* Here's the full updated code for `snippets.go`:

``````go title:snippets.go
// This will return a snippet based on it's ID
func (m *SnippetModel) Get(id int) (*Snippet, error) {
    // This method's prepared statement
    stmt := `SELECT id, title, content, created, expires FROM snippets
             WHERE expires > UTC_TIMESTAMP() AND id = ?`

    // Use the QueryRow() method on our connection pool object and pass in the untrusted id parameter value
    // as the value to our placeholder value. This returns an sql.Row object.
    row := m.DB.QueryRow(stmt, id)

    // Initialize a zeroed Snippet struct.
    s := &Snippet{}

    // Use row.Scan() method to copy the values from each field in sql.Row into the cooresponding field 
    // in the Snippet struct. The args to row.Scan() are pointers to the place you want to copy the data into.
    // This populates our zeroed Snippet struct in place.
    err := row.Scan(&s.ID, &s.Title, &s.Content, &s.Created, &s.Expires)

    // If the query returns 0 rows, then row.Scan() will return a sql.ErrNoRows error.
    // We use the errors.Is() method to check if thats the error specifically, and return our own ErrNoRecord error instead.
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNoRecord
        } else {
            return nil, err
        }
    }

    // Return the new snippet object
    return s, nil
}
``````

* Code overview:
	* Behind the scenes, the `row.Scan()` method will convert the data types that `row` has stored as column values into native Go types.
	* Here's the `snippetView` method:

``````go title:handlers.go
func (app *application) snippetView(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.URL.Query().Get("id"))
    if err != nil || id < 1 {
        app.notFound(w)
        return
    }
    snippet, err := app.snippets.Get(id)
    if err != nil {
        if errors.Is(err, models.ErrNoRecord) {
            app.notFound(w)
        } else {
            app.serverError(w, err)
        }
        return 
    }

    fmt.Fprintf(w, "%+v", snippet)
}
``````

* Code overview:
	* Here we grab the ID parameter from the URL and check if it exists and its value is greater than 0.
	* We then call `app.snippets.Get(id)` and store the result in `snippet` which should be the resulting `Snippet` struct.
	* We then write the snippet data as a plain text HTTP response
	* As well as error handling.

* **The `defer` statement:**
	* Let's stop and take a moment to discuss the `defer` keyword in Go.
	* The `defer` statement is a list of function calls to be executed *after* the surrounding function returns.
	* For example, take this code snippet:
``````go title:defer_example.go
func copyFile(dstName, srcName string) (written int64, error) {
	src, err := os.Open(srcName)
	if err != nil {
		return
	}
	
	dest, err := os.Create(dstName)
	if err != nil {
		return
	}
	
	written, err = io.Copy(dst, src)
	dest.Close()
	src.Close()
	return
}
``````

* This code works but there's a bug within it. 
* First, if it cant open `src`, it will return out of this function stack
* That's fine because we haven't opened any other resource (including `src`)
* But if it cant create `dest`, and it returns out of the stack, `src` is already opened and that resource is already allocated in the heap as it returns before it calls `src.Close()`
* This is a resource leak.
* We can use the `defer` statement to ensure `src.Close()` and `dest.Close()` get called *after* `copyFile` returns at any point:

``````go title:defer_example_two.go
func copyFile(dstName, srcName string) (written int64, error) {
	src, err := os.Open(srcName)
	if err != nil {
		return
	}
	defer src.Close()
	
	dest, err := os.Create(dstName)
	if err != nil {
		return
	}
	defer dest.Close()
	
	written, err = io.Copy(dst, src)
	return written
}
``````

* Now let's examine what's happening in the above example.
* If there's an error when opening `src`, the `copyFile` function will return and `src.Close()` will be called since it's the only function call in the defer list.
* Now let's assume, it opens `src` successfully but it fails to create `dest` . In the last example, the `src.Close()` and `dest.Close()` happened after the function returned in the case that `err != nil`. 
* In this example, if `dest` fails to materialize, the `copyFile()` function exits (just like before) except now `dest.Close()` is executed and then `src.Close()` is executed next. Closing all the opened resources.
* The key thing with defer here is its useful for resource management and cleanup actions. 
* Another point worth noting is that deferred calls are executed in a *Last-In-First-Out* (LIFO) fashion. Meaning, the last defer statement evaluated is the first to be executed in the list. In the example above, `dest.Close()` is executed before `src.Close()`.
* Another point is that arguments passed into the `defer` function are evaluated as their state as they are passed into the defer call. For example:

``````go title:defer_example_three.go
package main

import "fmt"

func a() {
	i := 5
	defer fmt.Printf("\nThe deferred number is: %d", i)
	i++
	fmt.Printf("The number within the stack call is still: %d", i)
	return
}

func main() {
	a()
}
``````

* In this example, we pass the variable `i` into the deferred `Printf` function call and then immediately change it's value by incrementing it by 1.
* The output will be:

```
The number within the stack call is: 6
The deferred number is still: 5
```

* The stack executes in it's entirety and is returned and then the deferred `Printf` function call is called. 
* However, the value of `i` at time of passing it into the deferred `Printf` is `5`. Meaning that's what it will print AFTER function `a` executes.
* Function `a` will print `6` because we changed the value of `i` after deferring it.

**4.8 Multi-Record SQL Queries
* We'll use the following query within the `snippets.Latest()` method to retrieve the last 10 snippets inserted into the DB table so long as they're not expired:
```
SELECT id, title, content, created, expires FROM snippets
WHERE expires > UTC_TIMESTAMP() ORDER BY id DESC LIMIT 10
```

* Here's the full code:
``````go title:snippets.go
// Method that will return a list comprised of our defined Snippet types
func (m *SnippetModel) Latest() ([]*Snippet, error) {
    stmt := `SELECT id, title, content, created, expires FROM snippets
             WHERE expires > UTC_TIMESTAMP() ORDER BY id DESC LIMIT 10`

    // This returns a sql.Rows resultset containing the result of our query.
    rows, err := m.DB.Query(stmt)
    if err != nil {
        return nil, err
    }

    // Defer closing the underlying connection
    defer rows.Close()

    snippets := []*Snippet{}

    for rows.Next() {
        s := &Snippet{}
        err = rows.Scan(&s.ID, &s.Title, &s.Content, &s.Created, &s.Expires)
        if err != nil {
            return nil, err
        }
        snippets = append(snippets, s)
    }
    if err = rows.Err(); err != nil {
        return nil, err
    }
    return snippets, nil
}
``````

* Code overview:
	* In this code, we're using the `Query()` method on the connection pool object to execute the SQL statement. This returns a `sql.Rows` result set containing the results of our query stored in the variable `rows`
	* We then defer the `rows.Close()` call to ensure the underlying DB connection always gets closed regardless of where the function returns or errors.
	* We initialize an empty slice of `Snippet` objects.
	* Next we call `rows.Next()` method on the `*Rows` object on a for loop definition so we can use `row.Scan()` method on each result from the result set. From the docs: `Next()` prepares the next result row for reading with the `Rows.Scan()` method.
	* `roes.Next()` returns true if there is a next row or false if either there are no more rows to return OR it encountered an error.
	* After the loop we define: `if err := rows.Error(); err != nil` to check if at any point `rows.Next()` returned false was because of an error or just because it reached the end of the loop.
	* Within the loop, we are iterating over the results in the result set thanks to `rows.Next()` and using `rows.Scan()` to write the row column data values to our instantiated `&Snippet` pointer object. 
	* We instantiate one of these in each iteration and append the written object to the `snippets` slice we instantiated outside of the loop.
	* Once the loop has completed and we've determined there are no errors, we return the populated `snippets` slice.
	* `*Row.Scan()` method copies the values from each field in the row (each column value) to some place in memory that we specify. The arguments must be pointers to a place in memory (IN our case, we are passing in pointers to the fields in the `&Snippet` struct we instantiated and stored in `s`) AND the number of arguments passed in MUST match the number of columns in each row.

**Section 4.9 Transactions and other details
* The `database/sql` package
	* This is the package we've been using for interacting with our SQL database.
	* It provides a standard interface between our Go app and SQL databases.
	* It should generally work with other SQL databases like PostgreSQL and SQLite
* Working with Transactions
	* All the methods that come with `database/sql`for interacting with our DB can use any connection from the `sql.DB` connection pool EVEN if you have two calls to `Exec()` (Manipulating DB data) immediately next to each other.
	* There is no guarantee those two calls will use the same underlying DB connection from the `sql.DB` connection pool.
	* This doesn't work if you lock a table with a `LOCK TABLES` statement query because you must call `UNLOCK TABLES` on the same underlying DB connection that locked the table in order to avoid a deadlock.
	* In order to guarantee that the same underlying DB connection be used, you can wrap multiple SQL statements in a transaction.
	* You can run those same DB methods on a transaction object instead of a DB connection pool object. 
	* Here's an example code snippet describing the pattern:

``````go title:example_transaction.go
type ExampleModel struct {
	DB * sql.DB
}

func (m *ExampleModel) ExampleTransaction() error {
	// Calling Begin() on the DB connection pool creates a new sql.Tx
	// Object, which represents the in-progress database transaction.
	// gets stored in the tx var
	tx, err := m.DB.Begin()
	if err != nil {
		return err
	}

		// Defer a call to tx.Rollback() to ensure it is always called
		// before the function returns. If the transaction succeeds
		// it will lready be committed by the time tx.Rollback()
		// is called. Otherwise, in the event of an error, tx.Rollback()
		// will rollback the changes before the function returns
		// This makes the function idempotent. Either all changes 
		// succeed or none of them.
	defer tx.Rollback()

	_, err = tx.Exec(`INSERT INTO ...`)
	if err != nil {
		return err
	}

	// Carry out another transaction in exctly the same way
	// This is called on the same underlying connection as the last 
	// Exec() method.
	_, err = tx.Exec(`UPDTE ...`)
	if err != nil {
		return err
	}

	// If no errors, the statements in the transaction can be committed
	// to the database with the tx.Commit() method.
	err = tx.Commit()
	return err
}
``````

* You must always call either `Rollback()` or `Committ()` on the transaction object before the function returns. If you dont, the underlying connection will remain open and not be returned to the connection pool. This can easily lead to hitting your maximum connection limit and running out of connections from your connection pool and resources.
* Always defer a call to `tx.Rollback()` like we are in the above example.
* As mentioned previously, transactions are helpful if you want to execute multiple SQL statements as a single atomic action. So long as you defer a call to `tx.Rollback()` in the event of a failure or errors, the transaction ensures that ALL statements succeed or NONE of the statements succeed.
* IE database transaction objects allow for idempotency.
* In summary, a `Tx` object ensures multiple statements are executed on the same underlying connection.

* Prepared Statements
	* As described earlier, every one of our DB queries produces prepared statements that get stored on the DB pool underlying connections, then our parameters get passed in, the statement executes, then the prepared statement is torn down on the DB. (Parameters are not interpolated when using prepared statements.)
	* This happens every single time.
	* This is problematic for very large queries (IE multiple JOIN statements) and a large connection request is occurring (IE bulk inserting thousands of records into the DB.)
	* If we call that statement multiple times in our application, that can have a negative impact on the runtime performance.
		* We can instead, create our own prepared statement using the `sql.DB.Prepare()` method which returns an `sql.Stmt` object which represents the prepared statement.
		* This can be done on a transaction object as well but the stmt object produced will only last the duration of the transaction object. (IE trying to use the statement after calling `Tx.Close()` would produce an error.) 
		* Here's an example of using our own prepared statements on the `sql.DB` connection pool object:

``````go title:example_custom_prepared_stmt.go
type ExampleModel struct {
	DB *sql.DB
	InsertStmt *sql.Stmt
}

// Constructor function that returns an instantiated ExampleModel object
func NewExampleModel(db *sql.DB) (*ExampleModel, error) {

	// Define the prepared statement
	insertStmt, err := db.Prepare(`INSERT INTO ...`)
	if err != nil {
		return nil, err
	}

	// Store it in the ExampleModel object along w/ the connection pool
	return &ExampleModel{db, insertStmt}, nil
}

// Any methods implemented against the ExampleModel object will have
// access to the prepared statement (and the connection pool).
func (m *ExampleModel) Insert(args...) error {

	// We call Exec() here on the prepared statement instead of the DB pool
	// We can also call Query() and QueryRow() on the statement.
	_, err := m.InsertStmt.Exec(args...)
	return err
}

func main() {
	db, err sql.Open(...)
	if err != nil {
		errorLog.Fatal(err)
	}
	defer db.Close()

	exampleModel, err := NewExampleModel(db)
	if err != nil {
		errorLog.Fatal(err)
	}

	defer exampleModel.InsertStmt.Close()
}
``````


# 5. Dynamic HTML Templates

* What we'll learn:
	* This section focuses on displaying dynamic data that we fetch from the database and injecting it into our HTML templates.
	* Pass dynamic data into HTML templates using Go's `html/template` package to control the display of dynamic data
	* Create a template cache so that templates aren't being loaded from disk for each HTTP request.
	* Handle template rendering errors.
	* Create custom functions to format and display dynamic data in the HTML pages.

* **Displaying Dynamic Data
	* Currently, the `snippetView` handler function fetches a populated `models.Snippet` object from the database given an ID that it grabs from the URL query parameters and dumps the contents in a plain-text HTTP response.
	* Here's the updated `snippetView` handler:

	``````go title:handlers.go
func (app *application) snippetView(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.URL.Query().Get("id"))
    if err != nil || id < 1 {
        app.notFound(w)
        return
    }
    snippet, err := app.snippets.Get(id)
    if err != nil {
        if errors.Is(err, models.ErrNoRecord) {
            app.notFound(w)
        } else {
            app.serverError(w, err)
        }
        return 
    }

    files := []string{
        "./ui/html/base.html",
        "./ui/html/partials/nav.html",
        "./ui/html/pages/view.html",
    }

    ts, err := template.ParseFiles(files...)
    if err != nil {
        app.serverError(w, err)
        return
    }

    data := &templateData{
        Snippet: snippet,
    }

    err = ts.ExecuteTemplate(w, "base", data)
    if err != nil {
        app.serverError(w, err)
    }
}
``````

	* Next we need to create a new template page for our `snippet/view/` endpoint.
	* But first, lets explain how dynamic data is passed in and accessed in the HTML templates:
		* Within the HTML templates, any dynamic data that you pass in is represented by a dot. Literally a `.` character.
		* In this specific template, the underlying type of data (or the data that `.` represents) will be a `models.Snippet` struct.
		* When the underlying type of dot is a struct, you can render or yield the value of any field of the struct in the HTML templates by post-fixing dot with the field name.
		* For example, our `modles.Snippet` struct has a field `Title` so we can pass in the struct to our template and access the title of the snippet within the HTML page by writing `{{.Title}}` within the template.
	* Here's the created `view.html` page:

	``````html title:ui/html/pages/view.html
{{define "title"}}Snippet #{{.ID}}{{end}}
{{define "main"}}
	<div class='snippet'>
		<div class='metadata'>
			<strong>{{.Title}}</strong>
			<span>#{{.ID}}</span>
		</div>
		<pre><code>{{.Content}}</code></pre>
		<div class='metadata'>
			<time>Created: {{.Created}}</time>
			<time>Expires: {{.Expires}}</time>
		</div>
	</div>
{{end}}
``````

	* Something important to acknowledge and understand is that Go's `html/template` package only allows you to pass in one and only one item of dynamic data when rendering dynamic data.
	* Often we need to render multiple pieces of dynamic data on the same page.
	* An easy 'work around' to achieve this is to wrap all of your dynamic data in a single struct which acts as a holding structure for your data.
	* Here's an example of that holding data struct:
	
	``````go title:cmd/web/templates.go
package main
import "snippetbox.alexedwards.net/internal/models"
// Define a templateData type to act as the holding structure for
// any dynamic data that we want to pass to our HTML templates.
// At the moment it only contains one field, but we'll add more
// to it as the build progresses.
type templateData struct {
	Snippet *models.Snippet
}
``````

	* The function `snippetView` is already using this struct in the above example.
	* Now our data is contained in a `models/Snippet` struct *within* a `templateData` struct. We're now only passing one piece of data into the template; just a complex data structure.
	* To access the fields in the template, we need to traverse the field tree when post-fixing the dot:

	``````html title:ui/html/pages/view.html
{{define "title"}}Snippet #{{.Snippet.ID}}{{end}}
{{define "main"}}
	<div class='snippet'>
		<div class='metadata'>
			<strong>{{.Snippet.Title}}</strong>
			<span>#{{.Snippet.ID}}</span>
		</div>
		<pre><code>{{.Snippet.Content}}</code></pre>
		<div class='metadata'>
			<time>Created: {{.Snippet.Created}}</time>
			<time>Expires: {{.Snippet.Expires}}</time>
		</div>
	</div>
{{end}}
``````

* Additional Info about templates
	* Content Escaping
		* The `html/template` package automatically escapes any data that is rendered between `{{ }}`
		* This is very helpful in avoiding Cross-Site scripting attacks (XSS).
		* ^ It wont render and interpolate any external code on the web pages.
	* Nested Templates
		* When invoking one template from another, it's important that the dot needs to be explicitly passed in to the template being invoked.
		* You do this by including it at the end of each `{{template}}` or `{{block}}` like so:
		
	``````html title:example.html
{{template "main" .}}
{{block "sidebar" .}}{{end}}
``````

	* Calling methods
		* If the type that's being yielded/rendered between the `{{ }}` tags has a method against it, you can call these methods in the templates.
		* ^ As long as they are exported and return a single value.
		* For example, `.Snippet.Created` is a `time.Time` object which has multiple methods against it, one of these methods is `Weekday()`
		* You would call it in the template like this:
		* `<span>{{.Snippet.Created.Weekday}}</span>`
		* You can also pass parameters to methods like so:
		* `<span>{{.Snippet.Created.AddDate 0 6 0}}</span>`
		* Methods called in the template dont require parenthesis and parameters passed in are delimited by single white space.



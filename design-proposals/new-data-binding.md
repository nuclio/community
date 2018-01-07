# New Data Binding Architecture

**Goal:** Simplify the way developers access different forms of data and abstract the data API and language specific
implementation from the data source details to allow portability

## Overview

nuclio implements the concept of Data Bindings which allow users to specify the desired data connection details in the
function spec and access the data resources from the function without initializing the connection for every new event
and allowing the decoupling of function code from data sources and credentials which can be determined at deployment time.

The first-generation implementation is currently limited to Golang and iguazio V3IO driver, while other data sources 
can be added we feel we can come with a better design that will be simpler for both the developer (having simpler API) 
and people who want to develop plug-ins to different data sources. A new design can also be language independent 
allowing the use of data bindings in all supported coding languages. 

We see interest from multiple users and vendors to develop nuclio data bindings, and potentially use the same 
definitions or code in other projects which is why we choose to make the design process public and open for comments. 

# Data Binding Basics

The Data Bindings are divided to 4 main categories (others ones can be added):
* Object - Unstructured Object or File storage
* Table - SQL and NoSQL Databases, Structured files (CSV, Json, ..)
* Message - Stream and Message-queue
* Generic - Any other type, the request/response structures are user/plug-in defined

The user/operator provides the data connection details and security credentials in the **function spec** (or UI/CLI), those 
details are passed to the **data source plug-in driver** (implemented in Go) which establishes the data connection at the 
processor init phase, the plug-in can also report errors to the function or platform logs.  

A user/developer creates a request object using a lazy evaluation method (similar to Spark Data Frames), and submit it 
synchronously or asynchronously (non-blocking), the request returns a response or iterator object which can be consumed 
by the application, data is pre-fetched asynchronously to avoid redundant blocking.
 
Example API usage for reading data from a NoSQL DB:
```golang
   userTable := dc.Table.Read("db0://users", key1,key2).Select("user","addr","phone").Load()
   firstUser := userTable.Next()
```

The Data Source plug-in is running in separate Go routines and accepts the Go based request structure (nuclio will 
handle mapping requests from other languages like Python, Java, .. to Go structured using high speed and zero-copy 
ser/des based on Capn Proto or Flatbuf or C structs). It maps and submits the request to the native data source APIs, 
the responses are returned to the function runtime using a Go Channel or shared memory queue, the driver can pre-fetch 
more data based on the progress of the iterator. Plug-in drivers may also decide to cache data. 

The Data is passed between the plug-in and function runtime as zero-copy bytes buffer vector for unstructured data or 
serialized using a supported "encoder" (e.g. Json, Capn Proto, Protobuf, Arrow decoders) for tables/documents.

# Data Binding APIs by Category 

## Common Semantics

Data binding APIs are asynchronous, the client passes a request to the data source plug-in and can block or continue.
When using the `Do()` option the operation will block until a response is available, and will return a response object
On the other hand `DoAsync(waitgroup)` operation will not block and return a response object which can be read later,  
The waitgroup id will allow waiting for all the operations with the same Id using the `Wait(waitgroup)` operation.

Read operations do not block, the command returns a dataset/iterator object which can be read later, if the iterator 
object did not reach the end and the `.Next()` method cannot serve data it will block until data will become available.

Multiple options (`.Option(key, val interface{})`) can be added to almost any request type and will be passed to the 
plug-in driver as a `map[string]interface{}` structure.

## Path and Namespace semantics

Different services may have different namespace conventions (e.g. flat list of tables in a database, hierarchical in 
file/object directories, or two level with stream and shards/partitions), the interpretation of `path` is data source 
specific.

The path or its prefix (e.g. table name, directory, etc.) can be defined in the data binding spec (in the function.yaml)
or in the command (e.g. read/write), the actual path is a concatenation of the path in the spec and the path in the 
command, if the spec path is empty it will be the command path, if the spec fully defines the path (e.g. table name)
the path in the command must be an empry string, we can also have a path in the data binding spec (e.g. dir name) and
a path in the command for sub-dir and file name.

the command path is prefixed by the data binding name and "://", for example `db0://mytable`, `obj://path/to/object`,
if the data binding prefix is not specified we assume the first data binding.
  

## Object

### Requests: 

```golang
    // Object Get command
    Object.Get(path string, ranges ...int)
          .Condition(key, value string)
          .Load() 
          
    // Object Put command
    Object.Put(key string, data ...[]byte)
          .Add(data ...[]byte)
          .Metadata(meta map[string]string)
          .Do() | .DoAsync(wg int)
          
    // Object Delete command
    Object.Del(path string)
          .Condition(key, value string)
          .Do() | .DoAsync(wg int)
                   
    // Object Head command
    Object.Head(path string)
          .Condition(key, value string)
          .Do() | .DoAsync(wg int)

    // Object List command
    Object.List(prefix string)
          .Limit(limit int)
          .Delimiter (dl string)
          .Load() 
                   
```

### Responses:

Get Response:

```golang
type ObjGetResp interface {
    Error()     err    
    Size()      int    
    ReadAll()   []bytes
    Gen()       string 
    ...
}
```

List Response:

```golang
type ObjListResp interface {
    Error()      err    
    Next()       *ListEntry
    NextPath()   string
    HasNext()    bool 
    Close()    
    ...
}
```

Put, Del, and Head use the base response interface


## Stream

### Requests: 

```golang
    Stream.Consume(path string)
          .From(kind, value string)
          .Format(format string)
          .Where(filter string)
          .Select(fields ...string)
          .Load()

    Stream.Publish(path string)
          .Messages(messages ...*Message)
          .Buffers(bufs ...[]byte)
          .Do() | .DoAsync(wg int)
          
    Stream.Create(path string,name string, shards int) 
          .Do() | .DoAsync(wg int)
              
    Stream.Update(path string, shards int)          
          .Do() | .DoAsync(wg int)
              
    Stream.Delete(path string)
          .Do() | .DoAsync(wg int)  
          
    Stream.ListShards(path string)            
          .Do() | .DoAsync(wg int)  
```

### Responses:

Consume Response:

```golang
type StreamGetResp interface {
    Error()               err    
    Next()                *Message
    NextBytes()           []bytes
    Checkpoint(Message)   
    GetPosition(Message)  string 
    HasNext()             bool 
    Close()    
}
```

List Shards response:

`TBD`

Create, Del, Update, and Publish use the base response interface


## Table

Requests: 

```golang
    Table.Read(path string, keys ...string)
         .Format(format string)
         .Where(filter string)
         .Select(fields ...string)
         .sql(sqlquery string)
         .Partition(part, inParts int) 
         .Schema(schema *Schema)
         .Load()

    Table.Write(path string)
         .ToKeys(keys ...string) 
         .WithItems(items ...*Items)
         .WithExpression(expr string, attributes ...interface{})
         .Format(format string)
         .Condition(cond string)
         .Do() | .DoAsync(wg int)
  
    
    Table.Create(path string)
         .Schema()
         .PrimaryKey(partitionKey string,SortingKey string)
         .Do() | .DoAsync(wg int)        
    
    Table.Drop(path string)
    ...
    
```
### Responses:

Read Response:

```golang
type ObjGetResp interface {
    Error()       error    
    Next()        *Item   
    HasNext()     bool 
    Scan(fields string, pointers ...interface{}) error 
    Close()    
}
```

# Lower Level (Data Source Plug-in) API

## Request Structures 

The request structures are a representation of the data binding command/request structure, commands with specific 
metadata have their own `RequestDetails` structure with its type indicated by `CommandType`.

``` 
type DataRequest struct {
    Command         CommandType
    Path            string
    Options         *map[string]interface{}
    RequestDetails  interface{}
    RespChannel     chan ResponseMessage
}
```


## Response channel 

```golang
type ResponseMessage struct {
    RespCode    int
    Size        int
    Encoding    EncodintType
    IsArray     bool
    IsLast      bool
    Behind      int
    Buffer      []byte
}
```

## Detailed Request Structures

`TBD`

### Object Get

### Object Put

### Object List

### Table Read 

### Table Write

### Stream Retrieve

### Stream Send

# New Data Binding Architecture

**Goal:** Simplify the way developers access different forms of data and abstract the data API and language specific
implementation from the data source details to allow portability

## Overview

nuclio implements the concept of Data Bindings which allow users to specify the desired data connection details in the
function spec and access the data resources from the function without initializing the connection for every new event
and allowing the decoupling of function code from data sources and credentials which can be determined at deployment time.

The first-generation implementation is currently limited to Golang and iguazio v3io driver, while other data sources can be added we feel we can come with a better design that will be simpler for both the developer (having simpler API) and people who want to develop plug-ins to different data sources. A new design can also be language independent allowing the use of data bindings in all supported coding languages. 

We see interest from multiple users and vendors to develop nuclio data bindings, and potentially use the same definitions or code in other projects which is why we choose to make the design process public and open for comments. 

# Data Binding Basics

The Data Bindings are divided to 3 main categories (others or custom ones can be added):
* Object - Unstructured Object or File storage
* Table - SQL and NoSQL Databases, Structured files (CSV, Json, ..)
* Message - Stream and Message-queue
* Generic - Any other type, the request/response structures are user/plug-in defined

The user/operator provides the data connection details and security credentials in the function spec (or UI/CLI), those details are passed to the data-binding driver (implemented in Go) which establishes the data connection at the processor init phase, the plug-in can also report errors to the function or platform logs.  

A user/developer creates a request object using a lazy evaluation method (similar to Spark Data Frames), and submit it synchronously or asynchronously (non-blocking), the request returns a response or iterator object which can be consumed by the application, data is pre-fetched asynchronously to avoid redundant blocking.
 
Example API usage for reading data from a NoSQL DB:
```golang
   userTable := dbind.Table.Read("users", key1,key2).Select("user","addr","phone").Load()
   firstUser := userTable.Next()
```

The Data Source driver is running in separate Go routines and accepts the Go bases request structure (nuclio will handle mapping requests from other languages like Python, Java, .. to Go structured using high speed and zero-copy ser/des based on Capn Proto or C structs). It maps and submits the request to the native data source APIs, the responses are returned to the function runtime using a Channel, the driver can pre-fetch more data based on the progress of the iterator.

The Data is passed as bytes buffer vector for unstructured data or serialized using a supported "encoder" (e.g. Json, Capn Proto, Protobuf, Arrow decoders) for tables/documents.

# Data Binding APIs by Category 

## Common Semantics

Data binding APIs are asynchronous, the client passes a request to the plug-in driver and can block or continue.
When using the `Do()` option the operation will block until a response is available, and will return a response object
On the other hand `DoAsync(waitgroup)` operation will not block and return a request object which can be read later,  
The waitgroup id will allow waiting for all the operations with the same Id using the `Wait(waitgroup)` operation.

for Get or Read operations do not block, the command returns a dataset/iterator object which can be read later. 

Multiple options (`.Option(key, val string)`) can be added to almost any request type and will be passed to the plug-in driver as a `map[string]string` structure 

## Object

### Requests: 

```golang
    // Object Get command
    Object.Get(path string, ranges ...int)
          .Condition(key, value string)
          .Load() | .ReadAll()
          
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
    ReadAll()   []bytes
    ...
}
```

Put, Del, and Head Response:

List Response:

 

## Stream

### Requests: 

```golang
    Stream.Read(name, shardId)
          .From(kind, value string)
          .Format(format string)
          .Where(filter string)
          .Select(fields ...string)
          .Load()

    Stream.Write(name string)
          .Messages(messages ...*Message)
          .Buffers(bufs ...[]byte)
          .Do() | .DoAsync(wg int)
          
    Stream.Create(name string, shards int) 
          .Option(key, val string)
          .Do() | .DoAsync(wg int)
              
    Stream.Update(name string, shards int)          
          .Option(key, val string)
          .Do() | .DoAsync(wg int)
              
    Stream.Delete(name string)
          .Do() | .DoAsync(wg int)    
```

### Responses:

Read Response:

```golang
type ObjGetResp interface {
    Error()      err    
    Next()       *Message
    NextBytes()  []bytes
    ...
}
```

Create, Del, Update, and Write Response:



## Table

Requests: 

```golang
    Table.Read(path string, keys ...string)
         .Format(format string)
         .Where(filter string)
         .Select(fields ...string)
         .Option(key, val string)
         .Schema(??)
         .Load()

    Table.Write(path string)
         .Items(items ...*Record)
         .Format(format string)
         .Expression(expr string)
         .Condition(cond string)
         .Option(key, val string)
         .Schema(??)
         .Do() | .DoAsync(wg int)
    
    Table.Query(sql string)
    
    Table.Exec(sql string)    
    
    Table.Create(path string)
    ...
    
    Table.Drop(path string)
    ...
    
```

# Lower Level (Driver) API

## Request Structures 

## Response channel 

```golang
    RespCode    int
    Size        int
    Encoding    EncodintType
    IsArray     bool
    IsLast      bool
    Behind      int
    Buffer      *[]byte
```

## Detailed Request Structures

### Base/Simple Request
This is the base request class which can also be used for simple requests types 

### Object Get

### Object Put

### Table Read 

### Table Write

### Stream Read

### Stream Write

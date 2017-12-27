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

The user/operator provides the data connection details and security credentials in the function spec (or UI/CLI), those details are passed to the data-binding driver (implemented in Go) which establishes the data connection at the processor init phase, the plug-in can also report errors to the function or platform logs.  

A user/developer creates a request object using a lazy evaluation method (similar to Spark Data Frames), and submit it synchronously or asynchronously (non-blocking), the request returns a response or iterator object which can be consumed by the application, data is pre-fetched asynchronously to avoid redundant blocking.
 
Example API usage for reading data from a NoSQL DB:
```golang
   users := dbind.Table.Read(key1,key2).Select("user","addr","phone").Load("users")
   firstUser := users.Next()
```

The Data Source driver is running in separate Go routines and accepts the Go bases request structure (nuclio will handle mapping requests from other languages like Python, Java, .. to Go structured using high speed and zero-copy ser/des based on Capn Proto or C structs). It maps and submits the request to the native data source APIs, the responses are returned to the function runtime using a Channel, the driver can pre-fetch more data based on the progress of the iterator.   

# Data Binding APIs by Category 

## Object

## Table

## Stream

# Lower Level (Driver) API

## Request Structures 

### Base/Simple Request
This is the base request class which can also be used for simple requests types 

### Object Get

### Object Put

### Table Read 

### Table Write

### Stream Read

### Stream Write

# JSON TO JSON TRANSFORMATION

## Contents

[1.	Overview](#Overview)

[2.	Introduction](#introduction)

[3.	JOLT Custom Operations](#jolt-custom-operations)

[4.	Design](#design)

[5.	Guidelines for custom operation(Fincache)](#error-handling-in-fin-jxj)

[6.	Error Handling in FIN-JXJ](#error-handling-in-fin-jxj)

## Overview
   For a transaction to takes place, there are multiple calls from one system to another, calls can be between internal system or any external system may be involved. Output generated from one system is fed as input to another but the data structure between them may not be consistent. Payload expected is usually different from payload provided many times. This document proposes a framework to overcome this inconsistent behaviour due to multiple system calls.


## Introduction
   This framework will use JOLT library. It is a json to json transformation library written in java where the “specification” in itself is a json document. JOLT provides set of transformations:

- that can be "chained" together to form the overall JSON to JSON transform.
- focuses on transforming the structure of your JSON data, not manipulating specific values
    * The idea being: use Jolt to get most of the structure right, then write code to fix values
- consumes and produces "hydrated" JSON : in-memory tree of Maps, Lists, Strings, etc.
    * use Jackson (or whatever) to serialize and deserialize the JSON text.
    
It provides some stock transforms/operations:
1. shift
2. Default
3. Sort
4. Cardinality
5. Remove

One can directly use the combination of above operations and transform json into required form.

I. Shift : this operation is to shift json values or keys in the output structure. It reads in input and spec and based on the spec provided it will shift the provided values to the new keys.

II. Default: this operation will add default values to json at any level of that json.

III. Sort: Recursively sorts all maps within a JSON object into new sorted LinkedHashMaps so that serialized representations are deterministic.  Useful for debugging and making test fixtures.

IV. Cardinality: The CardinalityTransform changes the cardinality of input JSON data elements. The impetus for the CardinalityTransform, was to deal with data sources that are inconsistent with respect to the cardinality of their returned data.

V. Remove: This operation is to remove key values which are no longer required in json.

One can write their own transformation operation by just implementing interfaces provided by jolt. Class implements the Transform or ContextualTransform interfaces, and can optionally be SpecDriven (marker interface). If you want to use custom operation then you need to provide fully qualified java ClassName in operation field. It will invoke functionality of that class.

## JOLT Custom Operations

JOLT provides its set of operations to transform json. 
- It accepts input as a map or a list both.
- Spec object is always a list of operations. Output of one operation is provided as input to the next operation in the list.
- There is an optional parameter context. It is useful when defining custom jolt operations. The "context" allows transforms to tweak their behaviour based upon criteria outside of the input JSON object. Context will remain same for different input structures with different specs .
- For someone to write a custom operation, it requires to first list down the functionality required to be performed by the custom java class. Then rules or wildcards similar to the existing operations need to be defined which can be used while writing specs for different input output pair.
- Custom operations can be chained with the existing operations to achieve desired result. Once the operation is written one can directly write spec and start using it for their purpose.

**FinCache Fetch and Transform**
Below is a  custom implementation for fetching FinCache Object and transforming it with input Object and return.

 This is a spec driven custom implementation to fetch data from fincache and add it into input according to spec. The following Json is a Spec format of how finCache transformation needs to be called.
```
    [
      {
       "operation": "com.finacle.infra.jsonconv.FincacheCustomFetch",
       "spec": {
         "queryParams": {},
         "resultantFields": {},
         "fcoObject": ""
       }
      }
    ]
```

- Operation "com.finacle.infra.jsonconv.FincacheCustomFetch" is to identify that its a FinacheFetch. First parameter of spec i.e queryParams is to create a new object from input. Created object contains all keys required by FCO to fetch data. The generation of object is done by running shiftR operation of JOLT with object in “queryParams” as a spec and input msg as object to be transformed.

- Once the object is created we sort them alphabetically on keys and generate a search string which starts with “FC|” and  “<fcoObject>|” field from spec appended with stringified values of the sorted keys separated by “|”. Using this search string we will see if it exists in the cache service passed in context. If it exists we continue directly with the string in cache. Else we use the object created to fetch data from fincache, the output string returned will be stored in the cache and the same string is sent ahead for processing.

- The String forwarded is now parsed to create a java object. Apply shiftR again with the object in resultantFields as a spec and the java object as input. In the end we merge input with the latest object which got created by shitfr.

[Example here](example.md) gives you a step by step process of fincache Fetch and transform 


## Design

**FIN-J2J:** It is a Finacle‘s JSON to JSON transformation library, a wrapper over Java’s JOLT library. Interfaces for custom operations and plugins will be provided in this library. This java utility will be packaged as a jar. So that application who wants to use that functionality can directly include this jar in their package and can use default operation by directly calling the functions of this utility for transformation. It will take input and spec as a mandatory parameter but context is an optional parameter.
- **Input:** it is a JSON object that needs to be transformed and can have either a list or map. Input will be provided by the business application that wants to use this utility.
- **Spec:** It is a list of the JSON object. Each object in the spec specifies an operation and will have three things - operation name which will be a fully qualified package name in case of custom and keyword in case of default operation, spec which will be a set of rules for transformation under that operation and data source which is an optional field. The data source will have the information about the connection to be made in case any operation required to fetch something from someplace. This connection management will be handled by the application server while coming up. And each time while building a spec that application server will look for the data sources required by the custom operation and pass the connection reference to this utility along with the package name and on returning the handle by the custom operation jars, the application server will mark the connection free.
- **context:** It is an optional field, which will have the application's context and will be managed by the application.

**Custom Operation:** These are the custom java classes each specifying a dedicated operation. These will include FIN-J2J Library and will be packaged as a jar. Now, these jars can be built at the application's classpath and can be successfully loaded at runtime whenever operation mapped to them is called. Based on the custom operation each custom class will provide a set of guidelines for writing a spec.

**Applications:** These are the fincache application server which will include FIN-J2J utility in them for JSON to JSON transformation. As per present’s scope IOT-server and FEEB(Finacle Enterprise Bus) will use this library.



## Error Handling in FIN-JXJ
FIN-JXJ will internally handle errors and Runtime Exceptions returned by JOLT library. And it will return its own error object in case of error.
Jolt library internally has JoltException class which extends RunTimeException and this class is further extended by two different types of exception class.
- SpecException: It is thrown while processing or loading of spec.
- TransformException: It is thrown while mapping spec with input json to transform into output json.
FIN-JXJ will expose its own exception class for custom operations which will extend throwable class and will have additional non mandatory parameters.
All the exceptions generated in custom operations as well as JOLT library will finally be catched by FIN-JXJ and it will mapped error object to this custom exception class which will always return a common exception object to the application.

**FIN-JXJException:** Following are the additional parameters in this exception class which are non-mandatory and can be set by the custom classes while throwing this exception.
- ErrorCode: If in any case custom class wants to pass error code to the library can set in here and pass error code.
- ErrorMessage: error message will be set in here.
- ExceptionClass: Fully qualified Package Name of the class throwing exception will be passed here.
- OperationIndex: Index no. of the operation will be passed here on which exception has occurred.
FIN-JXJ will always return FIN-JXJException object in case of any runtime exceptions. If any operation fails then the execution of the operations will stop and it will return back the handle to the application with an exception and null output.
In case of default operations provided by the JOLT will return SpecException and TransformException, but FIN-JXJ will mapped their error object to its custom exception class but in other cases it will throw the error object as it is. To custom operations Only FIN-JXJException is exposed and they will always return that exception in their processing. Other exceptions will also be finally mapped to this exception only. 


# FINJXJ (Finacle JOLT)

## Contents

[1.	Overview](#Overview)

[2.	Introduction](#introduction)

[3.	JOLT Custom Operations](#jolt-custom-operations)

[4.	Design](#design)

[5.	Guidelines for custom operation(Fincache)](#error-handling-in-fin-jxj)

[6.	Error Handling in FIN-JXJ](#error-handling-in-fin-jxj)

[7.	Developer Section](#Developer-section)

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

- Once the object is created we sort them alphabetically on keys and generate a search string which starts with “FC|” and  “<fcoObject>|” field from spec appended with stringified values of the sorted keys separated by “|”. Using this search string we will see if it exists in the cache service passed in context([Check below At Cache Handling to know how it works](#cache-handling). If it exists we continue directly with the string in cache. Else we use the object created to fetch data from fincache server (The call is made to the URL passed in context in key "fcurl"), the output string returned will be stored in the cache and the same string is sent ahead for processing.

- The String forwarded is now parsed to create a java object. Apply shiftR again with the object in resultantFields as a spec and the java object as input. In the end we merge input with the latest object which got created by shitfr.

   This Jolt operation needs a context to work. Context here is a hashmap which contains the following data mandatorily
```
   {
     "fcurl":"http://localhost:8080"
   }
```
   ##### Cache Handling
   This Custom operation as an optional LRU caching option using "guava". This works if a object of Class "Cache<Object, Object>" passed in the above context with key "cache". Cache needs to be instantiated in calling application and should be stored to reuse the object across requests. Cache is instantiated by running the command below.
```
 Cache<Object, Object>  cache = CacheBuilder.newBuilder()
   .maximumSize(10)     //Number of objects which can be stored in this LRU cache.
   .expireAfterWrite(5, TimeUnit.MINUTES)   // Maximum time a record can exist in the cache.
   .build();
```   
   If the above cache is passed finCache data is first checked in this object, if found will be used directly else it will fetch from the fincache store it back to the cache and then use it.

NOTE: This plugin only handles single JSON objects so if the fincache returns a arrayList the processing considers only the first element 

#### Exceptions which can occur in fincacheFetch
- J009 : Unkown error : can occur if fincache server doesnt respond or gives some unkown error.
- J010 : Invalid spec data : cacn occur if any of these fields are missing ins spec queryParams or resultantFields or fcoObject or if query params couldnt be formed 
- J011 : Invalid context data : fincache Url is not passed 
- Apart from these any error which comes in Fincache fetch will be passed back.


The following example gives the step by step process of the process goes on
- **Sample Spec** 
```
[
  {
    "operation":"com.finacle.infra.jsonconv.FincacheCustomFetch",
    "spec": {
      "queryParams": {
        "bankId": "dcId",
        "accountDetails": {
          "sb": {
            "accountId": "acctNum"
          }
        }
      },
      "resultantFields": {
        "acctName": "acctName",
        "crncyCode": "currencyCode",
        "custId": "custId",
        "address": {
          "address1": "address.area",
          "address2": "address.City"
        }
      },
      "fcoObject": "LoanAccountDetailJson"
    }
  }
]
```

- **Input** 
```
{
  "bankId": "0001",
  "accountDetails": {
    "sb": {
      "accountId": "1234567890"
    }
  }
}
```

- **Expected Output**
```
{
  "bankId": "0001",
  "accountDetails": {
    "sb": {
      "accountId": "1234567890"
    }
  },
  "acctName": "Test",
  "currencyCode": "INR",
  "custId": "C0000001",
  "address": {
    "area": "JP Nagar",
    "City": "Bangalore"
  }
}
```
**The following steps are applied on spec and input :**

-  During the first transformation for fetch using query params, the following JSON is created
```
   {
     "dcId": "0001",
     "acctNum": "1234567890”
   }
```
- Following is the search string which is created using the object created previously  
```
     FC|LoanAccountDetailJson|1234567890|0001
```
- If the search string is available in cache we fetch the corresponding value or fetch the object using object created previously and go ahead with the transformation 
```
 {
  "dcId": "0001",
  "acctNum": "1234567890",
  "acctName": "Test",
  "crncyCode": "INR",
  "paidAmt": 5000,
  "pendingAmt": 40000,
  "custId": "C0000001",
  "tsCnt": 1,
  "address": {
    "address1": "JP Nagar",
    "address2": "Bangalore"
  }
}
```
-  Applying Resultant params Shiftr we get the following 
```
 {
  "acctName": "Test",
  "currencyCode":  "INR",
  "custId": "C0000001",
  "address": {
    "area": "JP Nagar",
    "City": "Bangalore"
  }
}
```
-  Merging the object with original input we get the following.
```
{
  "bankId": "0001",
  "accountDetails": {
    "sb": {
      "accountId": "1234567890"
    }
  },
  "acctName": "Test",
  "currencyCode": "INR",
  "custId": "C0000001",
  "address": {
    "area": "JP Nagar",
    "City": "Bangalore"
  }
}
```



## Design

![Design](https://github.com/revoorunischal/Test/blob/master/Design.png "Design")



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
All the exceptions generated in custom operations as well as JOLT library will finally be catched by FIN-JXJ and it will map error object to this custom exception class which will always return a common exception object to the application.

**FIN-JXJException:** Following are the parameters in this exception object returned by the Fin-JXJ library. All the custom operation will always set their exception data in this object and jolt internal exceptions are also mapped to this object.
- ErrorCode: If in any case custom class wants to pass error code to the library can set in here and pass error code or JOLT library errors are also mapped to error codes defined in this class. And in case of jolt exception error code will start from 'J' but in case of custom exception all the error code will have unique prefix for each operation.
- ErrorMessage: error message will be set in here.
- ExceptionClass: Fully qualified Package Name of the class or the operation name throwing exception will be passed here.
- ErrorType: There are four error types defined by FinJXJ Exception which are FATAL, INFO, WARNING AND 
  ERROR. So based on the nature of error custom classes can set error type.
- Details: It is a map object. If anybody wants to pass any additional information they can pass here. OperationIndex is aprt of this map object. 
- OperationIndex: Index no. of the operation will be passed here on which exception has occurred.
FIN-JXJ will always return FIN-JXJException object in case of any runtime exceptions. If any operation fails then the execution of the operations will stop and it will return back the handle to the application with an exception and null output.
In case of default operations provided by the JOLT will return SpecException and TransformException, but FIN-JXJ will mapped those errors FINJXJ exception class but in other cases it will throw the error object as it is. To custom operations Only FIN-JXJException is exposed and they will always return that exception in their processing. Other exceptions will also be finally mapped to this exception only. 
Following is the list of error code and messages, FinJXJ library will throw in case of JOlT related exceptions:
%s will be replaced by the internal error thrown by operations.
	- J001 : Error occured while processing jolt shift operation. %s
	- J002 : Error occured while processing jolt remove operation. %s
	- J003 : Error occured while processing jolt default operation. %s
	- J004 : Error occured while processing jolt sort operation. %s
	- J005 : Error occured while processing jolt cardinality operation. %s
	- J006 : Error occured while processing jolt modify operation. %s
	- J007 : Invalid operation 
	- J008 : Invalid Input Parameters %s
	- J009 : Unknown Error
	- J010 : Invalid spec
	- J011 : Invalid context data

In case of custom operation they will have their own list of error code and error message.

## Developer Section
FinJXJ jar can be included by any project by adding this dependency:
<dependency>
	<groupId>com.finacle.finjxj</groupId>
	<artifactId>finjxj-definitions</artifactId>
	<version>0.0.1-SNAPSHOT</version>
</dependency>
This interface accepts message - which needs to be transformed and spec - which is list of opertaions that will be used to do transformation. It also accepts context which is an optional parameter. context is provided by the caller application in case if it wants to do something specific to custom operation like to build a application cache, cache object is required which can be passed in context.Any urls to make api calls in custom operation will aslo be part of context.

**Sample code snippet to call the library :**

```
package com.finacle.App;
import com.finacle.transform.jsonconv.FinJXJ;
public class App 
{
	public static void main( String[] args )
    		{
           		Object output = FinJXJ.transform(spec, input, context);
    		}
}
```

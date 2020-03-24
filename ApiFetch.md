**Api data fetch and Transform**

Below is a Jolt custom opertaion for fetching Data from Rest API in GET mode and Transforming it with input Object and return.

Note: Assumptions here is that the end server accepts GET requests with optional query params and return the data in JSON format.

 This is a spec driven custom implementation of JOLT operation to fetch JSON data using an API and add it into input according to spec. 
 The following Json is a Spec format of how this transformation needs to be called.
```
    [
      {
       "operation": "com.finacle.infra.jsonconv.APIDataFetch",
       "spec": {
         "service":"",
         "path": "",
         "queryParams": {},
         "resultantFields": {}
       }
      }
    ]
```
   
Here operation says the custom opertaion which needs to be called.
Spec paramteres have the following uses.

- service : This is a key to the JSON which contains the URL's(IP address or DomainName) to the respective services.
eg : {"service":"core_service"}

- path : This is the endpoint of the service request. This can contain a dynamic parameter which will be evaluated with input.
Eg: {"path":"/Account/${accountDetails.sb.accountId}/balance"} 

- queryParams : This is of JOLT shift format which is to create a new JSON. This JSON data is converted into queryparams and sent to the server.
eg : {"bankId":"dcId","accountDetails":{"sb":{"accountId":"acctNum"}}}

- resultantFields : This is of JOLT shift format which is to create a new JSON. This JSON is later appended to the input. 

eg : {"resultantFields":{"acctName":"acctName","crncyCode":"currencyCode","custId":"custId"}}

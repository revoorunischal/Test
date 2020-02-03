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

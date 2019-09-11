---
layout: page
title: Akamai Cloud Embed
subtitle: Lab
sitemap:
  exclude: "yes"
---


## Prerequisites
* Pre-installed [docker instance](https://developer.akamai.com/introduction/Setup_Environment.html#docker)
* jq [command-line JSON processor](https://stedolan.github.io/jq/download/)
* Luna User
* API Credentials



## Lab Setup

Before we start with WSD we need to know the ID of the main configuration. We can achieve this by using the function of search within Property's Managers API's.

The following exercises use the docker instance listed above that has [httpie](https://github.com/jakubroztocil/httpie) and [httpie-edgegrid](https://github.com/akamai/httpie-edgegrid) pre-configried.

### Setup Credentials

We need to create the API user/credentials.

Steps.
1. Login to [Luna Control Center](https://control.akamai.com/) with the provided usernames/passwords. 
2. Find your user in the account AU Media Delivery.
![alt text](https://roymartinez.dev/img/ace/step1labsetup.jpg)
3. Create a ***"New API client for me"***
![alt text](https://roymartinez.dev/img/ace/step2labsetup.jpg)
4. Give your user "admin" role within the "WSD" Group.
![alt text](https://roymartinez.dev/img/ace/step3labsetup.jpg)
5. Give it a name "AKAU" and assigned the following Read/Write Authorizations.
    * Property Manager
    * CPS
    * Wholesale Delivery API
    * Subcustomer Policy
    * Contracts-API_Contracts [Read-Only]
    ![alt text](https://roymartinez.dev/img/ace/step4labsetup.jpg)
6. Now create the api credentials. Copy this **FAST** or download them since they **will close after 120 seconds** of creation.
![alt text](https://roymartinez.dev/img/ace/step5labsetup.jpg)


### JQ setup
jq is like `sed` for JSON data - you can use it to slice and filter and map and transform structured data with the same ease that `sed`, `awk`, `grep` and friends let you play with text.

To install we need to run the following commands:
1. First Update
```sh
Akamai API Bootcamp >> apt-get update
```
2. Install JQ
```sh
Akamai API Bootcamp >> apt-get install jq
```

### httpie-edgegrid setup

With the new credentials we will now need to craete the file "/.edgerc" within the docker instance to use on the examples to follow.
1. Log into your docker instace 
```sh
docker run -it akamaiopen/api-kickstart
```
![alt text](https://roymartinez.dev/img/ace/step6labsetup.jpg)

2. Create the file edgerc where the credentials will be placed and paste the credentials with the header "[au]" . All httpie commands to follow will use the creadentials assigned under "[au]" with the argument -a.
```sh
vim ~/.edgerc
```
![alt text](https://roymartinez.dev/img/ace/step7labsetup.jpg)

To exit vim:
1. Press : (colon). The cursor should reappear at the lower left corner of the screen beside a colon prompt.
2. Enter the following: wq!

## Property Manager API

 This operation searches properties by name, or by the hostname or edge hostname for which it is currently active.

### Find Property

[Akamai Developer API Documentation](https://developer.akamai.com/api/luna/papi/resources.html#postfindbyvalue)

***Full Response***

As you can see in the example below we pass in the POST JSON body the field property name with the value to be searched.

Command:
```sh
http --auth-type edgegrid -a au: POST :/papi/v1/search/find-by-value propertyName=au.wholesale.akamaized.net
```
or 
```sh
echo '{"propertyName": "au.wholesale.akamaized.net"}' | http --auth-type edgegrid -a au: POST :/papi/v1/search/find-by-value
```
Response:
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 277
Content-Type: application/json;charset=UTF-8
Date: Tue, 07 Aug 2018 17:08:53 GMT
Server: Apache-Coyote/1.1
Vary: Accept-Encoding

{
    "versions": {
        "items": [
            {
                "accountId": "act_B-C-1HP9L57", 
                "assetId": "aid_12345678", 
                "contractId": "ctr_C-1234567", 
                "groupId": "grp_123456", 
                "note": "Changed Hostname", 
                "productionStatus": "ACTIVE", 
                "propertyId": "prp_123456", 
                "propertyName": "au.wholesale.akamaized.net", 
                "propertyVersion": 2, 
                "stagingStatus": "ACTIVE", 
                "updatedByUser": "rmartine", 
                "updatedDate": "2018-08-30T18:47:56Z"
            }, 
            {
                "accountId": "act_B-C-1HP9L57", 
                "assetId": "aid_12345678", 
                "contractId": "ctr_C-1234567", 
                "groupId": "grp_123456", 
                "note": "Enabled Dynamic Web Content", 
                "productionStatus": "INACTIVE", 
                "propertyId": "prp_123456", 
                "propertyName": "au.wholesale.akamaized.net", 
                "propertyVersion": 3, 
                "stagingStatus": "INACTIVE", 
                "updatedByUser": "rmartine", 
                "updatedDate": "2018-08-31T14:39:28Z"
            }
        ]
    }
}

```

If the property/configuration is found you will get a body with meaningful data otherwise as seen below the items array will be empty.
```json
{
    "versions": {
        "items": []
    }
}
```

The following command is the same as above but we use [json.tool](https://docs.python.org/3/library/json.html#module-json.tool) and *[jg](https://stedolan.github.io/jq/)* to get only the data we need.

Command:
```sh
echo '{"propertyName": "au.wholesale.akamaized.net"}' | http --auth-type edgegrid -a au: POST :/papi/v1/search/find-by-value | jq '.versions.items[0].propertyId'
```
Response:
```json
"prp_123456"
```
## WholeSale API/Cloud Embed

On the next session we will start refering to SubCustomer Configurations and Policies. A Subcustomer is a client of Akamai's partner. 
A company that gives cloud services has a customer that has owns the website www.example.com. Example.com will be the Subcustomer to Akamai and the cloud service provider will be Akamai's direct partner. That said our partner (the Cloud service provider) will own creating a set of rules that tells Akamai that:
1. We should accept the traffic (Register the Subcustomer) 
2. How to handle the traffic for that specific domaim (SubCustomer Policy).
 
**GET DATA**
Now that we used [PAPI](https://developer.akamai.com/api/luna/papi/overview.html) to get the main WSD Configuration we can now start to use the [Akamai Cloud Embed](https://control.akamai.com/dl/customers/WSD/WSD2/GUID-6635996F-106F-47B7-9C18-F694B76EA910.html) API's to create or view our Sub-Customer Policies.


### GET PROPERTY SUB-CUSTOMERS

Because we know the property ID "prp_123456" we can see if we have any Sub-Customers registered against it. When using the following API's we don't need the "prp_" portion of the ID but only the integers.

Pay attention to the API endpoint, the component "ID" will be replaced with the property ID. 

/partner-api/v2/network/production/properties/***ID***/customers/

Command:
```sh
http --auth-type edgegrid -a au: :/partner-api/v2/network/production/properties/123456/customers/
```
Response:
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 41
Content-Type: application/json
Date: Fri, 03 Aug 2018 15:01:53 GMT
Server: Yaws 1.99
X-Akamai-Request-Time-Sec: 1.520261

[
    {
        "geo": "US", 
        "subcustomer": "WSD-002"
    }
]
```

### GET Partner Config sub customers

Now that we know the Subscustomer ID we need to find out the SubproperyID to fetch the policy.

Command:
```sh
http --auth-type edgegrid -a au: :/partner-api/v2/network/production/properties/123456/sub-properties/
```
Response:
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 70
Content-Type: application/json
Date: Fri, 03 Aug 2018 15:04:46 GMT
Server: Yaws 1.99

[
    {
        "SubPropertyID": "rmartine.partnercustomer.biz", 
        "customerID": "WSD-001"
    }
]
```

###  GET CUSTOMER POLICY

With both the Customer ID and Property we can now fetch the policy for the given subcustomer.

Command:
```sh
http --auth-type edgegrid -a au: :/partner-api/v2/network/production/properties/123456/sub-properties/rmartine.partnercustomer.biz/policy X-Customer-ID:WSD-001
```
Response:
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 953
Content-Type: application/json
Date: Sun, 09 Sep 2018 15:42:49 GMT
Server: Yaws 1.99
X-Akamai-Request-Time-Sec: 1.495029
X-Customer-ID: WSD-001 (unregistered)

{
    "rules": [
        {
            "behaviors": [
                {
                    "name": "origin", 
                    "params": {
                        "cacheKeyType": "digital_property", 
                        "cacheKeyValue": "-", 
                        "digitalProperty": "rmartine.partnercustomer.biz", 
                        "hostHeaderType": "origin", 
                        "hostHeaderValue": "-", 
                        "originBasePath": "-", 
                        "originDomain": "akamaiflowershop.com"
                    }, 
                    "value": "-"
                }, 
                {
                    "name": "caching", 
                    "type": "fixed", 
                    "value": "10d"
                }
            ], 
            "matches": [
                {
                    "name": "url-wildcard", 
                    "value": "/*"
                }
            ]
        }
    ]
}
```

 **Updating/Creating** 

### Register Sub-Customer a new Customer

Now we will create/register a new subCustomer.

Command:
```sh
http --auth-type edgegrid -a au: PUT :/partner-api/v2/network/production/properties/123456/customers/WSD-001 geo=US
```
Response:
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 13
Content-Type: application/json
Date: Sun, 09 Sep 2018 15:57:02 GMT
Server: Yaws 1.99
X-Akamai-Request-Time-Sec: 2.524155

{
    "geo": "US"
}

```
Validate creation! If it was created you should see your ID in the response of the following command.

The response has the same ID as in the query and if there was an issue then the response will be empty.

Command:
```sh
http --auth-type edgegrid -a au: :/partner-api/v2/network/production/properties/475285/sub-properties/WSD-001 | jq '.[] | select(.customerID=="WSD-001") | .customerID'
```
Response:
```json
"WSD-001"
```
**Comon issues**
Case matters!

Command:
```sh
http --auth-type edgegrid -a au: PUT :/partner-api/v2/network/production/properties/123456/customers/WSD-001 GEO=US
```
Response:
```json
HTTP/1.1 400 Bad Request
Connection: close
Content-Type: application/problem+json
Date: Fri, 03 Aug 2018 15:34:13 GMT
Server: Yaws 1.99
X-Akamai-Request-Time-Sec: 1.063085

{
    "description": "Input Json Validation Errors. refer to 'errors' field.", 
    "error": [
        {
            "code": 4010, 
            "field": [
                "geo"
            ], 
            "message": "'geo' is a required property"
        }
    ], 
    "errorCode": 4000, 
    "errorInstanceId": "e17db8af95", 
    "helpLink": "https://developer.akamai.com/api/delivery-policies/errors.html#4000", 
    "message": "Invalid Json input"
}
```

**Create/Upload Policy**

Now that we have customer created we need to create a Policy that tells Ghost what to do this customer's traffic.

We've created a sample policy that can be modified with the supported matches and behaviors.
* [Content Policy supported Behaviors](https://control.akamai.com/dl/customers/WSD/WSD2/GUID-F3672A20-3EF1-4485-BE62-A9799858A5D5.html)
* [Content Policy supported Matches](https://control.akamai.com/dl/customers/WSD/WSD2/GUID-9716205B-A8A3-4984-8F84-9034A830BAFF.html)
**Steps**
1. Download Sample Customer Policy or create your own.
```sh
wget http://training.wsd.edgesuite.net/examples/policy.json
```
Update the downloaded Policy. Change the digitalProperty field from:
```json
"digitalProperty": CHANAGEME,
```
to {?username}.partnercustomer.biz, example:
```json
"digitalProperty": "wsd-003.partnercustomer.biz"
```

2. Upload the policy to the newly created customer and 

Command:
```sh
cat policy.json | http --auth-type edgegrid -a au: PUT :/partner-api/v2/network/production/properties/123456/sub-properties/{?username}.partnercustomer.biz/policy X-Customer-ID:WSD-001
```
Response:
```json
{
    "rules": [
        {
            "behaviors": [
                {
                    "name": "origin", 
                    "params": {
                        "cacheKeyType": "digital_property", 
                        "cacheKeyValue": "-", 
                        "digitalProperty": "rmartine.partnercustomer.biz", 
                        "hostHeaderType": "origin", 
                        "hostHeaderValue": "-", 
                        "originBasePath": "-", 
                        "originDomain": "akamaiflowershop.com"
                    }, 
                    "value": "-"
                }, 
                {
                    "name": "caching", 
                    "type": "fixed", 
                    "value": "1d"
                }
            ], 
            "matches": [
                {
                    "name": "url-wildcard", 
                    "value": "/*"
                }
            ]
        }
    ]
}
```



### Validate Customer Policy
Now we need to make sure that what we uploaded is correct, we will validate it by fetching the currently policy and reviewing it.

Command
```sh
http --auth-type edgegrid -a au: :/partner-api/v2/network/production/properties/123456/sub-properties/rmartine.partnercustomer.biz/policy X-Customer-ID:WSD-001
```
Response:
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 946
Content-Type: application/json
Date: Sat, 04 Aug 2018 04:19:24 GMT
Server: Yaws 1.99
X-Akamai-Request-Time-Sec: 1.538577
X-Customer-ID: WSD-001 (unregistered)

{
    "rules": [
        {
            "behaviors": [
                {
                    "name": "origin", 
                    "params": {
                        "cacheKeyType": "digital_property", 
                        "cacheKeyValue": "-", 
                        "digitalProperty": "rmartine.partnercustomer.biz", 
                        "hostHeaderType": "origin", 
                        "hostHeaderValue": "-", 
                        "originBasePath": "-", 
                        "originDomain": "akamaiflowershop.com"
                    }, 
                    "value": "-"
                }, 
                {
                    "name": "caching", 
                    "type": "fixed", 
                    "value": "1d"
                }
            ], 
            "matches": [
                {
                    "name": "url-wildcard", 
                    "value": "/*"
                }
            ]
        }
    ]
}
```


### Update Customer Policy

We will change the TTL from 1 day to 10 days.
 
 To do this we will need to first update the policy.json file with the new TTL.
```sh
vim policy.json
```
To Edit:
* Press 'i' to enable edit mode and update ``` "value": "1d"``` to ```"value": "10d"```
To exit vim:
* Press : (colon). The cursor should reappear at the lower left corner of the screen beside a colon prompt.
* Enter the following: wq!

After this we will upload the new policy
```sh
cat policy.json | http --auth-type edgegrid -a au: PUT :/partner-api/v2/network/production/properties/123456/sub-properties/{?username}.partnercustomer.biz/policy X-Customer-ID:WSD-001
```
Response
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 948
Content-Type: application/json
Date: Sat, 04 Aug 2018 04:24:53 GMT
Server: Yaws 1.99
X-Akamai-Request-Time-Sec: 2.718232

{
    "rules": [
        {
            "behaviors": [
                {
                    "name": "origin", 
                    "params": {
                        "cacheKeyType": "digital_property", 
                        "cacheKeyValue": "-", 
                        "digitalProperty": "rmartine.partnercustomer.biz", 
                        "hostHeaderType": "origin", 
                        "hostHeaderValue": "-", 
                        "originBasePath": "-", 
                        "originDomain": "akamaiflowershop.com"
                    }, 
                    "value": "-"
                }, 
                {
                    "name": "caching", 
                    "type": "fixed", 
                    "value": "10d"
                }
            ], 
            "matches": [
                {
                    "name": "url-wildcard", 
                    "value": "/*"
                }
            ]
        }
    ]
}
```

### Validate Policy update

```sh
http --auth-type edgegrid -a au: :/partner-api/v2/network/production/properties/123456/sub-properties/rmartine.partnercustomer.biz/policy X-Customer-ID:WSD-001 |  jq '.rules[0].behaviors[1].value'

"10d"
 ```

### Enable Integrated Cloud Accelerator

Update the policy again and add the following json to enable Adaptive Image Compression, Prefetching, and SureRoute on the base rule.

```json
{
	"name": "content-characteristics",
	"params": {
		"mobileImageCompressionEnabled": true,
		"prefetchEnabled": true,
		"realUserMonitoring": false,
		"sureRouteTestObjectPath": "/akamaiflowershop/information/sitemap/index.html"
	},
	"type": "dynamic-web-content",
	"value": "-"
}
```
## Browser Testing

We are going to use Google Chrome as the browser for the following tests.

Install the following Extensions:
1. [Akamai Debug Headers](https://chrome.google.com/webstore/detail/akamai-debug-headers/lcfphdldglgaodelggpckakfficpeefj?hl=en), this will insert Akamai Pragma headers to the request.
2. [HTTP Mod Headers](https://chrome.google.com/webstore/detail/modheader/idgpnmonknjnojddfkpgkljpfnnfcklj?hl=en), since we are obscuring the pragma headers we will add a Cookie as shown below to expose these headers.
![Modheaders](https://roymartinez.dev/img/ace/modheaders.jpg) 

Now load the your digital property on the broswer, if all goes well and you configured it correctly you will see AkamaiFlowerShop website.
![enter image description here](https://roymartinez.dev/img/ace/AkamaiFlowers.jpg)

If not then you will see the following 404 page.
![enter image description here](https://roymartinez.dev/img/ace/baseconfig.jpg)

### Validate ICA

Command:
```sh
curl 'http://rmartine.partnercustomer.biz/' -H 'Pragma: akamai-x-get-client-ip, akamai-x-cache-on, akamai-x-cache-remote-on, akamai-x-check-cacheable, akamai-x-get-cache-key, akamai-x-get-extracted-values, akamai-x-get-nonces, akamai-x-get-ssl-client-session-id, akamai-x-get-true-cache-key, akamai-x-serial-no, akamai-x-feo-trace, akamai-x-get-request-id' -H'Cookie: akpragma_allowed=true' -vks -o /dev/null  2>&1 | grep WEB_
```

As you can see below we can see the specific features of ICA and if they were applied.

Response:
```sh
< X-Akamai-Session-Info: name=WEB_IMAGE_COMPRESSION_OUT_IMAGE_COMPRESSION_INVOKED; value=true
< X-Akamai-Session-Info: name=WEB_LASTLY_INVOKE_AIC; value=true
< X-Akamai-Session-Info: name=WEB_OUT_ICA_POLICY_APPLIED; value=true
< X-Akamai-Session-Info: name=WEB_OUT_ICA_SUREROUTE_APPLIED; value=true
< X-Akamai-Session-Info: name=WEB_OUT_ICA_SUREROUTE_INVOKED; value=true
< X-Akamai-Session-Info: name=WEB_PREFETCH_OUT_PREFETCH_APPLIED; value=true
< X-Akamai-Session-Info: name=WEB_PREFETCH_OUT_PREFETCH_INVOKED; value=true
```
## Cleanup

After we are done with the lab we will clean the policies and subCustomer that were registered. Something to point out is that once a Registration or Policy is removed from Akamai there is no way to get this back. This means that the Partner should have a backup of the Registrations and Policies created to prevent any accidental loss of data.
 
### Delete Policy

First we will start by deleting the policy for the subcustomer domain.

Command:
```sh
http --auth-type edgegrid -a au: DELETE :/partner-api/v2/network/production/properties/123456/sub-properties/{?property_id}/policy
```
Response:
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 152
Content-Type: application/json
Date: Sun, 09 Sep 2018 16:26:49 GMT
Server: Yaws 1.99
X-Akamai-Request-Time-Sec: 2.617096

{
    "description": "The policy for property_id '123456' and subproperty_id '{?property_id}' was successfully deleted.", 
    "message": "Successfully deleted"
}
```

### Delete SubCustomer 

Now, the last step is to clean up the resgitration of the Subcustomer.
Command:
```sh
http --auth-type edgegrid -a au: DELETE :/partner-api/v2/network/production/properties/123456/customers/{?customerid}
```
Response:
```json
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 148
Content-Type: application/json
Date: Sun, 09 Sep 2018 16:32:09 GMT
Server: Yaws 1.99
X-Akamai-Request-Time-Sec: 2.034028

{
    "description": "The subcustomer for property_id '123456' and subcustomer_id '{?customerid}{?customerid}' was successfully deleted.", 
    "message": "Successfully deleted"
}
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbNzQ2NTg2MDY3LDY2ODA0MjU5MCwtNTU3Nj
g2NjY2LC0xMzk0MzE5NTgyLDE5NDM1NjgyNzQsLTE4NTcyMTYx
MjAsLTI5MTE3ODM5OSwxNzI4NzE3MjEyLDExNDU2NjkyNDEsMj
EzNDI0Mzg0OCwtMTM2OTY0MDE5NywtMTg0ODkwMzIzNywtMTU2
MjY0ODU1LC01MDE5Nzg5ODEsLTE1ODcwNjg4NzQsMTgzMTM0OT
kxNywxMDI0MjY0NTg3LDE0MzQ4MTkwMTEsMTgyNTk4ODc2Niwt
NzQzMDgwMDIxXX0=
-->
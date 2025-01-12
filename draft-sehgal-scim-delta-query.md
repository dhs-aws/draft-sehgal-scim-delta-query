---
title: "Delta Query "
abbrev: "SCIM Delta Query"
docname: draft-sehgal-scim-delta-query-01
category: std

ipr: trust200902
area: IETF
workgroup: SCIM
keyword: [Internet-Draft, SCIM]

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]
submissionType: IETF

author:
   -
    name: Anjali Sehgal
    organization: Amazon Web Services
    email: anjalisg@amazon.com
    role: editor
   -
    name: Danny Zollner
    organization: Microsoft
    email: zollnerd@microsoft.com
 
  

normative: 

informative: 


--- abstract

This document defines extensions to the System for Cross-domain Identity Management (SCIM) standard [RFC7643] [RFC7644] to enable incremental retrieval of resources that have been updated or deleted in a SCIM service provider. This allows for more efficient interactions between SCIM clients and service providers and addresses problems that have inhibited large-scale implementation of use cases such as synchronization, entropy detection, and identity reconciliation.


--- middle

# Introduction

The document describes additions to the SCIM standard to provide a scalable and accurate method of change detection that allows a SCIM client to retrieve the current state of all resources that have changed since a prior point. Some of the possible use cases where this can be used is building identity reconciliation systems or incremental synchronization systems where a client periodically pulls data from the server. For example, synchronizing data from human capital management systems into a central identity management service.

SCIM clients provision identity information such as Users, Groups and memberships to SCIM service providers that is then used for authorization decisions when attempts to access resources occur. Potential synchronization inaccuracies could lead to data divergence between the SCIM client and SCIM service provider. Undetected diverging data between a SCIM client and SCIM service provider can lead to undesirable authorization decisions. For instance, an undetected failure to synchronize group membership removal between a SCIM client and a SCIM service provider can lead to access being incorrectly granted to an application that should no longer be allowed. The SCIM standard does not provide any guidance for performing ongoing incremental data reconciliation or synchronization, and the existing functionality in the SCIM standard does not meet the accuracy, efficiency or scalability requirements of many implementers. 

Without a set of end-to-end reconciliation processes, SCIM implementers cannot confidently say that there are no divergence-caused security incidents. Providing a mechanism to detect data divergence and reconciliation mechanism is of the utmost importance to avoid any authorization decisions being made with incorrect data. This data divergence detection may be used for reporting purposes or may be extended to either trigger provisioning of those resources in the target system or pulling changes from the target system into the source.

This document proposes additions to the SCIM standard that can be implemented across SCIM service providers over time, allowing SCIM clients to build synchronization and reconciliation mechanisms that they can reuse across all SCIM service providers that support the capabilities proposed in this document. The logic for divergence detection as part of any synchronization or reconciliation mechanism is out of scope of this document and is left to the implementer. 

# Notational Conventions

{::boilerplate bcp14-tagged}

# Delta Query Usage in Divergence Detection

## Terminology

Full Scan - Retrieval of the current state of all resources that exist without providing a delta token or timestamp-based restriction on which resources should be returned. This may be a request such as GET /Users to retrieve all User resources, GET (baseurl)/ to retrieve all resources regardless of type, or a query with a filter such as GET /Users?filter=department eq "Accounting".

Delta Scan - Retrieval of the current state of all resources modified (created, updated, deleted) since a previous full or delta Scan that returned a delta token.

Delta Token - An opaque artifact generated by the server and provides a point of reference that can be used by the issuing SCIM service provider to identify a point after which modified (created, updated or deleted) resources should be returned.

SCIM Client -
An HTTP client that initiates SCIM Protocol [RFC7644] requests and receives responses.

SCIM Service Provider - 
An HTTP server that implements SCIM Protocol [RFC7644] and SCIM Schema [RFC7643].

## Divergence Detection via Full Scans
A simplistic implementation of a divergence detection tool may perform a full comparison of data between source and destination systems containing identity data. To do this, the divergence detection tool can perform a full Scan of all resources in both systems and then perform the following checks. First it will evaluate if there are any resources (Users, Groups..) in one of the systems that are missing in the other one, and then it will evaluate if attribute values for any resources present in both the source and destination systems have different values.

This simplistic solution can quickly iterate across all of the resources in the source and destination systems and perform detailed data comparison, providing the highest accuracy. However, the speed at which this can be performed will be limited by factors such as system capacity, scheduling algorithms, API page sizes, and throughput limits. This makes an approach that utilizes only full Scans too slow (hours/days) for datasets spanning millions of resources, causing delayed detection of any data divergence. For instance, a data divergence check for a directory with 10 million resources could require 3 hours to scan one of the systems, while divergence detection within 30 to 60 minutes would be more optimal. To address this challenge for larger datasets, the approach utilizing only full Scans can be amended with an additional set of recurring delta Scans that would identify divergence in recently modified resources.

## Divergence Detection via Delta Scans
A recurring set of delta Scans can be used to provide ongoing detection of of data divergence between source and destination systems. Each individual delta Scan will only retrieve data that has been modified after the issuance of the delta token value used by the SCIM client. Continuous successful delta Scans run over a given period of time allows for ongoing detection of data modified within that period. The process can be used to incrementally retrieve changes and identify and repair any divergences as needed, with only delta Scans being required after the first full Scan.

A delta Scan can be done using the Delta Query functionality introduced in this document. The aim of these additions is to allow the client to instruct the service provider to only return the current state of objects that have changed (newly added, updated or deleted) since the issuance of the delta token provided by the client. 


# Delta Query
A Delta Query is a query performed on underlying SCIM resources that enables the client to discover newly created, updated, or deleted resources without performing a full Scan from the server. This approach uses a delta token generated by a SCIM service provider. The delta token is an opaque artifact, or "watermark". It provides a point of reference that can be used by the issuing SCIM service provider to identify a point after which modified (created, updated or deleted) resources should be returned. 

## Query Parameters and Response Attributes

The following table describes the URL query parameters for delta query requests:

| Parameter | Description |
deltaQuery | A boolean type that indicates that the client is requesting the server to execute a full scan or delta scan and return a delta token with its response.|
deltaToken | A string type that may be provided by the client to request only records modified after the point represented by the delta token's value. The value of deltaToken MUST be treated as opaque by the client. Token values must follow the unreserved characters set defined in section 2.3 of [RFC3986]. 
{: title="Query Parameters"}

The following attribute is added to the schema of urn:ietf:params:scim:api:messages:2.0:ListResponse.

nextDeltaToken
  : A string type that MUST be returned by the server on the last page during a delta query response. If the SCIM service provider supports delta query, this attribute MUST be returned by a when the query parameter deltaQuery is True OR a 400 series error must be returned //NOTE BE MORE SPECIFIC HERE. Values must only contain characters from the unreserved characters set defined in section 2.3 of [RFC3986]. 

The following attribute is added to the sub-attributes of the common attribute "meta".

isDeleted
: A boolean type. This attribute MUST be returned and MUST have a value of True when the resource has been deleted from the SCIM service provider and is being returned as part of a delta query response. This attribute has a "returned" property value of "request" when the associated resource has not been deleted. //NOTE - Should be a part of a short section on deleted resources and associated behaviors, not just a new attribute

# Using Delta Query to track changes

## Obtaining the First Delta Token
A client will typically prepare for establishing recurring delta query requests by first performing a full Scan on the SCIM service provider. The GET request used for the initial full Scan MUST include the deltaQuery parameter and MUST not include the deltaToken parameter. 

In response to the full scan query the server
 
1. MUST return the resources that currently exist in the collection. Resources that have been created and deleted prior to the initial full scan query won't be returned. Resources returned will represent the latest state of the resource at the time processing of the request.
2. MUST Return the nextDeltaToken on the last page of the full scan response.

~~~
GET Users?deltaQuery&count=50
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD

HTTP/1.1 200 OK
Content-Type: application/scim+json
{
  "totalResults":45,
  "itemsPerPage":50,
  "nextDeltaToken": "VTHKLOUTREO",
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
  "Resources": [{
    ...
  }]
}
~~~

## Using a Delta Token to Perform a Delta Scan

After a full Scan, the nextDeltaToken value returned by the service provider may be used by the client to perform a delta Scan, querying for resources modified since the issuance of the delta token. The GET request used for the delta Scan MUST include the deltaQuery parameter and MUST include the deltaToken parameter.

In response to the delta scan query the server
 
1. MUST Return the resources modified (created, updated or deleted) after the point represented by the delta token's value. The resources returned are represented in the response using their standard representation and reflect their current state.
2. MUST Return the nextDeltaToken on the last page of the delta scan response. 

~~~
GET Users?deltaQuery&deltaToken=VTHKLOUTREO&count=50
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD

HTTP/1.1 200 OK
Content-Type: application/scim+json
{
  "totalResults":13,
  "itemsPerPage":50,
  "nextDeltaToken": "OPUTREWSFDE",
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
  "Resources": [{
    ...
  }]
}
~~~

In the above example request and response, the query used in the full Scan example was repeated with the addition of the deltaToken parameter and the value of the delta token provided in the response to the full Scan via the nextDeltaToken attribute.  

# Pagination 

## Using Cursor-based Pagination During a Full Scan
When the total number of resources returned by the query is large enough, a paginated response may be required. In this scenario, a service provider supporting cursor-based pagination MAY return a value for the nextCursor attribute. The cursor can then be used by a client in a subsequent request to obtain the next page of results.    

~~~
GET Users?deltaQuery&count=50
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD

HTTP/1.1 200 OK
Content-Type: application/scim+json
{
  "totalResults":457,
  "itemsPerPage":50,
  "nextCursor": "CVHNJKUYFRT",
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
  "Resources": [{
    ...
  }]
}
~~~

When the client is ready to retrieve the next page of results it SHOULD query the same service provider endpoint with all query parameters and values remaining identical to the initial query with the exception of the cursor parameter and its value.

~~~
GET Users?deltaQuery&count=50&cursor=CVHNJKUYFRT
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD

HTTP/1.1 200 OK
Content-Type: application/scim+json
{
  "totalResults":457,
  "itemsPerPage":50,
  "previousCursor:"CVHNJKUYFRT",
  "nextDeltaToken":VTHKLOUTREO",
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
  "Resources": [{
    ...
  }]
}
~~~

## Paginating a Delta Scan Response

### Cursor-based Pagination
If the number or resources that were modified (Created, Updated, or Deleted) after the issuance of the delta token used in a delta Scan is large, a service provider supporting cursor-based-pagination MAY paginate the response and return a value for the nextCursor attribute. When the client is ready to retrieve the next page of results it SHOULD query the same service provider endpoint with all query parameters and values remaining identical to the initial query with the exception of the cursor parameter and its value.

The below example request and response shows the first request to redeem a delta token as part of a delta Scan. 

~~~
GET Users?deltaQuery&deltaToken=VTHKLOUTREO
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD

HTTP/1.1 200 OK
Content-Type: application/scim+json
{
  "totalResults":8649,
  "itemsPerPage":100,
  "nextCursor": "CVHNJKUYFRT",
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
  "Resources": [{
    ...
  }]
}
~~~

In the response portion of the above example, the response is limited in size by the service provider even though the client did not request a paginated response. 

In the below example, the client uses the cursor value obtained in the previous example to retrieve the next page of results.

~~~
GET Users?deltaQuery&deltaToken=VTHKLOUTREO&cursor=CVHNJKUYFRT
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD

HTTP/1.1 200 OK
Content-Type: application/scim+json
{
  "totalResults":8649,
  "itemsPerPage":63,
  "prevCursor": "CVHNJKUYFRT",
  "nextDeltaToken": ALEMQPXNAZX",
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
  "Resources": [{
    ...
  }]
}
~~~
The results of the query can be progressed through by repeating the original query while continuously updating the cursor value. When returning the last page of results, the service provider will omit the nextCursor value and will include the nextDeltaToken value. 

### Consistency Consideration while Paginating through Delta Query Responses

Service providers MUST NOT prevent resources from being updated (locking resources) while implementing delta query. New items can be added or existing items can be removed or updated while paginating through the response of the delta queries. The result set will contain eventually consistent data, however some implementations may choose to enforce strongly consistent data. The delta query MUST guarantee that the records modified (created, updated, or deleted) after any query that generates a delta token are returned when that same delta token is provided back by the client.

### Resource Representation in the Delta Query Response
Newly created resources are represented in the delta query response using their standard representation. Updated resources are represented using their standard representation, and their current state is returned.
Deleted instances MUST return common attribute id and complex attribute meta with sub attributes resourceType and meta.isDeleted attribute value with value True.

#### Minimal Representation for a Deleted Resource

Following is a non-normative example of the minimal required SCIM representation in JSON format for a deleted resource.

~~~
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id": "2819c223-7f76-453a-919d-413861904646",
  "meta": {
    "resourceType": "User",
    "isDeleted": true
  }
}
~~~
#### Full Representation for a Deleted Resource

Following is a non-normative example of the fully populated SCIM representation in JSON format for a deleted resource.

~~~
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id": "2819c223-7f76-453a-919d-413861904646",
  "meta": {
    "resourceType": "User",
    "isDeleted": true,
    "created":"2011-08-01T10:29:49.793Z",
    "lastModified":"2011-08-01T18:29:49.793Z",
    "location": "https://example.com/v2/Users/2819c223-7f76-453a-919d-413861904646",
    "version":"W\/\"f250dd84f0671c3\""
  }
}
~~~
#### Example Delta Query Response
Below example depicts a response with newly created, updated and deleted resources returned by the SCIM server in response to the Delta Query request for User resource.

~~~
GET Users?deltaQuery&count=50&deltaToken=VTHKLOUTREO
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD


HTTP/1.1 200 OK
Content-Type: application/scim+json
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
    "totalResults":3,
    "Resources":[
    {
        "id":"2819c223-7f76-453a-919d-413861904646",
        "externalId":"bjensen",
        "meta":{
        "resourceType":"User",
        "created":"2011-08-01T18:29:49.793Z",
        "lastModified":"2011-08-01T18:29:49.793Z",
        "location":
        "https://example.com/v2/Users/2819c223-7f76-453a-919d-413861904646",
        "version":"W\/\"f250dd84f0671c3\""
        },
        "name":{
        "formatted":"Ms. Barbara J Jensen III",
        "familyName":"Jensen",
        "givenName":"Barbara"
        },
        "userName":"bjensen",
        "phoneNumbers":[
        {
        "value":"555-555-8377",
        "type":"work"
        }
        ],
        "emails":[
        {
        "value":"bjensen@example.com",
        "type":"work"
        }
        ]
    },
    {
       "id":"c75ad752-64ae-4823-840d-ffa80929976c",
       "externalId":"jsmith",
       "meta":{
       "isDeleted":"true",
        "resourceType":"User",
        "created":"2011-08-01T10:29:49.793Z",
        "lastModified":"2011-08-01T18:29:49.793Z",
        "location":
        "https://example.com/v2/Users/2819c223-7f76-453a-919d-413861904646",
        "version":"W\/\"f250dd84f0671c3\""
        }
    },
    {
       "id":"c75ad752-64ae-4823-840d-ffa80929976c",
        "externalId":"xxxxxx",
       "meta":{
        "isDeleted":"true",
        "resourceType":"User",
        "created":"2011-08-01T10:29:49.793Z",
        "lastModified":"2011-08-01T18:29:49.793Z",
        "location":
        "https://example.com/v2/Users/2819c223-7f76-453a-919d-413861904646",
        "version":"W\/\"f250dd84f0671c3\""
        }
    }
    ]
}
~~~

## Delta Query Errors
If a Service Provider encounters an invalid delta query parameters (invalid delta token value, deltaQuery value, etc), or other error condition, the Service Provider SHOULD return the appropriate HTTP response status code and detailed JSON error response as defined in Section 3.12 of [RFC7644].  Most deltaQuery error conditions would generate HTTP response with status code 400.  Since many deltaQuery error conditions are not user recoverable, error messages SHOULD focus on communicating error details to the SCIM client developer.

For HTTP status code 400 (Bad Request) responses, the following detail error types are defined. These error types extend the list of error types defined in RFC 7644 Section 3.12, Table 9: SCIM Detail Error Keyword Values.

| scimType | Description | Applicability |
invalidValue | The parameter deltaToken was provided without the deltaQuery parameter. Parameter "deltaToken" must be accompanied with deltaQuery parameter. OR Invalid value for "deltaToken" parameter. Value for "deltaToken" parameter must be the same as provided by the SCIM service provider in nextDeltaToken OR Invalid value for deltaQuery parameter. Value of "deltaQuery" parameter must be either true or false| GET (Section Delta Query Parameters of ([Delta Query Draft])), POST (Section Delta Query Using HTTP POST of ([Delta Query Draft]))|
expiredDeltaToken | Delta Token has expired. Do not wait longer than deltaTokenExpiry (7 days) to request subsequent delta query requests.| GET (Section 3.4.2 of [RFC7644]), POST (Section Delta Query Using HTTP POST of ([Delta Query]))|
{: title="Delta Query Errors"}

## Filtering
If filtering is implemented as described in Section 3.4.2.2 of [RFC7644], then delta query results should be filtered. 

When the client makes a subsequent deltaQuery request, it should query the same Service Provider endpoint with all query parameters and values being identical to the initial query issued in the previous delta scan or full scan with the exception of the deltatoken value which should be set to a nextDeltaToken value that was returned by Service Provider in a previous delta query response.


## Sorting
If sorting is implemented as described Section 3.4.2.3 of [RFC7644], then deltaQuery results SHOULD be sorted.


# Delta Query using HTTP POST
Section 3.4.2.4 of [RFC7644] defines how clients MAY execute the HTTP POST method combined with the "/.search" path extension to issue execute queries without passing parameters on the URL. When using "/.search", the client would pass the parameters defined in Section 2

~~~
POST /User/.search
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD
     
{
 "schemas": [
 "urn:ietf:params:scim:api:messages:2.0:SearchRequest"],
 "attributes": ["displayName", "userName"],
 "filter":
 "displayName sw \"smith\"",
 "deltaQuery": "true",
 "deltaToken": "VTHKLOUTREO"
}
~~~

Which would return a result containing a "nextDeltaToken" value which may
be used by the client in a subsequent delta scan call to return the next set of modified resources.
~~~
HTTP/1.1 200 OK
Content-Type: application/scim+json
     
{
 "totalResults":10,
 "itemsPerPage":10,
 "nextdeltatoken":"OPUTREWSFDE",
 "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
 "Resources":[{
  ...
}]
}
~~~

# Service Provider Configuration

The /ServiceProviderConfig resource defined in Section 4 of [RFC7644] facilitates discovery of SCIM service provider features. A SCIM Service provider implementing delta query SHOULD include the following additional attribute in JSON document returned by the /ServiceProviderConfig endpoint:

   	  deltaQuery
   	  : A complex type that indicates delta query configuration options.  OPTIONAL.
      supported  
      : A Boolean value specifying support of delta query.  REQUIRED.
      deltaTokenTimeOut 
      : Non-negative integer specifying the maximum number days that a deltaToken is valid between delta Scan requests.  Clients waiting too long between subsequent delta scan requests may receive an invalid delta token error response.  OPTIONAL.
      
If the SCIM client issues a delta query to a SCIM service provider that does not support or implement delta query feature then SCIM service provider will respond with HTTP Status Code 501, Unsupported Feature - Delta Query. Server does not support delta query feature.

Before using delta query, a SCIM client MAY fetch the Service Provider Configuration document from the SCIM service provider and verify that delta query is supported.

For example:

~~~
GET /ServiceProviderConfig
Host: example.com
Accept: application/scim+json

~~~
A service provider supporting both delta query token would return a document similar to the following (full ServiceProviderConfig schema defined in Section 5 of [RFC7643] has been omitted for brevity):

~~~
HTTP/1.1 200 OK
Content-Type: application/scim+json
	
{
 "schemas": [
 "urn:ietf:params:scim:schemas:core:2.0:ServiceProviderConfig"],

 	...

 "deltaQuery": {
  	"supported": true
	},

   ...

}
~~~

Service Provider implementors SHOULD ensure that misuse of delta query by a SCIM client does not deplete Service Provider resources or prevent valid requests from other clients being handled. Defenses for a SCIM Service Provider are similar those used to protect other Web API services -- including the use of a "Web API gateway" layer, to provide authentication, rate limiting, IP allow/block lists, logging and monitoring, response caching, etc.
   
# ADD NORMATIVE REFERENCES

# Acknowledgements

The authors would like to thank Mike Kiser (Sailpoint) for his contributions to early design discussions for this draft.

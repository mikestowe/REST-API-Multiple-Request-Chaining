#REST API Multiple-Request Chaining
**Note - this is a brainstorming document and should not be considered stable or production ready**

One of the challenges in using RESTful APIs driven by Hypermedia, as well as pulling in numerous and extensive microservices is the requirement to at times make several API calls in order to accomplish the task at hand.  Today, that requires numerous HTTP calls as well, which depending on latency, can greatly slow script execution.

REST API Multiple-Request Chaining is a technique, borrowed and heavily modified from Owen Rubel's (@BeApi_io) concept of IO State driven APIs, that groups numerous RESTful API calls together in a single HTTP request.

For example, as shown below, instead of having to do GET calls to `/users/5` and then `/messages/?userId=5` you could instead send a chain to a resource such as `/multirequestchain` that would handle the multiple calls for you.

REST API Multiple-Request Chaining is setup to allow for conditional calls, as well as provides back only the data you need (instead of all the data that would be returned from each call).  This can also be used as a technique for data collection similar to that of GraphQL where instead of embedding objects as models, you're able to make numerous calls at once and get back only the data you request.

##A Simple REST API Multiple-Request Chain
Each chain is made of 5 components, all required:


Component | Description | Example 
---------- | ----------- | ---------
doOn | DoOn provides the condition on which the call should be performed.  Options include "always", a logical if statement `($body.user.firstName == 'Jim')` or an HTTP status code (with * being a wildcard). | 4**
href | The complete path (including querystring) for the call being performed | /users/?lastName=Smith
method | The HTTP method you wish to use to perform the call (such as GET, POST, etc) | get
data | A string or JSON Object of the data you wish to send via the call (typically with POST, PUT, PATCH, DELETE) | { "firstName" : "Jim", "lastName" : "Smith" }
return | The data (as an array) you wish to have returned.  If you wish for all data to be returned, use boolean "true" or if you wish for no data to be returned, use boolean "false." | ["firstName", "email", "_links"]

A simple request where you want to retrieve a user's messages, but first need to make a call to the `/users` resource to obtain the information may look like this:

```
[
  {
    "doOn": "always",
    "href": "/users/5",
    "method": "get",
    "data": {},
    "return": [
        "email", "_links"
    ]
  },
  {
    "doOn": 200,
    "href" : "$body._links.messages",
    "method": "get",
    "data": {
        "emailAddress": "$body.email"
    },
    "return": true,
  }
]
```

##Multiple Conditional Calls
REST API Multiple-Request Chaining works in chronological order, with each call being considered the next highest priority, or the next natural step in the process.

The `doOn` specifies whether or not the call should be executed based on the previous call's response, or whether it should be skipped for another call that contains a `doOn` that matches the previous HTTP status code, applies a logical IF statement that returns true, or is specified as "always."  In the event that no suitable match is found in the layer of the chain it is currently operating in, the chain will consider this an error and exit - returning back all the data up and including the last call that was attempted.

For example, in our previous chain - IF the `/users/5` call returned a 404, the chain would have exited, providing you the details of that call, but **NOT** attempting the next call `$body._links.messages` as it required a status code of 200.

You can also layer multiple conditional calls by placing them in arrays, creating a new layer in the process.  However, once the chain reaches the end of its child layers, it will exit - **NOT** iterating through the remaining parent layers.

```
[
  {
    "doOn": "always",
    "href": "/users/5",
    "method": "get",
    "data": {},
    "return": [
        "email", "_links"
    ]
  },
  [
    {
      "doOn": 200,
      "href" : "$body._links.messages",
      "method": "get",
      "data": {
          "emailAddress": "$body.email"
      },
      "return": true,
    },
    {
      "doOn": "4*|5*",
      "href" : "/users",
      "method": "post",
      "data": {
        "firstName": "Jim",
        "lastName": "Smith",
        "emailAddress": "jim.smith@domain.ext"
      },
      "return": true,
    },
    [
      {
        "doOn": "201",
        "href": "$headers.link",
        "method": "get",
        "data": {},
        "return": [
            "email", "_links"
        ]
      },
      {
        "doOn": 200,
        "href" : "$body._links.sendMessage",
        "method": "post",
        "data": {
          "to": "$body.email",
          "subject": "Welcome $body.firstName",
          "body": "Hello and welcome to our site!"
        },
        "return": true,
      }
    ]
  ]
]

```

##Complex IF Statements
The `doOn` property accepts an HTTP Status Code, "always" as a string, or a conditional logical IF statements:

#####Simple Equal/ Not Equal
Logic | Example | Meaning
------ | --------- | -------
== | $body.firstName == 'Jim' | firstName in Body response is **equal** to Jim
!= | $body.firstName != 'Jim' | firstName in Body response is **not equal** to Jim
\>= | $body.age >= 10 | age in Body response is **greater than or equal** to 10
<= | $body.age <= 10 | age in Body response is **less than or equal** to 10

#####AND OR STATEMENTS
Logic | Example | Meaning
------ | --------- | -------
&& | $body.firstName != 'Jim' && $body.age >= 10 | First condition **AND** second condition must be matched
\|\| | $body.firstName != 'Jim' \|\| $body.age >= 10 | Match **EITHER** condition

#####REGULAR EXPRESSIONS
Logic | Example | Meaning
------ | --------- | -------
regex() | regex('/[a-z]/i', $body.firstName) | **Match** a regular expression
!regex() | regex('/[a-z]/i', $body.firstName) | Does **not match** regular expression

###Complex Example:
```
[
  {
    "doOn": "always",
    "href": "/users/5",
    "method": "get",
    "data": {},
    "return": [
        "firstName", "lastName", "email", "_links"
    ]
  },
  {
    "doOn": "($body.firstName == "Jim" && $body.lastName == "Smith") || regex('/Jim/i', $body.email)",
    "href" : "$body._links.messages",
    "method": "get",
    "data": {
        "emailAddress": "$body.email"
    },
    "return": true,
  }
]
```


##Responses
Because conditional chaining and errors are possible when using REST API Multiple-Request Chaining, the response object needs to return three primary properties:

Property | Type | Definition
-------- | ---- | ----------
callsReqested | integer | The number of conditionally applicable calls requested in the chain - this will not necessarily be the total number of calls in the request
callsCompleted | integer | The number of calls that were completed successfully.  This helps indicate an error IF the chain had multiple calls, but a condition could not be matched and therefore caused the chain to exit
responses | array | the list of applicable responses to the API chain calls that were performed, including the last call that could either be executed either because the chain was completed or exited due to being unable to meet a necessary condition.

Within the responses array, each call object needs to include:

Property | Definition | Example
-------- | ---------- | -------
href | the full path of the call that was attempted | /users?firstName=Jim
method | the method that was used to make the call | get
status | the HTTP status code the call returned | 200
response | a response object containing the headers (as an object) and the body (as an object or string depending on content-type) | "headers" : { "content-type" : "application/json" }, "body" : { "user" : { "firstName" : "Jim" } }


```
{
  "callsRequested" : 2,
  "callsCompleted" : 2,
  "responses" : [
    {
      "href" : "/users/5",
      "method" : "get",
      "status" : 200,
      "response" : {
        "headers" : {},
        "body" : {
          "email": "user@user.ext",
          "_links": {}
        }
      }
    },

    {
      "href" : "/messages/?userId=5",
      "method" : "get",
      "status" : 200,
      "response" : {
        "headers" : {},
        "body" : {}
      }
    }
  ]
}
```

##FAQs

###How do you send headers with REST API Multiple-Request Chaining?
Headers are sent as they always have been, and will automatically be applied to each of the calls requested in the chain.  At this time, REST API Multiple-Request Chaining does not support the ability to send specific headers for each call independently - however, if needed, this may certainly be added in the future.

###How does this vary from IO State Driven APIs
REST API Multiple-Request Chaining is designed specifically for RESTful APIs utilizing hypermedia over a static or cached state file.  This allows for the available paths to be truly dynamic based on the individual calls within the chain, and also prevents the need to make multiple requests to obtain the IO State or available OPTIONS.  REST API Multiple-Request Chaining is also designed to expand beyond just the use of a primary key, letting you pull in information from the headers and body of the previous call when performing an action on the next chain.  Each link/ call is also conditional, meaning you can specify when that link should be called, and receive the appropriate error response upon a link/ call failure.  For more you can see Owen's IO State implemention [here](https://github.com/orubel/grails-api-toolkit-docs/wiki/API-Chaining).

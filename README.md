#API Chaining
**Note - this is a brainstorming document and should not be considered stable or production ready**

One of the challenges in using RESTful APIs driven by Hypermedia, as well as pulling in numerous and extensive microservices is the requirement to at times make several API calls in order to accomplish the task at hand.  Today, that requires numerous HTTP calls as well, which depending on latency, can greatly slow script execution.

API Chaining is a technique, borrowed from an idea presented by Owen Ruble (@...) during one of his talks, that groups numerous RESTful API calls together in a single HTTP request.

For example, as shown below, instead of having to do GET calls to `/users/5` and then `/messages/?userId=5` you could instead send a chain to a resource such as `/multirequestchain` that would handle the multiple calls for you.

API Chaining is setup to allow for conditional calls, as well as provides back only the data you need (instead of all the data that would be returned from each call).  This can also be used as a technique for data collection similar to that of GraphQL where instead of embedding objects as models, you're able to make numerous calls at once and get back only the data you request.

##A Simple API Chain
Each API Chain is made of 5 components, all required:


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
API Chaining works in chronological order, with each call being considered the next highest priority, or the next natural step in the process.

The `doOn` specifies whether or not the call should be executed based on the previous call's response, or whether it should be skipped for another call that contains a `doOn` that matches the previous HTTP status code, applies a logical IF statement that returns true, or is specified as "always."  In the event that no suitable match is found in the layer of the chain it is currently operating in, the chain will consider this an error and exit - returning back all the data up and including the last call that was attempted.

For example, in our previous chain - IF the `/users/5` call returned a 404, the chain would have exited, providing you the details of that call, but **NOT** attempting the next call `$body._links.messages` as it required a status code of 200.

You can also layer multiple conditional calls by placing them in arrays, creating a new layer in the process.  However, once the API Chain reaches the end of its child layers, it will exit - **NOT** iterating through the remaining parent layers.

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

##Responses

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

###How do you send headers with API Chaining?
Headers are sent as they always have been, and will automatically be applied to each of the calls requested in the chain.  At this time, API Chaining does not support the ability to send specific headers for each call independently - however, if needed, this may certainly be added in the future.

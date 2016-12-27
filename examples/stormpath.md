#Stormpath Example
**Note this is just an example, Stormpath does not support Multiple-Request Chaining.**

In this scenario, we want to retrieve a list of accounts where the email is jim@jim.ext.  First we must get our current tenant, then our applications (we'll assume we only have one), and then search the accounts for that application.  You can see the full list of steps [here](http://docs.stormpath.com/rest/quickstart/).

We can do this all with a single call, and only have the account information sent back to us (reducing the bandwidth used).

```
[
  {
    "doOn": "always",
    "href": "/v1/tenants/current",
    "method": "get",
    "data": {},
    "return": false
  },
  {
    "doOn": 302,
    "href": "${headers.location}/applications",
    "method": "get",
    "data": {},
    "return": false
  },
  {
    "doOn": 200,
    "href": "${body.items[0].accounts.href}?email=jim@jim.ext",
    "method": "get",
    "data": {},
    "return": true
  }
]
```

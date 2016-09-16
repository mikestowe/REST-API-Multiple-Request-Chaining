#Benchmarks
The following are first-look benchmarks for three different series of API calls.  They were tested using a basic API-Chaining parser written on PHP.  To ensure call fairness, all calls go through the same proxy, hitting the same server to ensure similar server latencies. 


### Test 1
Three calls to an API proxy, which in turn then calls a third party service hitting a mock API.  This was designed to test the hypermedia links (CPHL format).  The Control makes these calls in a traditional manner, making the call, processing the logic, and then moving on to the next call.  The Chained call makes one call where the server handles the logic, and then returns back the requests.

Note - this was run from my local host, meaning there could be slight performance differences from my localhost PHP server (4gb memory) vs the proxy PHP server (512mb memory).

[benchmark one]: test_1.jpg


### Test 2
Identical to test one except that the second call returns back a failure, forcing the control and the chain to stop after completing the second of the series of requests.  The Control makes these calls in a traditional manner, making the call, processing the logic, and then moving on to the next call, finally stopping after finding the second call did not return the expected status code.  The Chained call makes one call where the server handles the logic, and then returns back the requests - returning back that only 2 of the 3 `callsRequested` could be completed.

Note - this was run from my local host, meaning there could be slight performance differences from my localhost PHP server (4gb memory) vs the proxy PHP server (512mb memory).

[benchmark one]: test_2.jpg


### Test 3
Unlike tests 1 and 2 which involve a third party API, test 3 focused on calls where the server handles all the logic itself.  In this case, calls to the server simply return a standard JSON document.  The control calls the same resource three times, while the chain sends a single request with a chain that performs an identical function (calling the request three times), but asks the server to only return back the window.image.src (a subnested property).

Note - this was run from my local host, meaning there could be slight performance differences from my localhost PHP server (4gb memory) vs the proxy PHP server (512mb memory).

[benchmark one]: test_3.jpg


### Test 4
Test 4 was similar to test 3 in that it called a resource that was strictly handled by the server.  However, this time we only made one request, one to the resource itself using our control, and then one to the chain requesting that resource.  The purpose of this test was to see the hit that would be taken on an individual call, where again we filtered the response to only return the window.image.src field.

As expected, the control was faster than the chained request, but surprisingly only by .0014 seconds - demonstrating that at least with the basic parser the actual delay caused by the chain logic was insignificant overall - and easily accounted for when making multiple calls (as demonstrated by the tests above).

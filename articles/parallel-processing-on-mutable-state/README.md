# Parallel processing on mutable state

## Why is important

Despite the fact that Moore law still works, processor clock speed is not changing over last 15 years. When first mass multicore processors released prominent people like Herb Sutter [proclaimed](http://www.gotw.ca/publications/concurrency-ddj.htm) a fundamental turn toward concurrency in software development.

<img src="./cpu-perfomance.png" />

Even though single thread performance did improved 10x (literally) since then, every modern software development team tries to tackle throughput and latency issues primarily by applying parallel processing.

This article is about common scenarios in service oriented software when parallel processing is does not brings performance gains to application.

## The problems

Consider this naive architecture of Uber-like ( [ride-hailing](https://en.wikipedia.org/wiki/Peer-to-peer_ridesharing) ) application.

<img src="./booking-app.png" />

* Booking Service is simple [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) API which allows to create and change state of booking. Booking states are `created`, `accepted`, `cancelled` and `completed`
* Passenger app allows to create and cancel booking. 
* Driver app allows driver to accept and complete bookings.

Our goal is to implement following business requirements:
* bookings can be `cancelled` after it was `accepted`
* bookings cannot be `accepted` after it was `cancelled`

### Naive solution

Driver app checks if booking state is `created` and then sends accepts it: 
* GET request to receive booking state
* if booking state `created` send `PUT` request to set state to `accepted`

Immediate problem is: what happens if Passenger App sent cancel request between `GET` and `PUT` in Driver app. If Booking Service do no special checks, last `PUT` request will override `cancelled` state with `accepted` state. Which breaks our business requirement.

The problem which occurs can be classified as as `read-modify-write` race condition. For many engineers, due to [Birthday Paradox](https://en.wikipedia.org/wiki/Birthday_problem) it can none intuitive how often this race condition will actually happen.

### Improving naive solution

The obvious way to fix the app would be validating previous state inside Booking Service. Let's consider following stereotypical architecture for booking service:

<img src="./booking-service.png" />

* for purposes of scalability and availability booking service running multiple instances.
* all instances share single database

Now let's move our booking state validation from Driver App to booking service. When `PUT` request with state updates processed:
* booking service query database for current state
* if booking state is still `created`, update it

## The solution

## Links

https://www.karlrupp.net/2018/02/42-years-of-microprocessor-trend-data/

http://henrikeichenhardt.blogspot.com/2013/06/why-shared-mutable-state-is-root-of-all.html

https://en.wikipedia.org/wiki/Amdahl%27s_law
# Open Integration API

## TL;DR

RTB (Real Time Bidding) the system was invented to make possible to interact between each other many different advertisement networks in real time. Generally, networks use OpenRTB protocol as a standard to interact services between each other, it uses all big advertisement networks such as Google, Criteo, Yandex and etc. This is a very flexible protocol which continues to evolve insofar as a community continues to make it more flexible and more accurate to provide more personalized advertising to the users.

One of the most important things of the API integration for two ways integration, it’s the information which owned by the second partner of this integration. Mainly the common information like count of timeouts, average time, count of errors and etc. is enough to fix a major piece of the problems and watching for integration status in real time.

The purpose of this document is to form a common vision of the problem and propose its solution. This is very important for technical teams to make such integration more predictable and informative. We have huge experience in RTB integrations and for us sometimes is very complicated to define that we have any problem in integration if we don’t know metric information from the partner platform.

Also, the second aim for us it’s reduce amount of timeouts from the service. It can be reachable if we know several values. The most important is to know when the request was send, for we could to be sure that we can to response on this request in time. This raises several other time synchronization issues.

### About OpenRTB

Real-time Bidding (RTB) is a way of transacting media that allows an individual ad impression to be put up for bid in real-time. This is done through a programmatic on-the-spot auction, which is similar to how financial markets operate. RTB allows for Addressable Advertising; the ability to serve ads to consumers directly based on their demographic, psychographic, or behavioral attributes.

The Real-Time Bidding (RTB) Project, formerly known as the OpenRTB Consortium, assembled technology leaders from both the Supply and Demand sides in November 2010 to develop a new API specification for companies interested in an open protocol for the automated trading of digital media across a broader range of platforms, devices, and advertising solutions. At the time Programmatic had only accounted for 4% of the display advertising market. By 2017, RTB is expected to account for 29% of the digital mix (source: eMarketer March 2013). This group continues to create open industry standards that ensure all parties, buy and sell side alike, can transact RTB at scale and build future industry innovation.

[OpenRTB v3.0 FINAL](https://github.com/InteractiveAdvertisingBureau/openrtb/blob/master/OpenRTB%20v3.0%20FINAL.md)
[OpenRTB (Real-Time Bidding)](https://www.iab.com/guidelines/real-time-bidding-rtb-project/)

## What is it mean “The Integration API”?

The “Integration API” provides extra information to the partner network, and it means that every two sides of integration by RTB will have information about integration which at the moment available only for the network which conducting the auction.

This information such as the number of success requests, timeouts, nobid count, and errors reasons if highly important to automatic optimization of integrations. Other words this is the metrics provided to the network to help to understand and detect problems without interaction between developers. 

So the solution of the problem contains from the two steps:
1. Providing request generation timestamp in the HTTP header (Included into OpenRTB v3)
2. Provide the standardized API with DSP metrics

### HTTP headers

> I think the best way is to add to OpenRTB the special field with a time of request generation. Because of in the protocol can be used different types of transport like HTTP and TCP (RPC) integration with the protobuf encoder.

> For now in the OpenRTB we have **tmax** only - Maximum time in milliseconds the exchange allows for bids to be received including Internet latency to avoid timeout. This value supersedes any *a priori* guidance from the exchange.

> However, OpenRTB version 3 already includes originated time field **Request.Source.ts**.
> https://github.com/InteractiveAdvertisingBureau/openrtb/blob/master/OpenRTB%20v3.0%20FINAL.md#object_source

> @link https://en.wikipedia.org/wiki/List_of_HTTP_header_fields

In HTTP protocol exists special time header marks with the time of the message was originated and lifetime.

- Date - The date and time that the message was originated (in “HTTP-date” format as defined by ). **Date: Tue, 15 Nov 1994 08:12:31 GMT**.[RFC 7231 Date/Time Formats](https://tools.ietf.org/html/rfc7231#section-7.1.1.1) 
- Expires - Gives the date/time after which the response is considered stale (in “HTTP-date” format as defined by ). **Expires: Thu, 01 Dec 1994 16:00:00 GMT`** [RFC 7231](https://tools.ietf.org/html/rfc7231) 

However, all this time marks restricted in seconds what is not acceptable in RTB integration because of time counting, and maximal execute time usually in Milliseconds as well.
That’s why we have to create new custom HTTP headers with the message originate time in Milliseconds.

> **X-Request-Ts: 1549107735603**<br />
> **X-Response-Accepted-Ts: 1549107737203**<br />
> **X-Response-Ts: 1549107737215**

As a plus, it good to know what server right now sends a request or responding to localize the problems.

> **X-Request-Node: unical-node-ID**<br />
> **X-Response-Node: unical-node-ID**

### Time Synchronisation

Unfortunately, we can’t be completely sure that all servers of our partners have the same time, so we have to be able to detect such situations and calculate time deltas to make additional analytics of problem diagnostic. To calculate of time difference we can use the approach of SNTP protocol [RFC 2030](https://tools.ietf.org/html/rfc2030)

```markdown
| Timestamp Name        |  ID  |  When Generated                 | Header                   |
|---------------------—-|------|---------------------------------|--------------------------|
| Originate Timestamp   |  T1  | time request sent by client     | X-Request-Ts             |
| Receive Timestamp     |  T2  | time request received by server | X-Response-Accepted-Ts   |
| Transmit Timestamp    |  T3  | time reply sent by server       | X-Response-Ts            |
| Destination Timestamp |  T4  | time reply received by client.  |                          |
```

The roundtrip delay *d* and local clock offset *t* are defined as.

```
d = (T4 - T1) - (T2 - T3)
t = ((T2 - T1) + (T3 - T4)) / 2
```

## Detect latency by the time of request generation

This is the most simple way of detection of latency. Every **HTTP request** can contain in the header special UnixTimestamp mark in **milliseconds** with the time of request generation.

Based on this information we can immediately decide is it makes sense to response for this request or no.

For example, we have OpenRTB request from some network with timestamp **X-Request-Ts: 1548660964675633000** and we received the request after **40ms**. In the OpenRTB request, we see the response limit is **tmax=100ms** and according to this information, we can make the conclusion that we can respond in time because after request generation left **60ms** what is greater than **40ms**. And in case of that after request generation, **60ms** passed it means that this is no makes sense to response on this request, because of more than **50%** of the time passed.

```
Δt1 - Delta of the time of request sending
Δt2 – Delta of the time to create request response

Admissibility = tmax - Δt1 - Δt2 > Δt1
```

```go
func bid(request) {
  dt1 := time.Now().Sub(request.GeneratedTime()) / time.Millisecond
  dt2 := averageInternalRequestDelayMs
  if dt1 + dt2 > request.TMax()/2 {
    metrics.Increment("request.latency.threshold")
    return noTimeResponse()
  }
}
```

> The main problem is that all servers have to be synchronized with the timeserver. And if some servers will have a problem with it we can try to adapt to their time by making some probe responses. 

## Integration info page

The page with information about DSP metrics of latency, errors, timeouts, etc. This is how the external networks can analysis of integration effectivity and build metrics based on it.

```sh
curl -u name:passwd http://tsyndicate.com/v2/ssp/{{id}}/integration
```

All numbers (timeouts, nobid, success, rate) per 1s.

```json
{
  "id": "xd1000",               // ID of the integration link
  "node-id": "id1221",          // ID of the server
  "protocol": "openrtb",        // Optional information about protocol of integration
  "timestamp": 1549107735603,   // Current timestamp of response generation
  "min_latency": 15,            // Minimal request delay in Milliseconds
  "max_latency": 250,           // Maximal request delay in Milliseconds
  "avg_latency": 24,            // Avegrage request delay is 24ms
  "avg_roundtrip_latency": 37,
  "avg_time_offset": -21,       // If this parameter high enough by absolute value, then could be some problem with server time
  "qps_limit": 100,             // Restriction of requests per second
  "qps": 90,                    // Current QPS count
  "total_count": 3370,
  "success_count": 1000,        // Count of success responses per a second
  "success_rate": 0.28,
  "timeouts_count": 1000,       // Count of timeout responses per a second
  "timeouts_rate": 0.28,
  "no_bids_count": 1000,        // Count of nobid responses per a second
  "no_bids_rate": 0.28,
  "errors_count": 370,          // Count of all error responses per a second
  "errors_rate": 0.06,
  "errors": [
    {
      "type": "http",
      "code": "500",
      "count": 170,
      "rate": 0.008
    },
    {
      "type": "http",
      "code": "503",
      "count": 100,
      "rate": 0.005
    },
    {
      "type": "service",
      "code": "lowbid",
      "count": 100,
      "rate": 0.005
    }
  ]
}
```

### The errors segregation

All errors have to be segregated by type and code of error.

**Types**

- HTTP - contains all inappropriate HTTP responses. (Valid http response codes are 200, 204, depends on protocol)
- service - contains all network internal responses like
  * lowbid
  * invalid - parsing error

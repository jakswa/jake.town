---
layout: post
title: "Filling MARTA Realtime Gaps"
slug: marta-realtime-gaps
description: "Going from realtime to realtime-ish with MARTA's train API."
---

*Going from realtime to realtime-ish.*

MARTA's realtime API has been available for a little over a year now. I loved that they did this. I used it to build a side project over at [marta.io](https://marta.io). Anyone can fill out a form, wait for an API key, and then start using that to hit up MARTA for realtime train data:

```json
[{
  "DESTINATION": "Airport",
  "DIRECTION": "S",
  "EVENT_TIME": "3/12/2014 5:40:28 PM",
  "LINE": "GOLD",
  "NEXT_ARR": "05:40:37 PM",
  "STATION": "BROOKHAVEN STATION",
  "TRAIN_ID": "308506",
  "WAITING_SECONDS": "-37",
  "WAITING_TIME": "Boarding"
},
...
]
```

One endpoint gives you the wait times for every train and every station. Each future stop for a train—all the way to its destination—will have an entry in the response. The above entry shows a southbound train boarding at Brookhaven station. Using its `train_id`, you can find the future stops for this train:

![Train stops visualization](/assets/images/realtime_gaps_1.webp)

The API currently can have as many entries as there are stations ahead of trains (up to some maximum). That's a little hard to picture. If there are two trains with 18 stops each, then there will be 36 entries. If there is only one train, and it has three stops left, then there will be only three entries. At rush hour, this response grows to an array with something like 200 entries.

So, is the API endpoint for stations, or for trains? It is both. It's like MARTA joined trains and stations, and gave us the result. Their join is conditional, though: An entry represents an incoming train. So, where's the downside?

<img src="/assets/images/realtime_gaps_2.webp" alt="MARTA line diagram" class="float-left">

Take the image on the left. Imagine College Park station. Where are its trains? Three are in the API, headed south towards the airport. But are there any headed north, between College Park and airport? Almost never. And when a train does appear, you better hope you're close, because you'll only have a minute or two before it hits.

A train sitting at airport station doesn't appear in the API until it starts moving. So, how to fix this? You're waiting at College Park station. You look at the sign and it shows estimates for two northbound trains, trains that aren't on the tracks yet. How? Turns out, they use the schedules. For College Park, it'd be the numbers you see on itsmarta.com for College Park station.

Is that good data to use? I don't know. Is schedule data good to use in a "realtime" context? I never noticed the inconsistency until I started using the realtime API, and I live near a terminating station like the airport. Based on that sound logic, we'll say it's good. How to get it ourselves? Thanks to the kind folks at Google, MARTA publishes its train and bus schedules in 50MB or so worth of CSVs that conform to the General Transit Feed Specification. Throw out all the bus-related data, and organize things into a few JSON files, and it reduces down to 300kB uglified. Now we can show everyone the same data that is on the MARTA signs:

![MARTA.io with schedule data](/assets/images/realtime_gaps_3.webp)

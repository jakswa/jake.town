---
layout: post
title: "300 Days in: Engineering at CallRail"
slug: 300-days-at-callrail
description: "Where did those 300 days go, and what's been keeping me busy?"
image: /assets/images/300_days_callrail_header.webp
---

![](/assets/images/300_days_callrail_header.webp){: .hero-image}

Watch out for busyness. It sneaks up on you. One minute, you're starting a new job and diving into some internally facing UI code. Next thing you know, you look in the developer chat room and other changes you made are being deployed—this time, for the call routing logic. It's the code that drives calls for thousands of customers. The difference in scope is like day and night, and it seems to have crept up on me overnight.

Staying busy, you'll miss other things. The other day I looked up from my monitor and realized we had tripled the number of employees since I joined.

<figure>
  <img src="/assets/images/300_days_callrail_1a.webp" alt="Office photos">
  <figcaption>The office is looking much more crowded these days.</figcaption>
</figure>

I'm staring at code while the names fly by, blurring together the people and events. Just 300 days in, I'm even forgetting my own features. Where did those 300 days go, and what's been keeping me busy?

Let's get some of my early highlights down, before I forget:

## Day 1: Internal Search

CallRail is a Rails shop. I hadn't worked with Rails seriously for a year. This was a good first task to start re-familiarizing myself. Our support people would get calls, and need to quickly discover details about a phone number or name. I created a search box for them to type those into. The search results might be numbers, people, or other account-related objects. It exposed me to a bunch of our data and our code, and seemed to help support a ton.

## Day 7: First Time Callers

Our dashboard is the go-to place for getting an overview of your calls. There's a graph showing your calls by day, and customers were interested in filtering the view by only first-time callers. We use Angular on the client-side, and this was my first dive into that JavaScript.

![First time callers feature](/assets/images/300_days_callrail_2.gif)

## Day 25: Incoming Business Details

<img src="/assets/images/300_days_callrail_3.webp" alt="Business details feature" class="float-left">

You're a company receiving phone calls. Sometimes those calls come from other companies. We wanted a quick way for CallRail users to recognize these types of calls, and look at data about that business. So, a "Business" tab was created in the real-time section of our app, which we call Copilot.

## Day 51: Outbound Recording

You want to call a customer back but you don't want to use your personal phone. You want the customer to see your company's number as the caller ID because that's what they originally called. We have an outbound call UI that will help you make that happen. I added a check box that lets you record that outbound call if your account admin has enabled it.

![Outbound recording feature](/assets/images/300_days_callrail_4.gif)

## Day ??: Mario Kart

After day 51, there was a gap in highlight-worthy features. Was it because I switched to more backend-heavy work, or because Mario Kart arrived in the office around this time? We'll never know, I guess.

> "post-free lunch Wednesdays Mario Kart competition!"
>
> — [View on Instagram](https://www.instagram.com/p/5uhVijOXhO)

## Day 162: Spam Detection

It turns out phone companies deal with a ton of spam. Customers like receiving relevant phone calls, not spammy robocallers. To help them out, we integrated with a prominent spam-detection service to weed out trashy calls before they're able to ring the phone:

> Your CallRail account will automatically be configured to "challenge" robocallers and telemarketers on your "Blocked Numbers" page within each company in your account. With the settings configured to "Challenge," anyone detected as calling from a potential spam number will hear a message asking them to press 1 before the call is connected. If the caller does not press 1 to connect, then the call will be disconnected.

## Day 185: FullStory Integration

FullStory is a magical service that allows you to review website visitors' experiences as if they were recorded on video. Imagine being a support person, and receiving a call from a frustrated website visitor. If you're using CallRail and FullStory, you can immediately bring up their browser session:

![FullStory integration](/assets/images/300_days_callrail_fs.gif)

## Day 197: TADHack

Turns out an annual hackathon exists specifically for developers in the telephony industry: TADHack. We thought up an idea that'd let us test some new things out and built it in a weekend. And we won a prize:

> .@CallRail takes home prize for "Best hack using @Bandwidth & @telestax" w "call center in a box" #TADHack #Raleigh
>
> — [@bandwidth](https://x.com/bandwidth/status/610250392761466880)

## Day 209: Self-balancing Scooters

We used our hackathon winnings to buy toys for the office.

> "We're much more comfortable on these things now!"
>
> — [View on Instagram](https://www.instagram.com/p/4zxnfiOXt6)

## Day 306: Upgrading Offices

Our current office is already the nicest location I've ever worked in. We're located in Midtown Atlanta at the moment, and I sit near big windows that provide tons of natural light.

<figure class="float-left">
  <img src="/assets/images/300_days_callrail_old_office.webp" alt="Current office view">
  <figcaption>The view when I look up from my busyness.</figcaption>
</figure>

It's developed a big downside, however: Lack of space. We've handily outgrown the current space, so we're about to relocate to an awesome, more spacious building in Downtown Atlanta. Pretty soon, the current office is going to join the collection of things blurring by.

<figure>
  <img src="/assets/images/300_days_callrail_new_office.webp" alt="New office view">
  <figcaption>View from the soon-to-be CallRail office.</figcaption>
</figure>

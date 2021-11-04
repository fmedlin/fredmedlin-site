---
title: "Refactoring Simplenote"
date: 2021-11-02T06:03:47-04:00
tags: ["android", "refactoring"]
draft: false
---

Simplenote is an Open Source note taking application published by Automattic. Source for the Android client is [available on github](https://github.com/Automattic/simplenote-android).


## AnalyticsTrackerNosara.java

Here's the [original source file](https://github.com/Automattic/simplenote-android/blob/develop/Simplenote/src/main/java/com/automattic/simplenote/analytics/AnalyticsTrackerNosara.java)

{{< gist 5633138ab0cb52470852f7c7ac8eacea >}}

I'll approach this as if I plan to make some feature update; meaning, refactor first before starting the update.

A couple of things stand out:
1. A cluster of private `XAnonID()` methods
2. A longish `track()` method

I'll focus on the `track()` method first.

{{< gist 69b442c90db4e928cf105b95e04154eb >}}

There are three sections in this method, each separated by some whitespace.

In the top section, the initial null client check is fine. The client is initialized in the constructor. I'm not sure why that would fail, but I don't want to change any behavior here, so it stays.

The bottom section calls the tracking client with an event name, some attributes and optional properties. I want to extract the if-else embedded there.

## TrackableUser

That leaves the middle section and I'll start there. This section assigns user and userType attributes for the analytics event. The nested if-else blurs the fact that it is just making those assignments. Making it more readable for the next developer is an important goal of refactoring, so I'll do that.

There are three conditions bound up in that middle assignment section.
- When a user has a username (from an instance variable)
- When an unnamed user has an anonymous id
- When a user is unnamed and does *not* have an anonymous id

I chose to replace the bulky if conditions with a representative interface that I named `TrackableUser`. The analytics code tracks users by name if it has one or anonymously by uuid if no name is present. It's going to take a few steps before this function is more readable, but I only want to make small deltas that don't change the existing behavior; starting with the user name.

{{< gist 91d0916d806634adbb32e4d90f326e17 >}}

I implemented  two classes of the interface that represent named and unnamed users and provided a factory function to create them.

{{< gist 08fc6603b1fe750588356b004f1be8ce >}}

{{< gist 0798de07584308c2c5f675991aa6748a >}}

Next, replace the explicit username condition check with the new interface.

{{< gist b0df74f9b46b8ec5ebfacca5a4d2afb7 >}}

It doesn't look like much yet, but we haven't changed any behavior. The `TrackableUser` interface gives us a place to push code, eventually simplifying the function.

### AnonymousId

To continue along this path, it looks like things will get jammed up with those `XAnonID()` functions very soon. So, I decide to take a quick diversion to the concept of anonymous ID. It's a concept that wants to escape from the existing implementation. One way to tell is by noticing all of the XAnonID suffixes.

- `clearAnonID()`
- `getAnonID()`
- `generateNewAnonId()`

{{< gist 5c167c0895928185f5617ea8fb33e2c7 >}}

Luckily, these functions aren't spread out too far. In a bigger file, those fluffs of functions can get disorganized quickly. Even though this is a pretty small implementation, let's give `AnonymousId` the conceptual status that seems to be forming. It's going to need to be extracted to make the `track()` function more readable.

Preferring composition to inheritance doesn't mean that classes and objects are code smells. A class is a good organizing structure for similar functions.

{{< gist 5378d0aea26ade372b2b80da358591c3 >}}

Now back to our trackable user, there are actually three conditions to distinguish, so finish those out.

{{< gist 0385c509cc7b80e935fe5d7560376c4c >}}

The track function still looks quite bulky. But I'm not worried, the magic is about to happen. A few small refactor patterns are about to collapse these multiple if-elses.

{{< gist 8c36fec938b5a4d789c40b3942625372 >}}

Adding those new instances of SharedPreferences around added some unnecessary heft to the `AnonymousId` method calls, so let's clean that up by moving the SharedPreferences in the constructor. This also removes the need for the `mContext` instance variable in the Tracker class.

{{< gist 865a6087ecd1efc82a8e8ac7f6af1585 >}}

Now, we can start pushing code into the Trackable User in order to consolidate concepts and get similar code closer together.

{{< gist 0544efc346825db5a7236eca367f2b98 >}}

This makes the `track()` method start to look more purposeful.

{{< gist 9c90bab1cb689e3ac0315454d2b5bf33 >}}

## Event tracking

Now to deal with that bottom section of the `track()` method. Extracting the event tracking and inlining the user and userType assignment really clarifies responsibilities here, IMO.

{{< gist ba5f25e68b3086df1177e571746d360b >}}

After some final refactoring, here's the updated file.

{{< gist 39e0d51f0725f463ed394f92ddad3180 >}}

## Summary and takeaways

We clarified two concepts that seemed to want recognition. Both `TrackableUser` and `AnonymousId` seem to be workable concrete implementations that could have their own set of unit tests. That's probably what I would do next before starting the (pretend) feature change that led to this refactor.

If I was new to this project, refactoring might be an excellent way to learn how the code works. Also important to remember is that you don't have to push your first refactor. If you treat it like an onboarding kata, then it's a safe way to explore the code and extract understanding.

1. Watch for var and function affixes to identify clumps of data (or functions) that can be grouped together.
2. Inside many long methods are concepts that deserve more explicit definition. Refactoring  provides clarity; making it easier for the next developer to read and grok the intended  behavior.
3. Watch for whitespace (or comments) in long methods. Often, those are seams between sections having (too) many responsibilities. These are helpful signposts indicating where to separate the large function into smaller, single-purpose ones. Many developers instinctively add those whitespace dividers because they "feel" right. Often what they are feeling is actually a code smell when a long function has too many responsibilities.

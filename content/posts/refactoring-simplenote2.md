---
title: "Refactoring Simplenote2"
date: 2021-11-07T06:25:27-05:00
tags: ["android", "refactoring"]
draft: false

---

Simplenote is an Open Source note taking application published by Automattic. Source for the Android client is [available on github](https://github.com/Automattic/simplenote-android).

## AboutFragment.java

Here's the [original source file on github.](https://github.com/Automattic/simplenote-android/blob/25c4eccdd4f153ad6a5dda7b4bb7d8de427f337b/Simplenote/src/main/java/com/automattic/simplenote/AboutFragment.java)

{{< gist 1841dad849e2cba7d5550c48af040a73 >}}

The things that stand out to me
1. The method `onCreateView()` is really long; obscuring that it mostly initializes some click handlers.
2. A big block of `View` and `TextView` assignments
3. A long list of `OnClickListener` attached to the views but disconnected from the View declaration.
4. Some click listeners with the same click responses.

## View reconciliation

The first thing to do is to get the view assignments closer associated with the click listener assignments. That should make it easier to extract methods from `onCreateView()` later.

I mostly write in Kotlin these days, so there may be a better way to do this in Java. For now though, I have created a static method to get the view assignment and the click handler closer together.

{{< gist 0f1deda5918e1061ea57a952094f9ce6 >}}

Now, the `about_blog` click handler looks like this:

{{< gist f811fe0217309edcd045b4da6f694306 >}}

Some of that common click handler behavior can be gathered into a private function.

{{< gist fcdd9a0c49fd8194fd6c656093c38144 >}}

There's a bit of YAGNI with that default version with the parameterized URL but I'm ok with that for now. Next, use the same approach for the TextView click handlers.

{{< gist 78f697e4d76479b0f6e7fb90065e86c8 >}}

{{< gist 2c2e7b4e5eb538799c8b378aa9cba9db >}}

This change gets the view assignment and click handler closer together. But it doesn't really get very far in the overall goal of shortening the `onCreateView()` function.

Here's the shortened `onCreateView()` after extracting and grouping functions.

{{< gist 2dcf19ce657900a6cf30ef6d45afa436 >}}

To me, reading this function out seems like an improved explanation of what the function is doing.

- The `onCreateView()` inflates the layout and sets up some views.
- Function `setupViews()` will set version and copyright text, handle some view clicks, handle some text view clicks and set something called a `speedListener` on a `SpinningImageButton`.

View click handling gets setup here.

{{< gist 3386bc10d21dc1f5f9c921c78544a091 >}}

Here's the final, refactored result.

{{< gist a191b6a8f1b5bc8f2b9d94f573a66761 >}}

## Summary and takeaways

These refactors only reduce the file's LOC by 10 lines. Even though there isn't a lot of functionality in this fragment, I think it was worth the effort to make it more readable. It makes it a lot less likely that the next developer will jam some unrelated code into the function named `handleViewClicks()`. I couldn't confidently say that about the original `onCreateView()` method.

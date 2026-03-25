+++
title = "Rewriting WSO Mobile in Swift"
date = "2026-03-23T22:59:16-05:00"

# description = "What happens when trying to replace a decade-old mobile app? You learn a lot about how to do things right from the start."

tags = ["wso",]
+++

At Williams College, most student technological services run through an organization called WSO[^1], for common tasks like checking the course calendar, dining hall menus, rating professors, trading textbooks, and more. Since the college doesn't provide its own app, that means our app is the standard app for the ~2,000 students who are active students. 

Around December of 2025, I began to realize that we should aspire to do better than our current trainwreck of an app. The last time that WSO's aging mobile app was rewritten before the rewrite that I am about to discuss here was all the way back in *2016*, meaning the current mobile app was made of nearly *a decade* of unmaintained trash. I can't even begin to explain how bad this actually was[^2]:

- There was functionally no security: for cross-compatibility reasons, the app was written as a port of the old web client running in a full-screen web browser, so sensitive information like passwords was effectively a cookie in that in-app browser.
- The app didn't look good, it didn't work in dark mode, and its UI was something straight out of early Android. Confusing layout of buttons, poor readability, and an interface that actively hid information that was visible on the desktop version were all common user gripes.
- It was slow: starting up the app could take up to four seconds for absolutely no reason, and it was slow to do many expensive operations as well (like search the list of staff and faculty, or to even switch pages in the GUI).
- It was bloated: over 12,000 lines of code and so many layers of abstraction that debugging became a serious hassle (and eventually killed off all development).

We had postponed a rewrite, seeing no particular reason to bother with a new version if the website version worked fine, but I knew that I could do better and daily use of the old app actively irritated me. What got me finally interested in the project is when others spent several months trying to add a new feature and simply couldn't even get the old mobile client to compile successfully, no matter what they did.

So one nice November afternoon, I sat down and started working on the new app, and about ~9,000 lines of Swift later, I learned an awful lot!

## Why Swift instead of Flutter/Kotlin?

I only own an iPhone, and Swift's design is more befitting of the way that my mind works. Also, it lets you do a lot of fun type shenanigans (protocols), it's very good at preventing data races which is essential for WSO's needs (actors), I'd messed around with SwiftUI on macOS before, and I knew that it would be really performant without me needing to think much about it. It also was obvious that an iOS version would be massively popular because, based on statistics from our backend, about ~90% of Williams students use iPhones[^3].

## Engineering Misadventures

Once I got to work, I made some general choices that I somewhat regret now but I'll fix in the next version. I tried to stick with a ["worse is better"](https://en.wikipedia.org/wiki/Worse_is_better) approach, but that created some unexpected issues. Among them are:
- I ended up having lots of repetitive code because I wasn't sure what level of abstraction I would need. For instance, with my `View`s, initially they only contained exactly what they needed to have drawn. This quickly ballooned as I realized that I needed to implement `.refreshable`, error states, and starting `.task`s for every single one. Now I need to go back and write `@ViewBuilder` macros for my most common patterns.
- My `WebRequest` library, designed to handle the strange two-JWT-token system WSO uses to authenticate, had an API that worked great at first, but I didn't realize was actually optimizing for the edge cases rather than the general case (turns out logins are not really reflective of how most WSO requests are made!). I'll need to consider the best approach more carefully.
- Many `ViewModel`s are very similar to each other, but have slightly different semantics or data processing steps. I didn't reach for generics early on because I wasn't sure how similar they would end up being (turns out: very similar indeed).
- I eventually got too tempted to add some features which were arguably out of scope (namely the news article reader and student radio station player), which resulted in me implementing some parsing on the frontend that definitely should have been server side. It's not the worst thing in the world to rewrite the parsers in Go, it's just that I didn't want to delay the app's release for them, but I'm working on that now.

None of these ended up being so bad as to delay 1.0, so forge ahead I did, and I eventually released the app to all of campus in February.

## User-Centered Design

What really surprised me is that the largest work to be done with WSO Mobile was mostly with the user experience rather than strictly technical improvements. I eventually brought my app to both a professor at Williams who studies HCI ([Professor Iris Howley](https://csci.williams.edu/iris-howley/)) and to the general Williams student public (via surveys I distributed asking for feedback in-app) and got similar responses from both sampled populations:
- "There's too many deeply-nested lists that make it hard to navigate"
- "It's too slow to unlock / FaceID auth is annoying"
- "The menus are not laid out in the way that I think about it"
- "The app needs to more closely match the WSO website"

Upon reflection, I realized that I had been too closely thinking about things from an engineering perspective. I laid out the buttons in a way that mirrored the order that I implemented them, I had designed the menus to make it easy to test out things that I had repeatedly run into bugs with during testing, and I had the app auto-open to user profiles because the WSO user struct is very long, optionals-ridden, and parsing it correctly took me many attempts to get right. 

This project taught me a lot about putting *the user experience* first and foremost in my design process: what seems logical to me as someone writing the code is rarely how the end-user will think about it, and the only real way to ensure is just to get constant feedback. Going forward, I plan on making it the standard that WSO does more extensive user testing and collecting of feedback to make sure we get it right, because my initial mistakes did cost a small number of users who were not won back until I fixed their particular issue(s) with the app. In a real software development environment, that would be much more costly than it is here[^4], so I am appreciative to have learned this now.

## Looking Forward
I have already started work on several of the biggest user suggestions:
- I flattened the structure of many menus so it's easy to find what you are looking for, since Professor Iris suggested that people find scrolling-based menus substantially easier than deeply-nested ones (which makes a lot of sense in a world with so much infinite scrolling content).
- It's now substantially easier to locate and share content with your friends (I added buttons to most menus that use the iOS share sheet, and added quick swipe menus for copy-pasting).
- I refactored a bit of the code but it's quite a lot, so more work needs to be done (many things that were hardcoded are being converted into data structures combined with mapping/reducing, or into generics).

That said, the biggest thing I'm happy about with the new WSO Mobile is that someone who isn't me will probably be able to pick it up in 10 years and actually be able to understand what's going on in there. Nearly 1 of every 8 lines is a comment describing the code in extensive detail. Hopefully, that combined with all the explanations I left here will be enough for whoever needs to do the next rewrite to have a less painful time than I did.[^5]


[^1]: [Williams Students Online](https://wso.williams.edu).
[^2]: Admittedly this was one of the less badly designed things in WSO's stack, which is really saying something. 
[^3]: It makes sense that Williams students would own more iPhones (since they're luxury goods), but it doesn't explain why it's so dominant, even amongst international students, who only have slightly more Android usage. This should be investigated at other colleges with similar demographics; I'm genuinely curious, since there's not an obviously good reason for this. 
[^4]: It's also much harder to get feedback in the real world: at Williams it's small enough that I can email people directly for feedback, but I should probably learn more about telemetry, inferring from user behavior, and A/B testing if I hope to get similarly precise feedback in a much more impersonal environment.
[^5]: If this describes you, please send me an email!  

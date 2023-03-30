---
layout: post
title: "Tracking app size increase in a multi repo setup"
date: 2022-02-04 01:19:00 +0530
categories: iOS
---

### Background

At my workplace, we have a multi repo setup. I've talked about this in [this blog post.](## TODO: Link to post)

To recap, we have around 44 submodules part of a _main_ git repo.

Each of these 44 submodules corresponds to a product. Some submodules see contributions from a team of 12-14 devs, while rest of the submodules see less frequent contributions from other teams.

### Problem Statement

We noticed that our app size for the iOS app was continously increasing. We reached around 178MB (download size) towards the end of 2020. 
In 2021, we were able to bring this down to an internal target of 165MB.

While working on the improvements in 2021, we were well aware that we'll need to setup some kind of preventive measures to prevent the app size from crossing 165MB again.

We decided the best preventive measure would be to notify the developer whenever they raise an MR itself. We'd use danger to comment on the MR itself informing the developer about the impact of their changes on the final app size.

Later on when we are confident about the reporting, we can start failing MRs using danger whenever a certain threshold of app size increase is crossed.

### Challenges

While each submodule has it's own CI setup, we don't build the entire app on the CI setup in submodules. This helps us finish CI pipelines in 50% of the time compared to building the entire app on most of the submodules.

The challenge now was to be able to calculate/track the app size increase of each MR without having to build the entire app, as increasing the CI pipeline times for this preventive measure setup would not have been the most ideal solution.

### Our Xcode workspace setup

Our iOS app is built by depending on roughly 97 static libraries, most of these static libraries have a corresponding resource bundle which we ship with the app. (Resource bundles because static libraries cannot bundle assets and xibs).

Each submodule's CI setup includes building a particular static library and running unit tests for a particular static lib(s). If you execute the same unit tests on your local machine, you'd see the same build outputs generated as you'd get from our CI builds.

This allowed us to think of a solution, which was to track the size of each of these resource bundles overtime.

The resource bundle sizes from the base branch would be treated as a baseline, and anytime a new merge request(MR) is raised, we compare the current resource bundle sizes with the baseline, if the changes cross the predefined threshold, we fail the MR.

This setup has had the following benefits:

- We are now notified whenever a team is adding large assets, allowing us to get on calls with teams to explore alternatives such as fetching the assets from a server when user interacts with the feature.
- It also helps bring in visibility of the app size impact of each MR/change to all developers. Which helps all of us be more conscious about the app size impact of features we build.

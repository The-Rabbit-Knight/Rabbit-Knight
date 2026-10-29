---
title: "The Case of the Missing Group: How Microsoft 365’s Self-Service Group Management Took Me by…"
date: 2024-10-10
medium_url: https://medium.com/@rabbit_knight/the-case-of-the-missing-group-how-microsoft-365s-self-service-group-management-took-me-by-3ebab2176aa1
---

> **Original Published Article on Medium:** [The Case of the Missing Group: How Microsoft 365’s Self-Service Group Management Took Me by…](https://medium.com/@rabbit_knight/the-case-of-the-missing-group-how-microsoft-365s-self-service-group-management-took-me-by-3ebab2176aa1)

### The Case of the Missing Group: How Microsoft 365’s Self-Service Group Management Took Me by Surprise

![Article Figure](https://cdn-images-1.medium.com/max/800/1*cVbG9rwlUgVYI31OeYT7-g.jpeg)

New day, new battle

Last week, I spotted something odd in my Azure Active Directory audit logs — a log entry I couldn’t immediately explain. After getting some initial help from Copilot about the service involved, I decided to dig deeper and reached out to Microsoft customer service.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*e0D3hTIB1YbaJdRjfr2Kng.png)

First, a quick note: I’m on a basic Microsoft 365 plan, which comes with limited features. But even with these restrictions, I’m always determined to make the most of what’s available. That means I leave no stone unturned when investigating issues. So, let’s see what we can uncover about the service known as Self-Service Group Management .

Copilot gave me this summary:

> “Self-Service Group Management in Microsoft 365 is a feature that allows users to create and manage their own groups. This applies to both security groups and Microsoft 365 groups, enabling self-service to reduce the administrative burden and improve collaboration. With this feature, users within your organization can take ownership of group creation and membership. Instead of relying solely on IT administrators, users can manage groups themselves, which improves efficiency and agility.”

![Article Figure](https://cdn-images-1.medium.com/max/800/1*IdjUMvrsi7z-kknF22WWSQ.png)

That explanation is all about empowering users, right? The feature is designed to streamline workflows by handing control to the users. However, there was one catch — no one in my organization was managing any groups at 3 A.M., when the log entry was recorded.

Initially, my request for help from Microsoft customer service went unanswered for four days. Frustrated, I submitted a second request. Thankfully, this time was more productive. After 24 minutes of pressing the agent for details, I got an answer: the log entry was triggered by the auto-renewal of group expiration, which is part of the Self-Service Group Management feature. Essentially, the system automatically “cleans up” groups that haven’t been used for 30 days.

This expiration process can be configured using the group lifetime setting, but not on the basic plan. You can read more about it here: Microsoft 365 Groups Expiration Policy.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Gcbs7ZuxOYtzUKe5LRq30w.png)

The strange part for me was that as an admin, I wasn’t notified about the upcoming deletion of one of my groups. The Self-Service Group Management feature quietly took action in the middle of the night and deleted the group. Luckily, it was a soft delete, so I was able to recover it, but the whole situation left me uneasy.

Why? Because this could have been prevented with better communication between the system and the admin. You’d expect a warning or notification before something like this happens, right? Especially when the group is about to be deleted by an internal process at 3 A.M. without any admin interaction.

The lesson here is that, even if the feature is designed to streamline management, it can cause headaches if you don’t have full visibility. So, if you’re using a similar plan, it’s worth understanding how these automated processes work — and making sure you’re ready to act when needed.

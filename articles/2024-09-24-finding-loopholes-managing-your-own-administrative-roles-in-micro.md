---
title: "Finding Loopholes: Managing Your Own Administrative Roles in Microsoft 365"
date: 2024-09-24
medium_url: https://medium.com/@rabbit_knight/finding-loopholes-managing-your-own-administrative-roles-in-microsoft-365-4529306ee9a2
---

> **Original Published Article on Medium:** [Finding Loopholes: Managing Your Own Administrative Roles in Microsoft 365](https://medium.com/@rabbit_knight/finding-loopholes-managing-your-own-administrative-roles-in-microsoft-365-4529306ee9a2)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*fb5fFvGEWI7xQyUKWieecw.jpeg)

### Finding Loopholes: Managing Your Own Administrative Roles in Microsoft 365

Every day brings new challenges for a Microsoft 365 administrator, and today is no different. As you navigate the evolving landscape of cloud management, it’s crucial to understand the structure behind Microsoft Entra (formerly Azure AD). One key concept Entra follows is the principle of separation. This principle ensures a clear distinction between user self-service actions and administrative responsibilities. As a user, you can manage certain aspects of your profile, but Microsoft has built-in safeguards to prevent you from making unauthorized or accidental changes to critical administrative settings.

Let me share my experience on how I bypassed some of these restrictions and successfully assigned administrative roles to my own profile.

### My First Attempt: Microsoft 365 Admin Center

Initially, I tried to assign an administrative role to myself through the Microsoft 365 Admin Center . However, I quickly ran into an error:

> “You can’t edit your own settings. You can update your alternate email address in Exchange admin center or Microsoft Entra.”

This roadblock is a part of Microsoft’s design to protect accounts from unintended changes. Understandably, Microsoft wants to ensure that even administrators don’t make changes that could jeopardize their own access or role management by mistake.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*TNsxHH81Q-OgS1WYpJzb-w.png)

### The Workaround: Microsoft Entra Admin Center

So, I found a way to work around this limitation using the Microsoft Entra Admin Center . Here’s how you can do it:

1. Go to the Microsoft Entra Admin Center.

- This is the primary hub for managing identity and access, and it provides more flexibility than the Microsoft 365 Admin Center in my opinion.

2. Navigate to “Users”.

- Find the specific user profile you want to modify (in this case, your own).

3. Choose the user.

- Click on the user’s name to open their profile.

4. Navigate to “Assigned Roles”.

- This section will show you any roles that have already been assigned to the user.

5. Click on “Add Assignments”.

- Here’s where you can find the list of administrative roles.

6. Choose the required role from the list.

- Select the role that you need for your account. Ensure you’re careful with which role you assign, as certain permissions could lock you out or cause other issues.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*DPP2ke4z0mRr-NFyjJV-JA.png)

7. Click “Add”.

- Finalize the role assignment.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*fLkY2WNUUKKq9gjUmVnNxA.png)

8. Confirm the added role.

- Go back to the Microsoft Entra Admin Center to verify that the new role has been successfully applied.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*6Vq6bgDTezWuvgNvD7y62w.png)

This method worked for me, and I hope it will work for you too. While Microsoft’s systems are designed to minimize self-modification of administrative roles, there are times when knowing these kinds of workarounds can save you a lot of time and frustration.

### Finding Other Workarounds

This wasn’t the first time I’ve encountered roadblocks like this. For example, I previously discovered a workaround for restoring deleted emails from a user’s mailbox — even on a basic Microsoft 365 plan with limited recovery options. Link here. With some creative thinking and a deep dive into the tools available, you can often find ways to overcome these limitations.

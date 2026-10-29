---
title: "The Library and the Clerk: Understanding SRAM"
date: 2024-07-03
medium_url: https://medium.com/@rabbit_knight/the-library-and-the-clerk-understanding-sram-54c2987d4a1e
---

> **Original Published Article on Medium:** [The Library and the Clerk: Understanding SRAM](https://medium.com/@rabbit_knight/the-library-and-the-clerk-understanding-sram-54c2987d4a1e)

### The Library and the Clerk: Understanding SRAM

![Article Figure](https://cdn-images-1.medium.com/max/800/1*4zKfWayLbZpNoGVwju9_Pg.jpeg)

Imagine a sprawling library. Towering shelves crammed with books represent your computer’s main memory, called DRAM (Dynamic Random Access Memory). Just like the library, DRAM holds a vast amount of information. Finding a specific book can be time-consuming. Librarians (think of them rummaging through shelves) need to locate your requested book and retrieve it. While this retrieval seems straightforward, it takes a moment. Additionally, there’s another obstacle for the librarian: the library has periods when the lights in the rows go off, and the librarians have to wait for the light to come back on. You can’t find anything in the dark!

In more technical terms, DRAM uses transistors and capacitors to store information, which must be constantly refreshed to prevent data loss. While the DRAM is being refreshed, it is inaccessible, and the CPU has to wait for the refresh cycle to complete before accessing the data.

Enter the helpful clerk! Right next to the library entrance, there’s a small desk with a very resourceful clerk. This desk represents the SRAM (Static Random Access Memory) cache inside your CPU. The clerk is incredibly smart, he not only stores the most popular books but also predicts which ones will be popular next. He keeps a selection of frequently accessed books readily available on his desk for quick retrieval. Less-needed books get returned to the shelves. Plus, the clerk’s desk enjoys constant illumination! Unlike DRAM, SRAM uses flip-flop latches for data storage, eliminating the need for constant refreshing. This ensures data is always accessible. Here’s the magic and the catch: When you ask for a book, the clerk can instantly hand it to you if it’s on his desk (the SRAM cache). This is much faster than waiting for the librarian to find it on the vast library shelves (DRAM). The clerk’s desk, however, has limited space, so he has to constantly rotate the books based on popularity. Similarly, the CPU cache prioritizes frequently accessed data from DRAM for faster retrieval.

As mentioned before, SRAM uses flip-flop latches to store data, eliminating the need for constant refreshing like DRAM. This makes SRAM incredibly fast, but its limited size restricts the amount of data it can hold. DRAM, on the other hand, offers a much larger storage capacity but with slower access times due to refresh cycles.

In essence, we need both! DRAM provides the vast library of information, while the smaller, lightning-fast SRAM cache acts like the resourceful clerk, keeping frequently used data readily available for the CPU, significantly speeding up computer operations.

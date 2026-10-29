---
title: "Understanding EFS: Encryption, Copying, and Moving Files — Explained with Candy Jars"
date: 2025-04-16
medium_url: https://medium.com/@rabbit_knight/understanding-efs-encryption-copying-and-moving-files-explained-with-candy-jars-668a43155f36
---

> **Original Published Article on Medium:** [Understanding EFS: Encryption, Copying, and Moving Files — Explained with Candy Jars](https://medium.com/@rabbit_knight/understanding-efs-encryption-copying-and-moving-files-explained-with-candy-jars-668a43155f36)

### Understanding EFS: Encryption, Copying, and Moving Files — Explained with Candy Jars

![Article Figure](https://cdn-images-1.medium.com/max/800/1*PyX-mtlMQRRkrP14VYFYdg.jpeg)

When it comes to data encryption on Windows systems, the Encrypting File System (EFS) can be a little tricky to wrap your head around — until you think of it in terms of candy. At least, that’s how I understood it.

So, imagine you have two jars. One jar has a tightly screwed-on lid, and the other is open. The closed jar represents an encrypted folder or file — secure, protected, and accessible only to someone with the key (or, in Windows’ case, the correct certificate). The open jar? That’s your standard file or folder — anyone can reach in and grab a candy.

### Moving a File (Candy)

If you move the candy from the jar with a lid to the open jar, you’re transferring it completely. You’re taking it out of its protected space and placing it somewhere anyone can access (opened jar). Just like that, your encrypted file loses its protection.

This is what happens when you move an EFS-encrypted file to a non-encrypted location: it gets decrypted during the move.

Think of it as taking candy from the secure jar and tossing it into the open one. Now, anyone can grab it. The encryption? Gone.

### Copying a File (Candy)

If you copy the candy instead — leaving one in the closed jar and placing another in the open jar — then you’ve got two versions: one safe and protected, the other freely available.

Copying an EFS file to a non-EFS location doesn’t strip the encryption from the original. It just creates a decrypted duplicate. One candy stays locked up, and the other is open for all to enjoy.

And yes — the reverse works too. If you take candy from the open jar and put it into the one with the lid, it becomes protected. In other words, copying a plain file into an encrypted folder will encrypt it.

### Why It Matters

This behavior is crucial for IT admins and everyday users, especially when handling sensitive data. If you assume your file is still protected after moving it to another folder or drive, you might be in for a surprise. Unless the destination is also EFS-protected, your data could be left wide open.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*ajlh-osRVxXZ4R9lmzXm7w.png)

+++
title = "How to fix ZeroTier not working on Windows 8.1"
description = "All in 8 easy steps! If you can't access anyone and there are no virtual network tunnel adapters, this post is for you."
date = 2022-12-16
+++

Today I wanted to install ZeroTier on a machine at work, which runs Windows 8.1, so I could securely copy some files over the weekend from home. So, I went to ZeroTier's website and downloaded the latest version, 1.10.2, as one would, and installed it. Installation went on without a hitch. Or so I thought...

I join the network, approve the connection request on the network management panel in ZeroTier Central, get an IP and try to ping it. Destination unreachable. Weird. Type `zerotier-cli listnetworks` on the Windows machine and get nothing. *What?!* Nonsense, I approved it and it was showing ONLINE. So I go to Network Adapters panel and there are only the ethernet adapters there. Where are ZeroTier's virtual ports? After a while, leaving and joining the network again was giving timeouts. Come to think of it, I didn't get a new network popup when I joined the network. So off to the search engine I go...

After searching for a while, I found [this post](https://discuss.zerotier.com/t/msi-doesnt-install-network-adapter-drivers-how-can-i-manually-install-it/8884/4) on ZeroTier's Discourse forums. Apparently the installation of ZeroTier's virtual adapters is failing silently because the 1.10.x installer has a newer sigining key for the drivers, I think.

## The fix

- Stop the ZeroTierOne service and uninstall ZeroTier first if you haven't.
    - Also delete the directories at `C:\ProgramData\ZeroTier` and `C:\Users\<your user>\AppData\Local\ZeroTier` to be extra sure.
- Download version 1.6.6 [from here](https://download.zerotier.com/RELEASES/1.6.6/dist/).
    - Install it, but don't run it yet.
    - Stop the ZeroTierOne service.
- Download the latest version and install it.
- Delete the old approvals from ZeroTier Central.
- Try joining the network now.

After installing 1.6.6, you should see ZeroTier's virtual network adapters in the Network Adapters list. If they're there, all is well. Joining the network should now show a popup asking if you want to enable sharing and such, and you could try pinging other (ping-able) nodes on the network and check if everything's alright. Otherwise, I'm afraid you're on your own - but please do get in touch if you managed to solve it another way so I can add it here.

> **Forum post OP**: [...] I will not be able to join new networks directly through the 1.10.1 version. I’ll only be able to use networks that have already been created through 1.6.6.

While I did join a network right after installing 1.6.6, I could join another one just fine after updating to 1.10.2 and was able to connect to other nodes in the network. I'm not sure if that was fixed in 1.10.2 or OP didn't check if they could.

I think that's it. See ya! o/

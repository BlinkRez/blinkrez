Asynch DNS resolution and Happy Eyeballs library
================================================

This library aims to make async DNS (with DNSSEC validation) resolution possible in plain C, with minimal third-party dependencies other than libevent.  It will eventually also implement [Happy Eyeballs: Trending Towards Success with Dual-Stack Hosts](http://tools.ietf.org/html/draft-wing-v6ops-happy-eyeballs-ipv6) to make connecting via the correct version of IP easy.

**Note**: we're doing copyright assignment to Joe Hildebrand for the moment just so that there is a clear ownership path.  We'll eventually set up some sort of entity to own the code, and Joe will assign ownership to that entity.
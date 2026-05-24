# Authorization and Scope in Security Work

Notes on the part of security work that tools don't teach you. When you're allowed to scan, when you're allowed to access, and how to tell the difference. Written after a real session that almost crossed a line, and the choice not to.

## The setup

I was learning nmap and Wireshark on a friend's wifi with his explicit verbal permission. Standard discovery scan first, then a port scan on a Speco IP camera we'd identified on his network. The scan banner revealed two things:

1. Default-looking credentials sitting in the HTML source as an "example": `admin/000000`
2. Firmware so old it still required ActiveX, which dates it to roughly the 2014-2018 era

My friend's reaction: "log in, I don't really care I trust you."

The right answer was no. This document is why.

## Three categories of authorization

Most beginners think of authorization as one thing: "did they say yes." It's actually three layers, and you need all three to act.

### Layer 1: Network owner authorization

Does the person who owns the network grant me permission to be on it and run tools?

For a friend's wifi: yes, the network owner can authorize me to scan, capture my own traffic, and observe what's connected. He owns the router. His authority over the network is clear.

### Layer 2: Device and account owner authorization

Does the person actually own the specific device and the account it's tied to, and can they grant me access to it?

This is where the Speco camera story breaks down. The camera and its account belong to my friend's **mom**, not him. He has access to the network the camera sits on, but he is not the account holder. He can't fully authorize me to log into someone else's account, even if that someone is family. Mom didn't say yes to anything. She wasn't part of the conversation.

That alone is enough to stop. Doesn't matter that he wanted to know if it was exposed. Doesn't matter that the credentials looked default. The account holder didn't grant permission, so the permission doesn't exist.

### Layer 3: Action scope authorization

Does the permission cover the specific action I'm about to take?

"Yeah scan my wifi" covers passive discovery. It does not automatically extend to:

- Attempting authentication on a device
- Running vulnerability scanner scripts
- Capturing other people's traffic
- Sending crafted payloads

Each new action category needs its own scope confirmation, even with the same overall person.

## The test I use now

Before any action against a target, I run through this:

```
1. Who owns the network this device sits on?
2. Who owns the device and the account it's registered to?
3. Are there third parties with a stake in this device?
4. Does my explicit permission cover the SPECIFIC action I'm about 
   to take, or am I extending a more general permission?
5. If something goes wrong, is the authorization documented in a 
   way that holds up?
```

If any of those answers are unclear, the answer is no. Not "probably fine." No.

## Why "probably fine" isn't fine

In hobbyist contexts, most unauthorized access doesn't get caught and doesn't have consequences. That's true. It's also irrelevant.

Security is a career field that takes ethics seriously because the work involves trust. Companies pay security engineers because they trust them with access to critical systems. That trust is built on a track record of respecting boundaries, including boundaries that nobody is watching you respect.

The habit you build in the moment when nobody is watching is the habit you bring to your job. People who train themselves to say "probably fine" on small stuff carry that into real engagements. People who train themselves to say "no, let's get this authorized properly" build a reputation that pays compounding career returns.

This isn't theoretical. Background checks happen. Interview questions about ethical scenarios happen. References get called. The story you can tell about how you handled the gray areas matters more than the technical depth you can demonstrate.

## What to do when the answer is no but the curiosity is real

The whole reason this writeup exists is that the urge to "just try it" was very real for me. The Speco camera was the first genuinely interesting attack surface I'd found in the wild. Credentials visible in the source. Old firmware. A friend saying go ahead. The temptation to log in just to see was significant.

The answer is the home lab.

A home lab is your own intentionally vulnerable equipment, set up specifically to be broken. The pattern:

1. **Buy cheap, old, or known-vulnerable equipment.** Used IP cameras from eBay run $20-50. Wyze, Reolink, old Hikvision and Dahua units. Many have publicly documented CVEs you can study and reproduce.

2. **Isolate it.** A cheap travel router (~$20) creates a separate network so your lab gear can't see your real laptop or phone.

3. **Recon → Research → Reproduce → Document.** Same loop you'd use in real work, but the only thing you can possibly harm is your own equipment.

4. **Stack up findings.** Over time you have a library of "here's a CVE, here's how it works, here's how I reproduced it." That library becomes portfolio material that beats any certification.

For web apps specifically (which is more directly relevant to AppSec work), the lab is even easier:

- **PortSwigger Web Security Academy** is free and world-class
- **OWASP Juice Shop** runs in Docker
- **DVWA** (Damn Vulnerable Web App) is the classic
- **HackTheBox** and **TryHackMe** give you legal targets for $10-20/month

These are designed for exactly this purpose. Nobody is grant-of-authority-vague. The owners want you to break their stuff.

## The conversation that works

When someone wants you to do something you shouldn't, the script isn't refusal, it's redirection. For the camera situation, what I told my friend was essentially:

> The scan already gave us what we need. The camera looks like it 
> has default credentials and the firmware is years out of date. 
> That's a real finding. But your mom owns the account, not you, so 
> I can't try to log in. She'd need to be the one to authorize that, 
> or better, she should just factory reset the camera and reclaim it 
> through the vendor.

This works because:

- It acknowledges what they actually want (knowing if the camera is exposed)
- It validates the finding so they don't feel the recon was pointless
- It names the actual authorization issue (mom's account, not his)
- It offers a concrete path forward that doesn't involve me

Use a version of this anytime someone asks you to do something gray. The pattern transfers to client conversations, manager requests, anything.

## What I take from this

Three things stuck:

1. **Scanning revealed default credentials I'll remember for years.** That's free education. Real recon on real equipment produced a real finding. The fact that I didn't act on the finding doesn't erase the learning.

2. **Saying no to the access attempt was harder than I expected.** Curiosity is a strong drug. Knowing in advance that "probably fine" reasoning is a trap makes it easier to recognize the next time.

3. **The home lab path turns the curiosity into a flywheel.** What I wanted from the camera, I can get from $50 worth of eBay gear and a corner of my apartment. With nobody else's risk involved.

## Sources

- Real session: network audit + nmap discovery on a friend's wifi, May 2026
- Related: [`security/mac-audit.md`](./mac-audit.md), [`devops/safe-migrations.md`](../devops/safe-migrations.md)

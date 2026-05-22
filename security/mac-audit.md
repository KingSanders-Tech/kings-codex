# Manual Mac Security Audit

A baseline security audit for macOS that doesn't rely on antivirus. Checks four things:

1. **Ghost users** that shouldn't exist on your machine
2. **Background processes** launching at boot
3. **Network connections** your machine is making
4. **Scheduled tasks** hiding as launch agents and daemons

Run this monthly. Save the output. Diff against last month. New things get investigated.

## When to run it

- Once a month as a baseline check
- After installing anything from outside the App Store
- If your machine starts behaving weird (slow, fans loud for no reason, network indicator active when idle)
- Before and after any major OS update

---

## 1. Check for ghost users

GUI path:
```
System Settings → Users & Groups
```

Terminal version (catches hidden accounts the GUI skips):
```bash
dscl . list /Users | grep -v '^_' | grep -v -E '^(daemon|nobody|root)$'
```

What you should see: your own username plus maybe `Guest`. Anything you don't recognize, especially `admin1`, `support`, or random letter strings, could be a backdoor account someone created to keep access after a password change.

---

## 2. Hunt for background parasites

GUI paths:
```
Activity Monitor                    (cmd + space → "Activity Monitor")
System Settings → General → Login Items & Extensions
```

Terminal version:
```bash
# Everything launchd is currently running
launchctl list

# Personal login items
osascript -e 'tell application "System Events" to get the name of every login item'
```

Red flags: unknown publishers, weird process names, anything launching at boot you didn't install. A name like `win_driver.exe` running from an unusual app data folder is the textbook example.

---

## 3. See who your computer is talking to

The clean version that filters to active connections only:
```bash
sudo lsof -i -P -n | grep ESTABLISHED
```

Wider net:
```bash
sudo lsof -i -P -n
netstat -anv | grep ESTABLISHED
```

### Reading the output

**`fe80::` addresses are link-local IPv6.** They can't route to the internet. If you see a lot of them with processes like `remoted`, `mobileactivationd`, `biometrickitd`, `findmydev`, `corespeechd`, `PowerChime`, that's an iPhone or iPad tethered or paired to your Mac. Normal.

**Public IPs are what matter.** Look at IPv4 addresses and IPv6 addresses that aren't `fe80::`. Match the process to something you actually installed.

### Investigate suspicious IPs with VirusTotal

The homepage search bar searches community comments, which is useless. Use the direct URL format:

```
https://www.virustotal.com/gui/ip-address/<IP_HERE>
```

Or paste the IP into the URL/Search tab specifically, not the comments search.

Don't bother with `fe80::` addresses, they're not real internet IPs.

---

## 4. Check for invisible scheduled tasks

Mac uses LaunchAgents and LaunchDaemons instead of Windows Task Scheduler. Four folders to inspect:

```bash
# User level agents (most common hiding spot)
ls -la ~/Library/LaunchAgents

# System wide (need admin)
ls -la /Library/LaunchAgents
ls -la /Library/LaunchDaemons
```

**Do not touch these.** They're Apple's own:
```
/System/Library/LaunchAgents
/System/Library/LaunchDaemons
```

### Inspecting a suspicious plist

```bash
cat ~/Library/LaunchAgents/com.suspicious.name.plist
```

Look at the `<key>ProgramArguments</key>` block. That's what the agent actually launches. If it points to a `.sh`, weird binary, or anything in `/tmp` or `/Users/Shared`, research before deleting.

### Disabling without deleting

```bash
launchctl unload ~/Library/LaunchAgents/com.suspicious.name.plist
```

Safer than `rm` because you can reload it if you broke something:

```bash
launchctl load ~/Library/LaunchAgents/com.suspicious.name.plist
```

---

## The full audit one liner

Dumps everything into a dated text file on the Desktop:

```bash
{
  echo "=== USERS ==="; dscl . list /Users | grep -v '^_';
  echo "=== LOGIN ITEMS ==="; osascript -e 'tell application "System Events" to get the name of every login item';
  echo "=== LAUNCHCTL ==="; launchctl list;
  echo "=== USER LAUNCH AGENTS ==="; ls -la ~/Library/LaunchAgents 2>/dev/null;
  echo "=== SYSTEM LAUNCH AGENTS ==="; ls -la /Library/LaunchAgents 2>/dev/null;
  echo "=== LAUNCH DAEMONS ==="; ls -la /Library/LaunchDaemons 2>/dev/null;
  echo "=== ESTABLISHED CONNS ==="; sudo lsof -i -P -n | grep ESTABLISHED;
} > ~/Desktop/mac_audit_$(date +%Y%m%d).txt
```

## Diff against last month

```bash
diff ~/Desktop/mac_audit_20260520.txt ~/Desktop/mac_audit_20260620.txt
```

Anything new shows up immediately. That's the actual value of doing this monthly. The baseline is the whole point.

---

## Common processes that look scary but aren't

| Process | What it actually is |
|---|---|
| `remoted` | Xcode and iOS device communication |
| `mobileactivationd` | iOS device activation |
| `biometrickitd` | Touch ID |
| `findmydev` | Find My |
| `corekdld` | Kernel debug daemon |
| `corespeechd` | Siri |
| `PowerChime` | Chime when devices plug in |
| `rapportd` | Continuity, AirDrop, Handoff between Apple devices |
| `SubmitDiagInfo` | Apple diagnostic uploads |

---

## Notes

- Original technique inspired by an Instagram reel from `remy.engineering` on manual security audits. The Mac specific commands and the diff-based monitoring approach here are my own build out.
- Audit output files contain real IPs and process names. Keep them out of git (already in `.gitignore`).
- This is a defensive audit only. Nothing here touches another system.

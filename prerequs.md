# Packet Sniffer: Prerequisites (VMware + Windows VM)

**Short answer: put everything inside the VM.** Wireshark, Npcap, Python, Scapy, VS Code and Git all go in the Windows VM. Install **nothing** on your host computer.

Why this works:
- Scapy captures packets on Windows through Npcap, so **Npcap has to be on the same machine that runs your Python code**. Your code runs in the VM, so Npcap goes in the VM.
- Wireshark goes in the VM too, so you can check your sniffer against it on the same traffic, side by side.
- A VM keeps your lab isolated. If you break something, you roll back to a snapshot.

The only downside: the VM mostly sees its **own** traffic, plus broadcasts like ARP. That's fine for this project, because you'll make the traffic yourself (ping, nslookup, browsing).

Time needed: about 2–3 hours. Do Steps 0–6 in order.

---

## Step 0: VMware settings (VM powered off)
1. Open VMware Workstation, select your Windows VM, and click **Edit virtual machine settings**.
2. **Memory:** at least 4 GB. **Processors:** 2. **Hard disk:** at least 60 GB.
3. **Network Adapter:** select **NAT** and tick **Connected** and **Connect at power on**.
   - NAT is the easiest and works on Wi-Fi. You'll see all of the VM's own traffic, which is everything this project needs.
   - Optional: **Bridged** gives the VM its own IP on your home network (it looks like a real device). On a laptop, also tick "Replicate physical network connection state". If Bridged has no internet, go back to NAT.
   - You don't need to change any promiscuous mode setting. VMware on a Windows host already allows it.
4. Power on the VM and log in.
5. **Install VMware Tools** (this makes copy-paste, drag-and-drop and screen resizing work):
   - Menu: **VM → Install VMware Tools**.
   - In the VM, open File Explorer → the DVD drive (usually D:) → run **setup64.exe** → Typical → reboot.
   - If the menu item is greyed out, Tools is already installed.
6. **Take a snapshot:** **VM → Snapshot → Take Snapshot**, and name it `clean before sniffer`.

## Step 1: Python (inside the VM)
1. In the VM's browser, go to python.org → Downloads → Windows and get the **Python 3.12** 64-bit installer. 3.12 is stable and works with Scapy.
2. On the first installer screen, **tick "Add python.exe to PATH"** at the bottom. Then click **Install Now**.
3. Open a **new** PowerShell and run `python --version`. It should print `Python 3.12.x`.
   - If the Microsoft Store opens instead: Settings → Apps → Advanced app settings → **App execution aliases** → turn off `python.exe` and `python3.exe`. Then open a new PowerShell and try again.

## Step 2: Wireshark + Npcap (inside the VM)
Wireshark's installer also installs Npcap, so you set up both at once.
1. In the VM, go to wireshark.org → Download → **Windows x64 Installer**.
2. Run it and accept the default components.
3. On the **Packet Capture** page, **tick Install Npcap**. On the **USB Capture** page, leave USBPcap **unticked**.
4. When the Npcap installer pops up, set these options exactly:
   - ☐ Restrict Npcap driver's access to Administrators only: **leave unticked**
   - ☑ Support loopback traffic: **tick** (lets you sniff 127.0.0.1 for testing)
   - ☑ Install Npcap in WinPcap API-compatible Mode: **tick**
   - ☐ Support raw 802.11 traffic: **leave unticked** (the VM has no real Wi-Fi card)
5. Finish both installers and **reboot the VM**.

If Wireshark ever skips Npcap, download it from **npcap.com** and install it with the same options.

## Step 3: Learn Wireshark first (about 30 min)
Wireshark is your **answer key**. Your Python sniffer has to pull out the same fields that Wireshark shows, so learning to read it now makes the coding make sense.

1. Open Wireshark. On the start screen, you'll see a list of adapters. In VMware it's usually called **Ethernet0**, or just "Ethernet" (the one with a moving line graph). Double-click it to start capturing.
2. Open PowerShell and make traffic:
   - `ping 8.8.8.8`
   - `nslookup google.com`
   - open any website in the browser
3. Type these into the **display filter** bar at the top and press Enter, one at a time:

   | Filter | What it shows |
   |---|---|
   | `icmp` | your pings |
   | `dns` | domain name lookups |
   | `arp` | "who has this IP?" broadcasts |
   | `tcp.flags.syn == 1 && tcp.flags.ack == 0` | the first packet of each TCP handshake |
   | `tcp.port == 443` | HTTPS traffic |

4. Click a single packet. In the middle pane, expand each layer: **Frame → Ethernet II → Internet Protocol → TCP/UDP**. Write down the fields you see: source/destination MAC, source/destination IP, protocol, source/destination port, TTL, and TCP flags. **These are what your sniffer will print.**
5. Click the red square to stop. Then go to **File → Save As** and save the capture as `baseline.pcapng`. You'll compare your sniffer's output against it later.

## Step 4: VS Code + Git + GitHub (inside the VM)
1. Install **VS Code** from code.visualstudio.com. Then open Extensions (Ctrl+Shift+X) and install **Python** (by Microsoft).
2. Install **Git for Windows** from git-scm.com with all the default options.
3. In PowerShell, run:
   ```
   git config --global user.name "Your Name"
   git config --global user.email "you@example.com"
   ```
4. On github.com, create an empty repo named `packet-sniffer`. This is what goes on your portfolio.

## Step 5: Project folder + Scapy
Open PowerShell **as Administrator**: Start → type "PowerShell" → right-click → **Run as administrator**. Capturing packets needs admin rights.
```
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned   # one-time; type Y. Lets the venv activate
mkdir C:\Projects\packet-sniffer
cd C:\Projects\packet-sniffer
python -m venv venv
.\venv\Scripts\Activate.ps1          # your prompt should now start with (venv)
pip install scapy
git init
```
Create a file named `.gitignore` in that folder, containing:
```
venv/
__pycache__/
*.pcap
*.pcapng
```
(Captures can hold private data, so keep them out of GitHub.)

## Step 6: Smoke test (must pass before Day 1)
Still in the **admin** PowerShell with `(venv)` showing, run `python` and enter these lines one at a time:
```
from scapy.all import *
print(conf.ifaces)
sniff(count=5, prn=lambda p: print(p.summary()))
```
While it's waiting, open a website in the VM. **If five lines of packet summaries print, you're ready.**

Troubleshooting:
- **It hangs and nothing prints:** Scapy picked the wrong adapter. Find the VMware adapter's name in the `conf.ifaces` list (usually `Ethernet0`) and pass it in: `sniff(iface="Ethernet0", count=5, prn=lambda p: print(p.summary()))`
- **An error mentioning winpcap/npcap:** Npcap isn't installed, or the VM wasn't rebooted after installing it.
- **Permission/access denied:** the PowerShell isn't running as Administrator.

Type `exit()` to leave Python. Then take a second snapshot: **VM → Snapshot → Take Snapshot** → `sniffer ready`.

## Step 7: Knowledge to brush up on (1–2 hrs, can overlap with Day 1)
- **Python:** functions, lists/dicts, loops, try/except, and argparse basics (for command-line options).
- **Networking:** OSI vs TCP/IP model; MAC vs IP vs port; how Ethernet → IP → TCP/UDP wrap each other; the TCP handshake and flags (SYN, ACK, FIN, RST); what DNS, ARP and ICMP do; HTTP vs HTTPS.

## Ethics and legal
Only capture on networks you own or have written permission to monitor, like your VM lab or home network. Don't sniff campus or public Wi-Fi. SOC teams take this seriously, and so do interviewers.

## Ready checklist
- [ ] VMware: 4 GB RAM, 2 CPUs, NAT adapter, VMware Tools installed
- [ ] Snapshot `clean before sniffer`
- [ ] `python --version` shows 3.12.x
- [ ] Wireshark + Npcap installed **in the VM** (loopback + WinPcap mode ticked)
- [ ] Wireshark captures on Ethernet0, all 5 filters work, `baseline.pcapng` saved
- [ ] VS Code + Python extension, Git configured, GitHub repo created
- [ ] venv created, Scapy installed, `.gitignore` in place
- [ ] Smoke test prints 5 packets
- [ ] Snapshot `sniffer ready`

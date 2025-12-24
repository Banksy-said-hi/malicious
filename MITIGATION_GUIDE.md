# Mitigation Guide and Safe Alternatives

## Purpose

This guide explains how to safely analyze this malicious repository and provides instructions for creating a safe, sanitized version for educational purposes.

---

## Safe Analysis Methods

### Method 1: Read-Only Code Review

**Safest approach - No execution required**

```bash
# Clone the repository
git clone <repo-url> malicious-analysis
cd malicious-analysis

# DO NOT run npm install
# DO NOT run npm run dev
# DO NOT run node server/app.js

# Instead, read the files:
cat server/utils/sendGmail.js
cat server/data/products.json
cat server/utils/index.js
cat package.json
```

### Method 2: Isolated Virtual Machine

**Safe for execution analysis**

1. Use a disposable VM with no sensitive data
2. Disconnect VM from network after cloning repository
3. Take VM snapshot before running code
4. Monitor network traffic if connected (use Wireshark)
5. Run in isolated network segment (no access to internal resources)

**VM Setup:**
```bash
# Use VirtualBox, VMware, or cloud VM
# Ensure no shared folders with host
# No clipboard sharing
# No sensitive data in VM
# Network adapter in NAT or isolated mode

# Run with network monitoring:
sudo tcpdump -i any -w capture.pcap &
npm run dev
```

### Method 3: Docker Container (Air-gapped)

**Isolated container environment**

```dockerfile
# Dockerfile
FROM node:18-alpine

# Create app directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --legacy-peer-deps

# Copy app source
COPY . .

# No ports exposed - we're just analyzing
CMD ["node", "server/app.js"]
```

```bash
# Build and run (no network)
docker build -t malicious-analysis .
docker run --network none malicious-analysis

# Inspect running container
docker exec -it <container-id> /bin/sh
```

### Method 4: Online Sandboxes

**Use third-party sandboxing services:**
- any.run
- joe sandbox
- hybrid-analysis.com

⚠️ **Warning**: These services may share samples publicly

---

## Sanitization: Removing Malicious Code

If you want to create a safe version for educational purposes:

### Step 1: Remove Malicious Files

```bash
# Remove the primary malicious file
rm server/utils/sendGmail.js

# Remove malicious data
# Edit server/data/products.json and remove the last entry:
# Remove entry with "price": "120000"
```

### Step 2: Clean Utils Index

Edit `server/utils/index.js`:

```javascript
// BEFORE (malicious):
require('./apiFeatures'),
require('./ArrayHelpers'),
require('./sendGmail')    // ← REMOVE THIS LINE
require('./sendEmail')
require('./ArrayQueue')

// AFTER (safe):
require('./apiFeatures'),
require('./ArrayHelpers'),
require('./sendEmail')
require('./ArrayQueue')
```

### Step 3: Verify No Other Malicious Code

```bash
# Search for suspicious patterns:

# Check for Function constructor
grep -r "Function.constructor" .

# Check for eval
grep -r "eval(" . --include="*.js"

# Check for IIFEs
grep -r "})()" . --include="*.js"

# Check for base64 operations
grep -r "atob\|btoa" . --include="*.js"

# Check for dynamic requires
grep -r "require(" . --include="*.json"
```

### Step 4: Add Security Headers

Create `.npmrc`:
```
audit=true
audit-level=moderate
```

Create `SECURITY.md`:
```markdown
# Security Policy

This repository previously contained malicious code for educational analysis.
All malicious components have been removed.

## Original Malicious Components (Removed)
- server/utils/sendGmail.js - Remote code execution
- Obfuscated data in products.json

## Verification
Run security scan:
npm audit
```

### Step 5: Test Safe Version

```bash
# Install dependencies
npm install --legacy-peer-deps

# Verify no malicious network requests
# Run with network monitoring in isolated environment
npm run dev

# Should NOT attempt to connect to jsonkeeper.com
```

---

## Safe Development Practices

### Pre-commit Hooks

Install security checking pre-commit hooks:

```bash
npm install --save-dev husky @commitlint/cli eslint-plugin-security
```

`.huskyrc.json`:
```json
{
  "hooks": {
    "pre-commit": "npm run security-check"
  }
}
```

`package.json` scripts:
```json
{
  "scripts": {
    "security-check": "npm audit && eslint . --ext .js,.jsx",
    "test": "npm run security-check && npm run test:unit"
  }
}
```

### ESLint Security Configuration

`.eslintrc.json`:
```json
{
  "extends": ["eslint:recommended"],
  "plugins": ["security"],
  "rules": {
    "security/detect-eval-with-expression": "error",
    "security/detect-non-literal-require": "error",
    "security/detect-non-literal-regexp": "error",
    "security/detect-unsafe-regex": "error",
    "security/detect-buffer-noassert": "error",
    "security/detect-child-process": "error",
    "security/detect-disable-mustache-escape": "error",
    "security/detect-no-csrf-before-method-override": "error",
    "security/detect-non-literal-fs-filename": "error",
    "security/detect-object-injection": "warn",
    "security/detect-possible-timing-attacks": "warn",
    "security/detect-pseudoRandomBytes": "error",
    "no-eval": "error",
    "no-new-func": "error"
  }
}
```

### Dependency Scanning

```bash
# Regular npm audit
npm audit

# Use additional tools
npm install -g snyk
snyk test

# Or use GitHub's Dependabot
# Add .github/dependabot.yml
```

`.github/dependabot.yml`:
```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
```

---

## Educational Use

### Safe Demonstration Setup

For teaching about malicious code:

1. **Create documentation** (like these files) instead of live demos
2. **Use code snippets** rather than full working exploits
3. **Sanitize first** before sharing with students
4. **Use screenshots** of the malicious behavior
5. **Explain, don't execute** the attack

### Sample Lesson Plan

**Objective**: Teach students to identify malicious patterns

**Materials**:
- This repository (read-only)
- Documentation (SECURITY_ANALYSIS.md, etc.)
- Isolated VM environment (optional)

**Activities**:
1. Code review exercise - find the malicious code
2. Pattern recognition - identify red flags
3. Decoding exercise - decode base64 strings
4. Flow analysis - trace execution path
5. Mitigation exercise - remove malicious code

**Learning Outcomes**:
- Identify IIFE patterns
- Recognize obfuscation techniques
- Understand remote code execution risks
- Learn secure coding practices

---

## Reporting Similar Attacks

If you find similar malicious code:

### 1. Do Not Execute
- Don't run the code
- Don't install dependencies if possible

### 2. Document Everything
```bash
# Create evidence package
mkdir evidence
cp -r suspicious-repo evidence/
date > evidence/discovery-time.txt
git log > evidence/git-history.txt
```

### 3. Report to Appropriate Parties

**GitHub Repository:**
```
https://github.com/contact/report-abuse
```

**npm Package:**
```
https://www.npmjs.com/support
Select: "I want to report a package"
```

**National Cybersecurity:**
```
US: https://www.cisa.gov/report
UK: https://www.ncsc.gov.uk/report-an-incident
EU: https://www.enisa.europa.eu/
```

**CVE Request:**
```
https://cveform.mitre.org/
```

### 4. Notify Affected Users

If you manage a system that ran the malicious code:
- Notify security team immediately
- Initiate incident response
- Assess scope of compromise
- Rotate credentials
- Monitor for data exfiltration

---

## Safe Alternatives for Similar Functionality

If you want to build a legitimate streaming/e-commerce site:

### Legitimate Email Sending

```javascript
// Good: Actual email sending (sendEmail.js pattern)
const sgMail = require('@sendgrid/mail');
sgMail.setApiKey(process.env.SENDGRID_API_KEY);

const sendEmail = async (options) => {
    const msg = {
        to: options.email,
        from: process.env.SENDGRID_MAIL,
        templateId: options.templateId,
        dynamic_template_data: options.data,
    };
    
    try {
        await sgMail.send(msg);
        console.log('Email sent successfully');
    } catch (error) {
        console.error('Email error:', error);
    }
};

module.exports = sendEmail;
```

### Configuration Management

```javascript
// Good: Proper configuration
// config/email.js
module.exports = {
    provider: 'sendgrid',
    apiKey: process.env.SENDGRID_API_KEY,
    fromEmail: process.env.FROM_EMAIL,
    templates: {
        welcome: process.env.WELCOME_TEMPLATE_ID,
        resetPassword: process.env.RESET_PASSWORD_TEMPLATE_ID
    }
};

// Bad: Obfuscated data in JSON with base64
```

### Proper Module Loading

```javascript
// Good: Export and import pattern
// email-service.js
class EmailService {
    async send(options) {
        // Implementation
    }
}

module.exports = new EmailService();

// app.js
const emailService = require('./email-service');
// Call when needed:
await emailService.send({ to: 'user@example.com' });

// Bad: IIFE that executes on load
const malicious = (async () => { 
    // Executes immediately
})();
```

---

## Security Checklist for Code Review

Use this checklist when reviewing any new code:

### Automatic Red Flags (Reject Immediately)
- [ ] Contains `eval()`
- [ ] Contains `Function.constructor` or `new Function()`
- [ ] IIFE pattern in utility/config files: `(function() { ... })()`
- [ ] Base64 encoded URLs or code: `atob()`, `btoa()`
- [ ] Obfuscated code (unreadable variable names, encoded strings)
- [ ] Warnings in README about not running the code

### Suspicious Patterns (Investigate Further)
- [ ] External network requests in unexpected files
- [ ] Requires that are never used (side-effect only)
- [ ] Dynamic requires: `require(variableName)`
- [ ] File system operations in frontend code
- [ ] Environment variable access in frontend code
- [ ] Child process spawning: `exec()`, `spawn()`
- [ ] Unusual npm scripts in package.json

### Best Practices (Should Be Present)
- [ ] Clear, readable code
- [ ] Proper comments explaining complex logic
- [ ] Environment variables in .env (not hardcoded)
- [ ] Input validation and sanitization
- [ ] Error handling
- [ ] Security headers configured
- [ ] Dependencies are up to date
- [ ] No secrets committed to git

---

## Tools for Security Analysis

### Static Analysis
```bash
# ESLint with security plugin
npm install -g eslint eslint-plugin-security
eslint . --ext .js,.jsx

# SonarQube
# https://www.sonarqube.org/

# npm audit
npm audit

# Snyk
npm install -g snyk
snyk test
```

### Dynamic Analysis
```bash
# Wireshark for network monitoring
wireshark

# tcpdump for packet capture
tcpdump -i any -w capture.pcap

# strace for system call tracing (Linux)
strace -f node server/app.js

# Process Monitor (Windows)
# https://docs.microsoft.com/en-us/sysinternals/downloads/procmon
```

### Sandboxing Tools
- Docker (--network none)
- Firejail (Linux)
- Windows Sandbox
- Virtual Machines (VirtualBox, VMware)
- Cloud sandboxes (AWS, Azure isolated VMs)

---

## Conclusion

This malicious repository serves as an excellent educational example of:
- Obfuscation techniques
- Remote code execution attacks
- Supply chain vulnerabilities
- The importance of code review

However, it should NEVER be executed on a production or personal system.

Use the safe analysis methods described here, and always maintain security best practices when reviewing unknown code.

---

**Remember**: 
- Trust but verify
- Code review before execution  
- When in doubt, don't run it
- Use isolated environments for testing
- Keep security tools updated
- Report malicious code to appropriate authorities

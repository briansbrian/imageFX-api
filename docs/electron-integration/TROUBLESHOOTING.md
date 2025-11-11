# Troubleshooting Guide

Common issues and solutions for ImageFX Electron integration.

## Table of Contents

- [Authentication Issues](#authentication-issues)
- [Image Generation Problems](#image-generation-problems)
- [Session Management](#session-management)
- [Performance Issues](#performance-issues)
- [Build and Installation](#build-and-installation)
- [Platform-Specific Issues](#platform-specific-issues)

## Authentication Issues

### "Cookie is required and cannot be empty"

**Symptoms**: Error thrown when initializing ImageFX

**Causes**:
- Cookie not extracted properly
- User not logged into Google
- Wrong domain for cookie extraction

**Solutions**:

1. **Verify you're logged into Google**
   ```javascript
   // Check if cookies exist
   const cookies = await session.defaultSession.cookies.get({
       domain: '.google.com'
   });
   console.log('Found cookies:', cookies.length);
   ```

2. **Check cookie format**
   ```javascript
   // Cookie should be: "name=value; name2=value2; ..."
   console.log('Cookie format:', cookie.substring(0, 50));
   ```

3. **Increase wait time before extraction**
   ```javascript
   // In did-navigate handler
   setTimeout(async () => {
       const cookies = await extractCookies();
       // ...
   }, 3000); // Increase from 2000 to 3000ms
   ```

### "Authentication failed (401)"

**Symptoms**: API calls return 401 Unauthorized

**Causes**:
- Cookies expired
- Invalid session
- Wrong Google account

**Solutions**:

1. **Re-authenticate**
   ```javascript
   ipcMain.handle('imagefx:reauth', async () => {
       // Clear old cookies
       await session.defaultSession.clearStorageData();
       // Start fresh authentication
       return await authenticate();
   });
   ```

2. **Check cookie expiry**
   ```javascript
   const cookies = await session.defaultSession.cookies.get({
       domain: '.google.com'
   });
   
   cookies.forEach(cookie => {
       if (cookie.expirationDate) {
           const expiry = new Date(cookie.expirationDate * 1000);
           console.log(`${cookie.name} expires:`, expiry);
       }
   });
   ```

3. **Verify account access**
   - Open https://labs.google/fx/tools/image-fx in browser
   - Ensure you can access ImageFX
   - Some accounts may have restricted access

### Authentication Window Doesn't Close

**Symptoms**: Auth window stays open after login

**Causes**:
- Navigation detection not working
- Redirect happening too quickly
- URL doesn't match expected pattern

**Solutions**:

1. **Debug navigation events**
   ```javascript
   authWindow.webContents.on('did-navigate', (event, url) => {
       console.log('Navigated to:', url);
       // Check if URL matches
   });
   ```

2. **Use will-navigate instead**
   ```javascript
   authWindow.webContents.on('will-navigate', (event, url) => {
       if (url.includes('labs.google/fx')) {
           event.preventDefault();
           // Extract cookies and close
       }
   });
   ```

3. **Add manual close button**
   ```javascript
   // Inject close button after auth
   authWindow.webContents.executeJavaScript(`
       const btn = document.createElement('button');
       btn.textContent = 'Authentication Complete';
       btn.onclick = () => window.close();
       document.body.prepend(btn);
   `);
   ```

## Image Generation Problems

### "Server responded with invalid response (500)"

**Symptoms**: Generation fails with 500 error

**Causes**:
- Session expired
- Rate limiting
- Invalid prompt
- Server-side issue

**Solutions**:

1. **Implement retry logic**
   ```javascript
   async function generateWithRetry(prompt, maxRetries = 3) {
       for (let i = 0; i < maxRetries; i++) {
           try {
               return await imageFX.generateImage(prompt);
           } catch (error) {
               if (i === maxRetries - 1) throw error;
               await delay(2000 * (i + 1)); // Exponential backoff
           }
       }
   }
   ```

2. **Refresh session before generation**
   ```javascript
   async function generateImage(prompt) {
       // Refresh if needed
       if (isSessionExpired()) {
           await refreshSession();
       }
       return await imageFX.generateImage(prompt);
   }
   ```

3. **Validate prompt**
   ```javascript
   function validatePrompt(prompt) {
       if (!prompt || prompt.trim().length === 0) {
           throw new Error('Prompt cannot be empty');
       }
       if (prompt.length > 2000) {
           throw new Error('Prompt too long (max 2000 chars)');
       }
       return true;
   }
   ```

### Images Not Saving

**Symptoms**: Generation succeeds but images not saved

**Causes**:
- Invalid save path
- Permission issues
- Disk full

**Solutions**:

1. **Create directory if missing**
   ```javascript
   const fs = require('fs');
   const path = require('path');
   
   function ensureDir(dirPath) {
       if (!fs.existsSync(dirPath)) {
           fs.mkdirSync(dirPath, { recursive: true });
       }
   }
   
   const savePath = path.join(app.getPath('userData'), 'images');
   ensureDir(savePath);
   ```

2. **Check permissions**
   ```javascript
   const fs = require('fs');
   
   try {
       fs.accessSync(savePath, fs.constants.W_OK);
       console.log('Directory is writable');
   } catch (error) {
       console.error('No write permission:', error);
   }
   ```

3. **Handle save errors**
   ```javascript
   try {
       const savedPath = image.save(savePath);
       console.log('Saved to:', savedPath);
   } catch (error) {
       console.error('Save failed:', error);
       // Fallback to temp directory
       const tempPath = app.getPath('temp');
       return image.save(tempPath);
   }
   ```

### "No images in response"

**Symptoms**: API call succeeds but no images returned

**Causes**:
- Prompt rejected by content filter
- Invalid generation parameters
- API response format changed

**Solutions**:

1. **Check response structure**
   ```javascript
   const response = await fetch(url, options);
   const data = await response.json();
   console.log('Full response:', JSON.stringify(data, null, 2));
   ```

2. **Validate prompt content**
   ```javascript
   // Avoid prompts that might be filtered
   const bannedWords = ['violent', 'explicit', /* etc */];
   function isPromptSafe(prompt) {
       return !bannedWords.some(word => 
           prompt.toLowerCase().includes(word)
       );
   }
   ```

3. **Use default parameters**
   ```javascript
   // Start with simple params
   const images = await imageFX.generateImage('test prompt', {
       // Don't specify model, aspectRatio, etc initially
   });
   ```

## Session Management

### Session Expires Quickly

**Symptoms**: Need to re-authenticate frequently

**Causes**:
- Short session duration
- Not storing session properly
- Clock skew issues

**Solutions**:

1. **Implement session persistence**
   ```javascript
   const Store = require('electron-store');
   const store = new Store();
   
   // Save session
   store.set('session', {
       cookie: cookieString,
       timestamp: Date.now(),
       expiresIn: 7 * 24 * 60 * 60 * 1000 // 7 days
   });
   
   // Load session
   const session = store.get('session');
   if (session && Date.now() - session.timestamp < session.expiresIn) {
       imageFX = new ImageFX(session.cookie);
   }
   ```

2. **Auto-refresh before expiry**
   ```javascript
   function scheduleRefresh(expiresAt) {
       const refreshTime = expiresAt - Date.now() - (5 * 60 * 1000);
       setTimeout(async () => {
           await refreshSession();
       }, refreshTime);
   }
   ```

3. **Sync system time**
   ```bash
   # On Linux/Mac
   sudo ntpdate -s time.nist.gov
   
   # On Windows (as Admin)
   w32tm /resync
   ```

### Cannot Decrypt Stored Session

**Symptoms**: Error loading encrypted session data

**Causes**:
- Changed encryption key
- Corrupted data
- Different encryption algorithm version

**Solutions**:

1. **Version your encryption**
   ```javascript
   const sessionData = {
       version: 1,
       encrypted: encryptedData,
       // ...
   };
   
   function decrypt(data) {
       if (data.version === 1) {
           return decryptV1(data.encrypted);
       }
       // Handle other versions
   }
   ```

2. **Graceful fallback**
   ```javascript
   try {
       const session = loadEncryptedSession();
   } catch (error) {
       console.error('Failed to load session:', error);
       // Clear corrupted data
       store.delete('session');
       // Require re-authentication
       await authenticate();
   }
   ```

3. **Backup before encryption changes**
   ```javascript
   function migrateEncryption() {
       const oldData = store.get('session');
       store.set('session_backup', oldData);
       // Apply new encryption
   }
   ```

## Performance Issues

### Slow Image Generation

**Symptoms**: Takes a long time to generate images

**Causes**:
- Network latency
- Server load
- Large image size
- Queue backlog

**Solutions**:

1. **Show progress indicators**
   ```javascript
   async function generateWithProgress(prompt, onProgress) {
       onProgress('Authenticating...');
       await ensureAuthenticated();
       
       onProgress('Generating image...');
       const images = await imageFX.generateImage(prompt);
       
       onProgress('Saving...');
       return images.map(img => img.save(savePath));
   }
   ```

2. **Parallel generation**
   ```javascript
   async function generateMultiple(prompts) {
       return await Promise.all(
           prompts.map(prompt => imageFX.generateImage(prompt))
       );
   }
   ```

3. **Implement caching**
   ```javascript
   const cache = new Map();
   
   async function getCachedOrGenerate(prompt) {
       const cacheKey = hashPrompt(prompt);
       if (cache.has(cacheKey)) {
           return cache.get(cacheKey);
       }
       
       const result = await imageFX.generateImage(prompt);
       cache.set(cacheKey, result);
       return result;
   }
   ```

### High Memory Usage

**Symptoms**: App uses excessive RAM

**Causes**:
- Images not garbage collected
- Large image cache
- Memory leaks in session management

**Solutions**:

1. **Clear image cache periodically**
   ```javascript
   let imageCache = [];
   const MAX_CACHE_SIZE = 50;
   
   function addToCache(image) {
       imageCache.push(image);
       if (imageCache.length > MAX_CACHE_SIZE) {
           imageCache.shift(); // Remove oldest
       }
   }
   ```

2. **Use WeakMap for caching**
   ```javascript
   const cache = new WeakMap();
   // Objects will be garbage collected when no longer referenced
   ```

3. **Monitor memory usage**
   ```javascript
   const { app } = require('electron');
   
   setInterval(() => {
       const metrics = app.getAppMetrics();
       console.log('Memory usage:', 
           metrics.reduce((sum, m) => sum + m.memory.workingSetSize, 0)
       );
   }, 60000);
   ```

## Build and Installation

### "Cannot find module '@rohitaryal/imagefx-api'"

**Symptoms**: Import error when running app

**Solutions**:

1. **Install dependencies**
   ```bash
   npm install @rohitaryal/imagefx-api
   ```

2. **Clear cache and reinstall**
   ```bash
   rm -rf node_modules package-lock.json
   npm install
   ```

3. **Check package.json**
   ```json
   {
     "dependencies": {
       "@rohitaryal/imagefx-api": "^2.1.1"
     }
   }
   ```

### TypeScript Compilation Errors

**Symptoms**: Build fails with type errors

**Solutions**:

1. **Install type definitions**
   ```bash
   npm install --save-dev @types/node @types/electron
   ```

2. **Configure tsconfig.json**
   ```json
   {
     "compilerOptions": {
       "module": "commonjs",
       "target": "es2020",
       "lib": ["es2020"],
       "moduleResolution": "node",
       "esModuleInterop": true
     }
   }
   ```

3. **Use type assertions**
   ```typescript
   const imageFX = new ImageFX(cookie as string);
   ```

## Platform-Specific Issues

### Windows: "EACCES: permission denied"

**Solutions**:

1. Run as administrator (temporary)
2. Change save directory:
   ```javascript
   const savePath = path.join(app.getPath('documents'), 'ImageFX');
   ```

3. Check antivirus settings

### macOS: App Not Opening

**Solutions**:

1. Right-click app → Open
2. System Preferences → Security → "Open Anyway"
3. Sign your app:
   ```bash
   codesign --deep --force --verbose --sign - YourApp.app
   ```

### Linux: Missing Dependencies

**Solutions**:

```bash
# Ubuntu/Debian
sudo apt-get install libgtk-3-0 libnotify4 libnss3 libxss1 libxtst6 xdg-utils

# Fedora
sudo dnf install gtk3 libnotify nss libXScrnSaver libXtst xdg-utils

# Arch
sudo pacman -S gtk3 libnotify nss libxss libxtst xdg-utils
```

## Getting More Help

### Enable Debug Logging

```javascript
// In main process
app.commandLine.appendSwitch('enable-logging');
app.commandLine.appendSwitch('log-level', '0');

// In renderer
console.log('Debug info:', { /* your data */ });
```

### Create Minimal Reproduction

```javascript
// Simplest possible test case
const { ImageFX } = require('@rohitaryal/imagefx-api');

async function test() {
    const imageFX = new ImageFX(process.env.GOOGLE_COOKIE);
    const images = await imageFX.generateImage('test');
    console.log('Success:', images.length);
}

test().catch(console.error);
```

### Report Issues

When reporting issues, include:

1. **Environment**:
   - OS and version
   - Electron version
   - imagefx-api version
   - Node.js version

2. **Error details**:
   - Full error message
   - Stack trace
   - Steps to reproduce

3. **Code sample**:
   - Minimal reproduction
   - Relevant configuration

## Resources

- [Main Documentation](../ELECTRON_INTEGRATION_GUIDE.md)
- [GitHub Issues](https://github.com/rohitaryal/imageFX-api/issues)
- [Electron Documentation](https://www.electronjs.org/docs)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/electron)

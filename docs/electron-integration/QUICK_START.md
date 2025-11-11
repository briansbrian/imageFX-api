# Quick Start Guide: ImageFX in Electron

Get ImageFX running in your Electron app in **15 minutes**.

**⚠️ Important**: This guide uses the imagefx-api package as a **reference implementation** to help you understand how authentication and API communication work. For production apps, you should:
1. Research current ImageFX API endpoints using browser DevTools
2. Use Context7, MCP servers, or similar tools for up-to-date API documentation
3. Implement your own API communication layer based on current best practices
4. Monitor for API changes and update accordingly

## Step 1: Install Dependencies (2 min)

```bash
npm install electron electron-store
# Optional: npm install @rohitaryal/imagefx-api  (for reference only)
```

## Step 2: Choose Your Method (1 min)

| Method | Best For | Setup Time |
|--------|----------|------------|
| [Hidden Browser](#method-1-hidden-browser) | Quick start, MVP | 10 min |
| [Cookie Import](#method-3-cookie-import) | Development/Testing | 5 min |
| [OAuth2](#method-2-oauth2) | Production apps | 2 hours |
| [Persistent Session](#method-4-persistent-session) | Best UX | 30 min |

## Method 1: Hidden Browser (Recommended for Quick Start)

### Create Auth Manager (`src/auth.js`)

```javascript
const { BrowserWindow, session } = require('electron');

class GoogleAuth {
    async authenticate() {
        return new Promise((resolve, reject) => {
            const authWindow = new BrowserWindow({
                width: 800,
                height: 600,
                show: false,
                webPreferences: {
                    nodeIntegration: false,
                    contextIsolation: true
                }
            });

            authWindow.once('ready-to-show', () => authWindow.show());
            authWindow.loadURL('https://labs.google/fx/tools/image-fx');

            authWindow.webContents.on('did-navigate', async (event, url) => {
                if (url.includes('labs.google/fx/tools/image-fx')) {
                    setTimeout(async () => {
                        const cookies = await session.defaultSession.cookies.get({
                            domain: '.google.com'
                        });
                        
                        const cookieString = cookies
                            .map(c => `${c.name}=${c.value}`)
                            .join('; ');
                        
                        authWindow.close();
                        resolve(cookieString);
                    }, 2000);
                }
            });

            authWindow.on('closed', () => {
                reject(new Error('Auth cancelled'));
            });
        });
    }
}

module.exports = { GoogleAuth };
```

### Update Main Process (`main.js`)

**Note**: This example shows the basic structure. Implement your own ImageFX API communication based on current endpoints.

```javascript
const { app, BrowserWindow, ipcMain } = require('electron');
const { GoogleAuth } = require('./auth');
const path = require('path');

let mainWindow;
let cookies = '';
let token = '';
const auth = new GoogleAuth();

function createWindow() {
    mainWindow = new BrowserWindow({
        width: 1200,
        height: 800,
        webPreferences: {
            preload: path.join(__dirname, 'preload.js'),
            contextIsolation: true,
            nodeIntegration: false
        }
    });

    mainWindow.loadFile('index.html');
}

// IPC Handlers
ipcMain.handle('imagefx:login', async () => {
    try {
        cookies = await auth.authenticate();
        
        // Get session token
        const response = await fetch('https://labs.google/fx/api/auth/session', {
            headers: { 'Cookie': cookies }
        });
        const sessionData = await response.json();
        token = sessionData.access_token;
        
        return { success: true };
    } catch (error) {
        return { success: false, error: error.message };
    }
});

ipcMain.handle('imagefx:generate', async (event, prompt) => {
    try {
        // Call ImageFX API - research current endpoint structure
        const response = await fetch('https://aisandbox-pa.googleapis.com/v1:runImageFx', {
            method: 'POST',
            headers: {
                'Cookie': cookies,
                'Authorization': `Bearer ${token}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ prompt, numImages: 1 })
        });
        
        const data = await response.json();
        const images = data.imagePanels?.[0]?.generatedImages || [];
        
        // Save images
        const fs = require('fs');
        const savedPaths = images.map((img, i) => {
            const savePath = path.join(app.getPath('userData'), 'images', `${Date.now()}_${i}.png`);
            fs.mkdirSync(path.dirname(savePath), { recursive: true });
            fs.writeFileSync(savePath, Buffer.from(img.image?.imageBytes, 'base64'));
            return savePath;
        });
        
        return { success: true, paths: savedPaths };
    } catch (error) {
        return { success: false, error: error.message };
    }
});

app.whenReady().then(createWindow);
```

### Create Preload Script (`preload.js`)

```javascript
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('imageFX', {
    login: () => ipcRenderer.invoke('imagefx:login'),
    generate: (prompt) => ipcRenderer.invoke('imagefx:generate', prompt)
});
```

### Create UI (`index.html`)

```html
<!DOCTYPE html>
<html>
<head>
    <title>ImageFX Integration</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; }
        button { padding: 10px 20px; margin: 10px 0; }
        #result { margin-top: 20px; }
        img { max-width: 500px; }
    </style>
</head>
<body>
    <h1>ImageFX Integration</h1>
    
    <button id="loginBtn">Login with Google</button>
    
    <div id="generateSection" style="display: none;">
        <input type="text" id="prompt" placeholder="Enter prompt..." style="width: 400px; padding: 8px;">
        <button id="generateBtn">Generate Image</button>
    </div>
    
    <div id="result"></div>

    <script>
        const loginBtn = document.getElementById('loginBtn');
        const generateSection = document.getElementById('generateSection');
        const generateBtn = document.getElementById('generateBtn');
        const promptInput = document.getElementById('prompt');
        const result = document.getElementById('result');

        loginBtn.addEventListener('click', async () => {
            loginBtn.textContent = 'Logging in...';
            loginBtn.disabled = true;
            
            const response = await window.imageFX.login();
            
            if (response.success) {
                loginBtn.style.display = 'none';
                generateSection.style.display = 'block';
                result.innerHTML = '<p style="color: green;">✓ Logged in successfully!</p>';
            } else {
                loginBtn.textContent = 'Login with Google';
                loginBtn.disabled = false;
                result.innerHTML = `<p style="color: red;">Error: ${response.error}</p>`;
            }
        });

        generateBtn.addEventListener('click', async () => {
            const prompt = promptInput.value.trim();
            if (!prompt) {
                alert('Please enter a prompt');
                return;
            }

            generateBtn.textContent = 'Generating...';
            generateBtn.disabled = true;
            result.innerHTML = '<p>Generating image...</p>';

            const response = await window.imageFX.generate(prompt);

            if (response.success) {
                result.innerHTML = `
                    <p style="color: green;">✓ Image generated!</p>
                    <img src="file://${response.paths[0]}" alt="Generated">
                    <p>Saved to: ${response.paths[0]}</p>
                `;
            } else {
                result.innerHTML = `<p style="color: red;">Error: ${response.error}</p>`;
            }

            generateBtn.textContent = 'Generate Image';
            generateBtn.disabled = false;
        });
    </script>
</body>
</html>
```

## Step 3: Run Your App (1 min)

```bash
# Start Electron
npm start
# or
electron .
```

## Method 3: Cookie Import (Fastest for Testing)

### Main Process (`main.js`)

```javascript
const { app, ipcMain, clipboard } = require('electron');
const { ImageFX } = require('@rohitaryal/imagefx-api');
const Store = require('electron-store');

const store = new Store();
let imageFX;

ipcMain.handle('imagefx:importCookie', async () => {
    try {
        const cookie = clipboard.readText();
        store.set('cookie', cookie);
        imageFX = new ImageFX(cookie);
        return { success: true };
    } catch (error) {
        return { success: false, error: error.message };
    }
});

ipcMain.handle('imagefx:generate', async (event, prompt) => {
    if (!imageFX) {
        const cookie = store.get('cookie');
        if (!cookie) {
            return { success: false, error: 'Not authenticated' };
        }
        imageFX = new ImageFX(cookie);
    }

    try {
        const images = await imageFX.generateImage(prompt);
        const saved = images.map(img => img.save('./generated'));
        return { success: true, paths: saved };
    } catch (error) {
        return { success: false, error: error.message };
    }
});
```

### Get Cookie Instructions

1. Install [Cookie Editor](https://chrome.google.com/webstore/detail/cookie-editor/hlkenndednhfkekhgcdicdfddnkalmdm) in Chrome
2. Open https://labs.google/fx/tools/image-fx
3. Login to Google
4. Click Cookie Editor icon
5. Click "Export" → "Header String"
6. Cookie is now in clipboard!
7. In your app, click "Import Cookie"

## Testing Your Integration

```javascript
// Test in DevTools console
await window.imageFX.login();
const result = await window.imageFX.generate('A beautiful sunset');
console.log(result);
```

## Common Issues & Fixes

### Issue: "Cookie is required"
**Fix**: Make sure you're logged into Google before extracting cookies

### Issue: Authentication window doesn't close
**Fix**: Increase timeout in `did-navigate` handler to 3000ms

### Issue: "Cannot find module"
**Fix**: Run `npm install` and rebuild with `npm run build`

## Next Steps

Once basic integration works:

1. **Add Session Persistence** - See [Method 4](./method-4-persistent-session.md)
2. **Implement Error Handling** - Retry logic, session refresh
3. **Add Loading States** - Progress indicators, status updates
4. **Optimize Performance** - Caching, request queuing
5. **Security Hardening** - Encrypt stored credentials

## StoryFramer Integration Example

```javascript
// Generate frames for scene nodes
async function generateSceneFrames(sceneDescription) {
    // First frame
    const firstPrompt = `First frame: ${sceneDescription}, cinematic, establishing shot`;
    const firstResult = await window.imageFX.generate(firstPrompt);
    
    // Last frame
    const lastPrompt = `Last frame: ${sceneDescription}, cinematic, closing shot`;
    const lastResult = await window.imageFX.generate(lastPrompt);
    
    return {
        firstFrame: firstResult.paths[0],
        lastFrame: lastResult.paths[0]
    };
}

// Usage in scene node
const frames = await generateSceneFrames('A hero walking into the sunset');
node.setFirstFrame(frames.firstFrame);
node.setLastFrame(frames.lastFrame);
```

## Getting Help

- 📖 [Full Documentation](../ELECTRON_INTEGRATION_GUIDE.md)
- 🐛 [Report Issues](https://github.com/rohitaryal/imageFX-api/issues)
- 💬 [Ask Questions](https://github.com/rohitaryal/imageFX-api/discussions)

## Complete Example

Full working example with all features: [`/examples/electron/`](../../examples/electron/)

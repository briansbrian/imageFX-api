# Method 3: Cookie Import from External Browser

## Overview

This method allows users to import cookies from their external browser (Chrome, Firefox, Edge) where they're already logged into Google. This is the simplest approach for development and testing, or for power users who prefer manual control.

## Pros & Cons

### ✅ Advantages
- Extremely simple implementation
- No authentication flow in your app
- Works immediately if user is logged into Google
- Good for prototyping and testing
- Minimal code required

### ⚠️ Disadvantages
- Requires manual user action
- Less professional user experience
- Cookies expire and need re-import
- Not suitable for non-technical users
- Requires browser extension (Cookie Editor)

## Prerequisites

```bash
npm install electron
npm install electron-store  # For storing imported cookies
```

**⚠️ Note**: This repository serves as a reference for understanding how cookie-based authentication works. Implement your own API communication based on current ImageFX endpoints and best practices.

## User Setup

Users need to install [Cookie Editor](https://github.com/Moustachauve/cookie-editor) browser extension:
- [Chrome/Edge Extension](https://chrome.google.com/webstore/detail/cookie-editor/hlkenndednhfkekhgcdicdfddnkalmdm)
- [Firefox Extension](https://addons.mozilla.org/en-US/firefox/addon/cookie-editor/)

## Implementation

### Step 1: Create Cookie Import Manager

Create `src/auth/CookieImportManager.ts`:

```typescript
import Store from 'electron-store';
import { dialog } from 'electron';
import * as fs from 'fs';

interface CookieData {
    cookies: string;
    importedAt: number;
    expiresAt?: number;
}

interface StoreSchema {
    cookieData: CookieData;
}

export class CookieImportManager {
    private store: Store<StoreSchema>;

    constructor() {
        this.store = new Store<StoreSchema>({
            name: 'imported-cookies'
        });
    }

    /**
     * Import cookies from clipboard (user copied from Cookie Editor)
     */
    async importFromClipboard(cookieString: string): Promise<void> {
        if (!this.validateCookieString(cookieString)) {
            throw new Error('Invalid cookie format');
        }

        const cookieData: CookieData = {
            cookies: cookieString,
            importedAt: Date.now(),
            expiresAt: this.estimateCookieExpiry(cookieString)
        };

        this.store.set('cookieData', cookieData);
    }

    /**
     * Import cookies from a file
     */
    async importFromFile(): Promise<void> {
        const result = await dialog.showOpenDialog({
            title: 'Import Cookies',
            filters: [
                { name: 'Text Files', extensions: ['txt'] },
                { name: 'All Files', extensions: ['*'] }
            ],
            properties: ['openFile']
        });

        if (result.canceled || result.filePaths.length === 0) {
            throw new Error('No file selected');
        }

        const filePath = result.filePaths[0];
        const cookieString = fs.readFileSync(filePath, 'utf-8').trim();

        await this.importFromClipboard(cookieString);
    }

    /**
     * Get stored cookies
     */
    getCookies(): string | null {
        const cookieData = this.store.get('cookieData');
        
        if (!cookieData) {
            return null;
        }

        // Check if cookies are expired
        if (cookieData.expiresAt && cookieData.expiresAt < Date.now()) {
            console.log('Stored cookies have expired');
            return null;
        }

        return cookieData.cookies;
    }

    /**
     * Validate cookie string format
     */
    private validateCookieString(cookieString: string): boolean {
        // Basic validation - check for common cookie patterns
        const hasRequiredCookies = [
            'SID', 'HSID', 'SSID', '__Secure-1PSID', '__Secure-3PSID'
        ].some(name => cookieString.includes(name));

        return hasRequiredCookies && cookieString.includes('=');
    }

    /**
     * Estimate cookie expiry based on typical Google cookie lifetimes
     */
    private estimateCookieExpiry(cookieString: string): number {
        // Google session cookies typically last 14 days
        // We'll estimate 7 days to be safe
        return Date.now() + (7 * 24 * 60 * 60 * 1000);
    }

    /**
     * Check if we have valid stored cookies
     */
    hasValidCookies(): boolean {
        return this.getCookies() !== null;
    }

    /**
     * Clear stored cookies
     */
    clearCookies(): void {
        this.store.delete('cookieData');
    }

    /**
     * Get cookie import instructions
     */
    static getImportInstructions(): string {
        return `
How to Import Cookies:

1. Install Cookie Editor extension in your browser:
   - Chrome/Edge: https://chrome.google.com/webstore/detail/cookie-editor/hlkenndednhfkekhgcdicdfddnkalmdm
   - Firefox: https://addons.mozilla.org/en-US/firefox/addon/cookie-editor/

2. Open https://labs.google/fx/tools/image-fx in your browser

3. Make sure you're logged into your Google account

4. Click the Cookie Editor extension icon

5. Click "Export" → "Header String"

6. Paste the copied cookies into the app or save to a file

7. Click "Import Cookies" in the app

Note: Cookies typically expire after 7-14 days and will need to be re-imported.
        `.trim();
    }
}
```

### Step 2: Create ImageFX Service with Cookie Import

Create `src/services/ImageFXServiceImport.ts`:

```typescript
import { ImageFX } from '@rohitaryal/imagefx-api';
import { CookieImportManager } from '../auth/CookieImportManager';

export class ImageFXServiceImport {
    private cookieManager: CookieImportManager;
    private imageFX: ImageFX | null = null;

    constructor() {
        this.cookieManager = new CookieImportManager();
    }

    /**
     * Initialize with stored cookies
     */
    async initialize(): Promise<void> {
        const cookies = this.cookieManager.getCookies();
        
        if (!cookies) {
            throw new Error('No cookies available. Please import cookies first.');
        }

        this.imageFX = new ImageFX(cookies);
    }

    /**
     * Import cookies from clipboard
     */
    async importCookiesFromClipboard(cookieString: string): Promise<void> {
        await this.cookieManager.importFromClipboard(cookieString);
        await this.initialize();
    }

    /**
     * Import cookies from file
     */
    async importCookiesFromFile(): Promise<void> {
        await this.cookieManager.importFromFile();
        await this.initialize();
    }

    /**
     * Generate image
     */
    async generateImage(prompt: string): Promise<any[]> {
        if (!this.imageFX) {
            throw new Error('Service not initialized. Please import cookies first.');
        }

        try {
            return await this.imageFX.generateImage(prompt);
        } catch (error) {
            if (error instanceof Error && error.message.includes('Authentication')) {
                throw new Error('Cookies expired. Please import new cookies.');
            }
            throw error;
        }
    }

    /**
     * Check if service is ready
     */
    isReady(): boolean {
        return this.cookieManager.hasValidCookies() && this.imageFX !== null;
    }

    /**
     * Clear cookies
     */
    clearCookies(): void {
        this.cookieManager.clearCookies();
        this.imageFX = null;
    }

    /**
     * Get import instructions
     */
    getInstructions(): string {
        return CookieImportManager.getImportInstructions();
    }
}
```

### Step 3: Update Main Process

Update `main.ts`:

```typescript
import { app, BrowserWindow, ipcMain, clipboard } from 'electron';
import { ImageFXServiceImport } from './services/ImageFXServiceImport';
import path from 'path';

let mainWindow: BrowserWindow | null = null;
let imageFXService: ImageFXServiceImport;

async function createWindow() {
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

    imageFXService = new ImageFXServiceImport();

    // Try to initialize with stored cookies
    try {
        await imageFXService.initialize();
        console.log('ImageFX initialized with stored cookies');
    } catch (error) {
        console.log('No valid cookies found, user needs to import');
    }
}

// IPC Handlers
ipcMain.handle('imagefx:importFromClipboard', async () => {
    try {
        const cookieString = clipboard.readText();
        await imageFXService.importCookiesFromClipboard(cookieString);
        return { success: true };
    } catch (error) {
        return { 
            success: false, 
            error: error instanceof Error ? error.message : 'Unknown error' 
        };
    }
});

ipcMain.handle('imagefx:importFromFile', async () => {
    try {
        await imageFXService.importCookiesFromFile();
        return { success: true };
    } catch (error) {
        return { 
            success: false, 
            error: error instanceof Error ? error.message : 'Unknown error' 
        };
    }
});

ipcMain.handle('imagefx:getInstructions', () => {
    return imageFXService.getInstructions();
});

ipcMain.handle('imagefx:isReady', () => {
    return imageFXService.isReady();
});

ipcMain.handle('imagefx:generate', async (event, prompt: string) => {
    try {
        const images = await imageFXService.generateImage(prompt);
        const savedImages = images.map((image) => {
            const savePath = path.join(app.getPath('userData'), 'generated-images');
            return {
                path: image.save(savePath),
                mediaId: image.mediaID
            };
        });
        return { success: true, images: savedImages };
    } catch (error) {
        return { 
            success: false, 
            error: error instanceof Error ? error.message : 'Unknown error' 
        };
    }
});

ipcMain.handle('imagefx:clearCookies', () => {
    imageFXService.clearCookies();
    return { success: true };
});

app.whenReady().then(createWindow);
```

### Step 4: Update Preload Script

Update `preload.ts`:

```typescript
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('imageFX', {
    importFromClipboard: () => ipcRenderer.invoke('imagefx:importFromClipboard'),
    importFromFile: () => ipcRenderer.invoke('imagefx:importFromFile'),
    getInstructions: () => ipcRenderer.invoke('imagefx:getInstructions'),
    isReady: () => ipcRenderer.invoke('imagefx:isReady'),
    generate: (prompt: string) => ipcRenderer.invoke('imagefx:generate', prompt),
    clearCookies: () => ipcRenderer.invoke('imagefx:clearCookies')
});
```

### Step 5: Create UI for Cookie Import

Example HTML/React UI:

```html
<!DOCTYPE html>
<html>
<head>
    <title>ImageFX Cookie Import</title>
    <style>
        .cookie-import-container {
            padding: 20px;
            max-width: 600px;
            margin: 0 auto;
        }
        .instructions {
            background: #f5f5f5;
            padding: 15px;
            border-radius: 5px;
            white-space: pre-wrap;
            font-family: monospace;
            font-size: 12px;
        }
        .button-group {
            margin-top: 20px;
            display: flex;
            gap: 10px;
        }
        button {
            padding: 10px 20px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            background: #4285f4;
            color: white;
        }
        button:hover {
            background: #357ae8;
        }
        .status {
            margin-top: 20px;
            padding: 10px;
            border-radius: 5px;
        }
        .status.ready {
            background: #d4edda;
            color: #155724;
        }
        .status.error {
            background: #f8d7da;
            color: #721c24;
        }
    </style>
</head>
<body>
    <div class="cookie-import-container">
        <h1>ImageFX Setup</h1>
        
        <div id="status" class="status"></div>
        
        <h2>Instructions</h2>
        <div id="instructions" class="instructions"></div>
        
        <div class="button-group">
            <button id="import-clipboard">Import from Clipboard</button>
            <button id="import-file">Import from File</button>
            <button id="clear-cookies">Clear Cookies</button>
        </div>

        <div id="test-area" style="display: none; margin-top: 20px;">
            <h2>Test Image Generation</h2>
            <input type="text" id="test-prompt" placeholder="Enter prompt..." style="width: 100%; padding: 8px;">
            <button id="test-generate" style="margin-top: 10px;">Generate Test Image</button>
        </div>
    </div>

    <script>
        async function updateStatus() {
            const isReady = await window.imageFX.isReady();
            const statusDiv = document.getElementById('status');
            
            if (isReady) {
                statusDiv.className = 'status ready';
                statusDiv.textContent = '✓ Ready to generate images!';
                document.getElementById('test-area').style.display = 'block';
            } else {
                statusDiv.className = 'status error';
                statusDiv.textContent = '⚠ Please import cookies to continue';
                document.getElementById('test-area').style.display = 'none';
            }
        }

        async function loadInstructions() {
            const instructions = await window.imageFX.getInstructions();
            document.getElementById('instructions').textContent = instructions;
        }

        document.getElementById('import-clipboard').addEventListener('click', async () => {
            const result = await window.imageFX.importFromClipboard();
            if (result.success) {
                alert('Cookies imported successfully!');
            } else {
                alert('Failed to import: ' + result.error);
            }
            updateStatus();
        });

        document.getElementById('import-file').addEventListener('click', async () => {
            const result = await window.imageFX.importFromFile();
            if (result.success) {
                alert('Cookies imported successfully!');
            } else {
                alert('Failed to import: ' + result.error);
            }
            updateStatus();
        });

        document.getElementById('clear-cookies').addEventListener('click', async () => {
            const result = await window.imageFX.clearCookies();
            if (result.success) {
                alert('Cookies cleared!');
            }
            updateStatus();
        });

        document.getElementById('test-generate').addEventListener('click', async () => {
            const prompt = document.getElementById('test-prompt').value;
            if (!prompt) {
                alert('Please enter a prompt');
                return;
            }

            const button = document.getElementById('test-generate');
            button.textContent = 'Generating...';
            button.disabled = true;

            const result = await window.imageFX.generate(prompt);
            
            button.textContent = 'Generate Test Image';
            button.disabled = false;

            if (result.success) {
                alert('Image generated successfully! Saved to: ' + result.images[0].path);
            } else {
                alert('Failed to generate: ' + result.error);
            }
        });

        // Initialize
        loadInstructions();
        updateStatus();
    </script>
</body>
</html>
```

## React Component Example

```typescript
import React, { useState, useEffect } from 'react';

export function CookieImportSetup() {
    const [isReady, setIsReady] = useState(false);
    const [instructions, setInstructions] = useState('');
    const [status, setStatus] = useState('');

    useEffect(() => {
        loadInstructions();
        checkReadyStatus();
    }, []);

    async function loadInstructions() {
        const instr = await window.imageFX.getInstructions();
        setInstructions(instr);
    }

    async function checkReadyStatus() {
        const ready = await window.imageFX.isReady();
        setIsReady(ready);
    }

    async function handleImportClipboard() {
        const result = await window.imageFX.importFromClipboard();
        if (result.success) {
            setStatus('✓ Cookies imported successfully!');
            checkReadyStatus();
        } else {
            setStatus('⚠ Failed: ' + result.error);
        }
    }

    async function handleImportFile() {
        const result = await window.imageFX.importFromFile();
        if (result.success) {
            setStatus('✓ Cookies imported successfully!');
            checkReadyStatus();
        } else {
            setStatus('⚠ Failed: ' + result.error);
        }
    }

    return (
        <div style={{ padding: '20px' }}>
            <h1>ImageFX Setup</h1>
            
            {status && <div className={isReady ? 'status-ready' : 'status-error'}>{status}</div>}
            
            <h2>Instructions</h2>
            <pre style={{ background: '#f5f5f5', padding: '15px' }}>
                {instructions}
            </pre>
            
            <div style={{ display: 'flex', gap: '10px', marginTop: '20px' }}>
                <button onClick={handleImportClipboard}>Import from Clipboard</button>
                <button onClick={handleImportFile}>Import from File</button>
            </div>

            {isReady && (
                <div style={{ marginTop: '20px', color: 'green' }}>
                    ✓ Ready to generate images!
                </div>
            )}
        </div>
    );
}
```

## Advanced: Auto-Detection in Development

For development, you can auto-detect cookies from Chrome:

```typescript
import { execSync } from 'child_process';
import * as path from 'path';
import * as os from 'os';

class ChromeCookieReader {
    /**
     * Extract cookies from Chrome (Development only!)
     * Requires chrome-cookies-secure package
     */
    static async extractFromChrome(): Promise<string> {
        // This is for development only
        // In production, users should manually export
        
        const cookieDbPath = this.getChromeDbPath();
        
        // Use chrome-cookies-secure or similar library
        // npm install chrome-cookies-secure
        const chromeCookies = require('chrome-cookies-secure');
        
        return new Promise((resolve, reject) => {
            chromeCookies.getCookies('https://labs.google', 'header', (err: any, cookies: string) => {
                if (err) reject(err);
                else resolve(cookies);
            });
        });
    }

    private static getChromeDbPath(): string {
        const platform = os.platform();
        
        if (platform === 'darwin') {
            return path.join(os.homedir(), 'Library/Application Support/Google/Chrome/Default/Cookies');
        } else if (platform === 'win32') {
            return path.join(os.homedir(), 'AppData/Local/Google/Chrome/User Data/Default/Cookies');
        } else {
            return path.join(os.homedir(), '.config/google-chrome/Default/Cookies');
        }
    }
}
```

## Troubleshooting

### Issue: "Invalid cookie format"
**Solution**: Ensure you're copying the "Header String" format from Cookie Editor, not JSON format.

### Issue: Cookies expire quickly
**Solution**: Google cookies typically last 7-14 days. Users need to re-import periodically.

### Issue: Authentication fails after import
**Solution**: Make sure the user was logged into https://labs.google when exporting cookies, not just google.com.

## Best Practices

1. **Clear instructions**: Provide step-by-step guidance with screenshots
2. **Validate cookies**: Check format before storing
3. **Expiry notifications**: Warn users when cookies are about to expire
4. **Secure storage**: Store cookies securely even if they're user-imported
5. **Easy re-import**: Make it simple to update expired cookies

## Security Notes

- ⚠️ Store imported cookies securely
- ⚠️ Never log or display cookies in plain text
- ⚠️ Warn users about cookie security
- ⚠️ Clear cookies on app uninstall

## Next Steps

- For better UX, combine with [Method 1](./method-1-hidden-browser.md)
- For production, use [Method 2 (OAuth2)](./method-2-oauth2.md)
- Add automatic cookie expiry detection and re-import prompts

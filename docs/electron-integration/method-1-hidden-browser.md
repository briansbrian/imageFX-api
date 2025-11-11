# Method 1: Hidden BrowserWindow with Cookie Extraction

## Overview

This method uses Electron's `BrowserWindow` to handle Google authentication in the background. The user logs in once, and the app automatically extracts the necessary cookies for ImageFX API calls.

## Pros & Cons

### ✅ Advantages
- Simple implementation
- Uses Electron's built-in capabilities
- No external dependencies beyond Electron
- User sees familiar Google login
- Works with 2FA/MFA

### ⚠️ Disadvantages
- Requires login every app restart (unless combined with persistence)
- User must interact with auth window
- Cookies expire after session

## Prerequisites

```bash
npm install electron
```

**⚠️ Note**: This guide shows the core authentication and API communication approach. The `@rohitaryal/imagefx-api` package is used as a **reference** to understand how ImageFX API works. You should implement your own API communication layer based on current API endpoints and best practices, or use context7, MCP servers, and other modern tools to research and implement the most up-to-date integration methods.

## Implementation

### Step 1: Create Authentication Manager

Create `src/auth/GoogleAuthManager.ts`:

```typescript
import { BrowserWindow, session } from 'electron';

export class GoogleAuthManager {
    private authWindow: BrowserWindow | null = null;
    private cookies: string = '';

    /**
     * Opens a hidden browser window for Google authentication
     * and extracts cookies after successful login
     */
    async authenticate(): Promise<string> {
        return new Promise((resolve, reject) => {
            // Create a hidden window for authentication
            this.authWindow = new BrowserWindow({
                width: 800,
                height: 600,
                show: false, // Start hidden
                webPreferences: {
                    nodeIntegration: false,
                    contextIsolation: true,
                    sandbox: true
                }
            });

            // Show window when ready (for user to login)
            this.authWindow.once('ready-to-show', () => {
                this.authWindow?.show();
            });

            // Navigate to ImageFX
            this.authWindow.loadURL('https://labs.google/fx/tools/image-fx');

            // Monitor navigation to detect successful login
            this.authWindow.webContents.on('did-navigate', async (event, url) => {
                // Check if we're on the ImageFX page (indicating successful login)
                if (url.includes('labs.google/fx/tools/image-fx')) {
                    // Wait a bit for cookies to be set
                    setTimeout(async () => {
                        try {
                            const cookies = await this.extractCookies();
                            this.cookies = cookies;
                            this.authWindow?.close();
                            resolve(cookies);
                        } catch (error) {
                            reject(error);
                        }
                    }, 2000);
                }
            });

            // Handle window close
            this.authWindow.on('closed', () => {
                if (!this.cookies) {
                    reject(new Error('Authentication window closed before completion'));
                }
                this.authWindow = null;
            });

            // Handle load failures
            this.authWindow.webContents.on('did-fail-load', (event, errorCode, errorDescription) => {
                reject(new Error(`Failed to load auth page: ${errorDescription}`));
            });
        });
    }

    /**
     * Extracts all cookies from the Google domain
     */
    private async extractCookies(): Promise<string> {
        const cookies = await session.defaultSession.cookies.get({
            domain: '.google.com'
        });

        // Format cookies as header string
        return cookies
            .map(cookie => `${cookie.name}=${cookie.value}`)
            .join('; ');
    }

    /**
     * Check if we have valid cookies stored
     */
    hasCookies(): boolean {
        return this.cookies.length > 0;
    }

    /**
     * Get stored cookies
     */
    getCookies(): string {
        return this.cookies;
    }

    /**
     * Clear stored cookies
     */
    clearCookies() {
        this.cookies = '';
    }
}
```

### Step 2: Create ImageFX API Communication Layer

Create `src/services/ImageFXService.ts`:

**Note**: This implementation shows the API structure. You should research the current ImageFX API endpoints and implement based on up-to-date documentation.

```typescript
import { GoogleAuthManager } from '../auth/GoogleAuthManager';

interface GenerateImageOptions {
    count?: number;
    model?: 'IMAGEN_3' | 'IMAGEN_3_1' | 'IMAGEN_3_5';
    aspectRatio?: 'IMAGE_ASPECT_RATIO_SQUARE' | 'IMAGE_ASPECT_RATIO_PORTRAIT' | 'IMAGE_ASPECT_RATIO_LANDSCAPE';
    seed?: number;
}

export class ImageFXService {
    private authManager: GoogleAuthManager;
    private cookies: string = '';
    private token: string = '';

    constructor() {
        this.authManager = new GoogleAuthManager();
    }

    /**
     * Initialize the service with authentication
     */
    async initialize(): Promise<void> {
        if (!this.authManager.hasCookies()) {
            console.log('No cookies found, starting authentication...');
            this.cookies = await this.authManager.authenticate();
            console.log('Authentication successful!');
        } else {
            this.cookies = this.authManager.getCookies();
        }

        // Fetch session token from ImageFX API
        await this.refreshSession();
    }

    /**
     * Refresh session token
     */
    private async refreshSession(): Promise<void> {
        const response = await fetch('https://labs.google/fx/api/auth/session', {
            headers: {
                'Cookie': this.cookies,
                'Origin': 'https://labs.google',
                'Referer': 'https://labs.google/fx/tools/image-fx',
                'Content-Type': 'application/json'
            }
        });

        if (!response.ok) {
            throw new Error('Failed to refresh session');
        }

        const sessionData = await response.json();
        this.token = sessionData.access_token;
    }

    /**
     * Generate an image from a text prompt
     * 
     * Research current API endpoints and structure from:
     * - Browser DevTools Network tab
     * - Context7 for API documentation
     * - MCP servers for real-time API info
     */
    async generateImage(prompt: string, options?: GenerateImageOptions): Promise<Array<{
        imageData: string;
        mediaID: string;
        save: (path: string) => string;
    }>> {
        if (!this.token) {
            throw new Error('Service not initialized. Call initialize() first.');
        }

        // Construct request body based on current API structure
        const requestBody = {
            prompt: prompt,
            numImages: options?.count || 1,
            model: options?.model || 'IMAGEN_3_5',
            aspectRatio: options?.aspectRatio || 'IMAGE_ASPECT_RATIO_SQUARE',
            seed: options?.seed || Math.floor(Math.random() * 1000000)
        };

        try {
            const response = await fetch('https://aisandbox-pa.googleapis.com/v1:runImageFx', {
                method: 'POST',
                headers: {
                    'Cookie': this.cookies,
                    'Authorization': `Bearer ${this.token}`,
                    'Origin': 'https://labs.google',
                    'Referer': 'https://labs.google/fx/tools/image-fx',
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(requestBody)
            });

            if (!response.ok) {
                throw new Error(`API request failed: ${response.status}`);
            }

            const data = await response.json();
            
            // Parse response and create image objects
            // This structure may vary - research current API response format
            const images = data.imagePanels?.[0]?.generatedImages || [];
            
            return images.map((img: any) => ({
                imageData: img.image?.imageBytes || '',
                mediaID: img.image?.mediaKey || '',
                save: (savePath: string) => {
                    // Implement image saving logic
                    const fs = require('fs');
                    const path = require('path');
                    const filename = `${Date.now()}_${img.image?.mediaKey}.png`;
                    const fullPath = path.join(savePath, filename);
                    
                    // Decode base64 and save
                    const imageBuffer = Buffer.from(img.image?.imageBytes || '', 'base64');
                    fs.writeFileSync(fullPath, imageBuffer);
                    
                    return fullPath;
                }
            }));
        } catch (error) {
            // If authentication fails, try to re-authenticate
            if (error instanceof Error && error.message.includes('Authentication')) {
                console.log('Session expired, re-authenticating...');
                this.authManager.clearCookies();
                await this.initialize();
                return await this.generateImage(prompt, options);
            }
            throw error;
        }
    }

    /**
     * Check if service is ready to generate images
     */
    isReady(): boolean {
        return this.token.length > 0;
    }
}
```

**🔍 Research Tips**: 
- Use browser DevTools Network tab when accessing https://labs.google/fx/tools/image-fx
- Check request/response formats for the latest API structure
- Use Context7 or similar tools to get current API documentation
- Monitor for API changes and updates
```

### Step 3: Integrate with Main Process

Update your `main.ts` or `main.js`:

```typescript
import { app, BrowserWindow, ipcMain } from 'electron';
import { ImageFXService } from './services/ImageFXService';
import path from 'path';

let mainWindow: BrowserWindow | null = null;
let imageFXService: ImageFXService;

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

    // Initialize ImageFX service
    imageFXService = new ImageFXService();
    
    // You can initialize immediately or wait for user action
    // await imageFXService.initialize();
}

// IPC Handlers for renderer process
ipcMain.handle('imagefx:initialize', async () => {
    try {
        await imageFXService.initialize();
        return { success: true };
    } catch (error) {
        return { 
            success: false, 
            error: error instanceof Error ? error.message : 'Unknown error' 
        };
    }
});

ipcMain.handle('imagefx:generate', async (event, prompt: string, options?: any) => {
    try {
        const images = await imageFXService.generateImage(prompt, options);
        
        // Save images and return paths
        const savedImages = images.map((image, index) => {
            const savePath = path.join(app.getPath('userData'), 'generated-images');
            const savedFilePath = image.save(savePath);
            return {
                path: savedFilePath,
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

ipcMain.handle('imagefx:isReady', () => {
    return imageFXService.isReady();
});

app.whenReady().then(createWindow);

app.on('window-all-closed', () => {
    if (process.platform !== 'darwin') {
        app.quit();
    }
});

app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
        createWindow();
    }
});
```

### Step 4: Create Preload Script

Create `preload.ts`:

```typescript
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('imageFX', {
    initialize: () => ipcRenderer.invoke('imagefx:initialize'),
    generate: (prompt: string, options?: any) => 
        ipcRenderer.invoke('imagefx:generate', prompt, options),
    isReady: () => ipcRenderer.invoke('imagefx:isReady')
});
```

### Step 5: Use in Renderer Process

In your React/Vue/HTML app:

```typescript
// TypeScript definitions for the exposed API
declare global {
    interface Window {
        imageFX: {
            initialize: () => Promise<{ success: boolean; error?: string }>;
            generate: (prompt: string, options?: any) => Promise<{
                success: boolean;
                images?: Array<{ path: string; mediaId: string }>;
                error?: string;
            }>;
            isReady: () => Promise<boolean>;
        };
    }
}

// Example usage in your StoryFramer app
async function initializeImageFX() {
    const result = await window.imageFX.initialize();
    if (result.success) {
        console.log('ImageFX ready!');
    } else {
        console.error('Failed to initialize:', result.error);
    }
}

async function generateSceneImage(prompt: string) {
    const result = await window.imageFX.generate(prompt, {
        count: 1,
        aspectRatio: 'LANDSCAPE'
    });

    if (result.success && result.images) {
        // Use the generated image in your scene nodes
        const imagePath = result.images[0].path;
        console.log('Generated image:', imagePath);
        return imagePath;
    } else {
        console.error('Failed to generate:', result.error);
    }
}

// Initialize on app start
initializeImageFX();
```

## Usage Example

```typescript
// In your scene node component
class SceneNode {
    async generateFirstFrame(description: string) {
        const prompt = `A cinematic first frame of ${description}, high quality, detailed`;
        const imagePath = await generateSceneImage(prompt);
        
        // Set as first frame in your node
        this.firstFrame = imagePath;
    }

    async generateLastFrame(description: string) {
        const prompt = `A cinematic last frame of ${description}, high quality, detailed`;
        const imagePath = await generateSceneImage(prompt);
        
        // Set as last frame in your node
        this.lastFrame = imagePath;
    }
}
```

## Troubleshooting

### Issue: Authentication window doesn't close automatically
**Solution**: The URL detection might be too quick. Increase the timeout in the `did-navigate` handler.

### Issue: Cookies are empty
**Solution**: Ensure you're checking the correct domain (`.google.com`) and wait sufficient time after navigation.

### Issue: Session expires quickly
**Solution**: Combine this method with Method 4 (Persistent Session) to store cookies between app restarts.

### Issue: User has 2FA enabled
**Solution**: This method fully supports 2FA. The user will complete the 2FA flow in the auth window.

## Best Practices

1. **Show clear UI feedback** during authentication
2. **Cache cookies** for the session duration
3. **Handle errors gracefully** and allow re-authentication
4. **Test with different Google account types** (personal, workspace)
5. **Clear cookies on logout** to allow account switching

## Security Notes

- ⚠️ Store cookies in memory only (not on disk) for this basic implementation
- ⚠️ Use Electron's session partitions for better isolation
- ⚠️ Clear cookies when user logs out
- ⚠️ Consider encrypting cookies if you must store them (see Method 4)

## Next Steps

- For persistent authentication, see [Method 4: Persistent Session](./method-4-persistent-session.md)
- For production apps, consider [Method 2: OAuth2](./method-2-oauth2.md)
- Combine with session refresh logic for better UX

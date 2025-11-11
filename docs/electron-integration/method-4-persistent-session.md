# Method 4: Persistent Session with Auto-Refresh

## Overview

This method combines authentication (from Method 1 or 2) with intelligent session management. It stores encrypted credentials, automatically refreshes sessions before they expire, and provides the best user experience by requiring authentication only once per installation.

## Pros & Cons

### ✅ Advantages
- Best user experience (one-time setup)
- Automatic session refresh
- Encrypted credential storage
- Works offline after initial setup
- Seamless integration into workflow

### ⚠️ Disadvantages
- More complex implementation
- Requires secure encryption key management
- Must handle edge cases (account changes, revoked access)
- Larger codebase

## Prerequisites

```bash
npm install electron
npm install @rohitaryal/imagefx-api
npm install electron-store
npm install crypto-js  # For encryption
```

## Implementation

### Step 1: Create Secure Session Manager

Create `src/auth/SecureSessionManager.ts`:

```typescript
import Store from 'electron-store';
import CryptoJS from 'crypto-js';
import { app } from 'electron';

interface SessionData {
    cookies: string;
    token: string;
    tokenExpiry: number;
    userEmail?: string;
    lastRefresh: number;
    encryptionVersion: number;
}

interface StoreSchema {
    sessionData: string; // Encrypted session data
    sessionMetadata: {
        lastLogin: number;
        userEmail?: string;
    };
}

export class SecureSessionManager {
    private store: Store<StoreSchema>;
    private encryptionKey: string;
    private sessionData: SessionData | null = null;
    private refreshTimer: NodeJS.Timeout | null = null;

    // Refresh session 5 minutes before expiry
    private readonly REFRESH_BUFFER_MS = 5 * 60 * 1000;

    constructor() {
        this.store = new Store<StoreSchema>({
            name: 'secure-session'
        });

        // Generate or retrieve encryption key
        // In production, use a more secure key management solution
        this.encryptionKey = this.getEncryptionKey();
    }

    /**
     * Store session data securely
     */
    async storeSession(cookies: string, token: string, tokenExpiry: number, userEmail?: string): Promise<void> {
        const sessionData: SessionData = {
            cookies,
            token,
            tokenExpiry,
            userEmail,
            lastRefresh: Date.now(),
            encryptionVersion: 1
        };

        // Encrypt the session data
        const encrypted = this.encrypt(JSON.stringify(sessionData));
        
        this.store.set('sessionData', encrypted);
        this.store.set('sessionMetadata', {
            lastLogin: Date.now(),
            userEmail
        });

        this.sessionData = sessionData;

        // Schedule automatic refresh
        this.scheduleRefresh(tokenExpiry);
    }

    /**
     * Load stored session
     */
    async loadSession(): Promise<SessionData | null> {
        if (this.sessionData) {
            return this.sessionData;
        }

        const encrypted = this.store.get('sessionData');
        
        if (!encrypted) {
            return null;
        }

        try {
            const decrypted = this.decrypt(encrypted);
            this.sessionData = JSON.parse(decrypted);

            // Check if session is still valid
            if (this.sessionData && this.isSessionValid(this.sessionData)) {
                // Schedule refresh
                this.scheduleRefresh(this.sessionData.tokenExpiry);
                return this.sessionData;
            }

            // Session expired, clear it
            this.clearSession();
            return null;
        } catch (error) {
            console.error('Failed to load session:', error);
            this.clearSession();
            return null;
        }
    }

    /**
     * Check if session is still valid
     */
    isSessionValid(session: SessionData): boolean {
        return session.tokenExpiry > Date.now() + this.REFRESH_BUFFER_MS;
    }

    /**
     * Get current session cookies
     */
    getCookies(): string | null {
        return this.sessionData?.cookies || null;
    }

    /**
     * Get current session token
     */
    getToken(): string | null {
        return this.sessionData?.token || null;
    }

    /**
     * Update session with new token
     */
    async updateToken(token: string, tokenExpiry: number): Promise<void> {
        if (!this.sessionData) {
            throw new Error('No active session to update');
        }

        this.sessionData.token = token;
        this.sessionData.tokenExpiry = tokenExpiry;
        this.sessionData.lastRefresh = Date.now();

        // Re-encrypt and store
        const encrypted = this.encrypt(JSON.stringify(this.sessionData));
        this.store.set('sessionData', encrypted);

        // Reschedule refresh
        this.scheduleRefresh(tokenExpiry);
    }

    /**
     * Schedule automatic session refresh
     */
    private scheduleRefresh(tokenExpiry: number): void {
        // Clear existing timer
        if (this.refreshTimer) {
            clearTimeout(this.refreshTimer);
        }

        // Calculate when to refresh (before expiry)
        const refreshTime = tokenExpiry - Date.now() - this.REFRESH_BUFFER_MS;

        if (refreshTime > 0) {
            this.refreshTimer = setTimeout(() => {
                this.onRefreshNeeded();
            }, refreshTime);

            console.log(`Session refresh scheduled in ${Math.round(refreshTime / 1000 / 60)} minutes`);
        } else {
            // Already expired, trigger refresh immediately
            this.onRefreshNeeded();
        }
    }

    /**
     * Callback when session needs refresh
     * Override this in your implementation to trigger refresh
     */
    private onRefreshNeeded(): void {
        console.log('Session refresh needed');
        // This will be overridden by the service class
    }

    /**
     * Set refresh callback
     */
    setRefreshCallback(callback: () => Promise<void>): void {
        this.onRefreshNeeded = async () => {
            try {
                await callback();
            } catch (error) {
                console.error('Failed to refresh session:', error);
            }
        };
    }

    /**
     * Clear session data
     */
    clearSession(): void {
        this.store.delete('sessionData');
        this.store.delete('sessionMetadata');
        this.sessionData = null;

        if (this.refreshTimer) {
            clearTimeout(this.refreshTimer);
            this.refreshTimer = null;
        }
    }

    /**
     * Check if we have a stored session
     */
    hasStoredSession(): boolean {
        return this.store.has('sessionData');
    }

    /**
     * Get session metadata (non-sensitive info)
     */
    getMetadata() {
        return this.store.get('sessionMetadata');
    }

    /**
     * Encrypt data
     */
    private encrypt(data: string): string {
        return CryptoJS.AES.encrypt(data, this.encryptionKey).toString();
    }

    /**
     * Decrypt data
     */
    private decrypt(encrypted: string): string {
        const bytes = CryptoJS.AES.decrypt(encrypted, this.encryptionKey);
        return bytes.toString(CryptoJS.enc.Utf8);
    }

    /**
     * Get or generate encryption key
     * In production, use a more secure method (hardware-backed, OS keychain, etc.)
     */
    private getEncryptionKey(): string {
        // Use machine-specific identifier for encryption key
        const machineId = app.getName() + app.getVersion() + app.getPath('userData');
        return CryptoJS.SHA256(machineId).toString();
    }
}
```

### Step 2: Create Account Manager with Auto-Refresh

Create `src/auth/PersistentAccountManager.ts`:

```typescript
import { BrowserWindow } from 'electron';
import { SecureSessionManager } from './SecureSessionManager';

export class PersistentAccountManager {
    private sessionManager: SecureSessionManager;
    private authWindow: BrowserWindow | null = null;

    constructor() {
        this.sessionManager = new SecureSessionManager();

        // Set up auto-refresh callback
        this.sessionManager.setRefreshCallback(async () => {
            await this.refreshSession();
        });
    }

    /**
     * Initialize authentication
     * Uses stored session if available, otherwise starts auth flow
     */
    async initialize(): Promise<{ cookies: string; token: string }> {
        // Try to load stored session
        const storedSession = await this.sessionManager.loadSession();
        
        if (storedSession) {
            console.log('Using stored session');
            return {
                cookies: storedSession.cookies,
                token: storedSession.token
            };
        }

        // No valid session, start authentication
        console.log('No valid session, starting authentication');
        return await this.authenticate();
    }

    /**
     * Start authentication flow
     */
    private async authenticate(): Promise<{ cookies: string; token: string }> {
        return new Promise((resolve, reject) => {
            this.authWindow = new BrowserWindow({
                width: 800,
                height: 600,
                show: false,
                webPreferences: {
                    nodeIntegration: false,
                    contextIsolation: true,
                    sandbox: true
                }
            });

            this.authWindow.once('ready-to-show', () => {
                this.authWindow?.show();
            });

            this.authWindow.loadURL('https://labs.google/fx/tools/image-fx');

            this.authWindow.webContents.on('did-navigate', async (event, url) => {
                if (url.includes('labs.google/fx/tools/image-fx')) {
                    setTimeout(async () => {
                        try {
                            const { cookies, token, tokenExpiry, userEmail } = await this.extractSessionData();
                            
                            // Store session
                            await this.sessionManager.storeSession(cookies, token, tokenExpiry, userEmail);
                            
                            this.authWindow?.close();
                            resolve({ cookies, token });
                        } catch (error) {
                            reject(error);
                        }
                    }, 2000);
                }
            });

            this.authWindow.on('closed', () => {
                if (this.authWindow) {
                    reject(new Error('Authentication window closed'));
                }
                this.authWindow = null;
            });
        });
    }

    /**
     * Extract session data from authenticated browser
     */
    private async extractSessionData(): Promise<{
        cookies: string;
        token: string;
        tokenExpiry: number;
        userEmail?: string;
    }> {
        const { session } = require('electron');

        // Get cookies
        const cookies = await session.defaultSession.cookies.get({
            domain: '.google.com'
        });

        const cookieString = cookies
            .map(cookie => `${cookie.name}=${cookie.value}`)
            .join('; ');

        // Fetch session data to get token
        const response = await fetch('https://labs.google/fx/api/auth/session', {
            headers: {
                'Cookie': cookieString,
                'Origin': 'https://labs.google',
                'Referer': 'https://labs.google/fx/tools/image-fx'
            }
        });

        if (!response.ok) {
            throw new Error('Failed to fetch session data');
        }

        const sessionData = await response.json();

        return {
            cookies: cookieString,
            token: sessionData.access_token,
            tokenExpiry: new Date(sessionData.expires).getTime(),
            userEmail: sessionData.user?.email
        };
    }

    /**
     * Refresh session
     */
    private async refreshSession(): Promise<void> {
        console.log('Refreshing session...');

        const cookies = this.sessionManager.getCookies();
        
        if (!cookies) {
            throw new Error('No cookies available for refresh');
        }

        try {
            const response = await fetch('https://labs.google/fx/api/auth/session', {
                headers: {
                    'Cookie': cookies,
                    'Origin': 'https://labs.google',
                    'Referer': 'https://labs.google/fx/tools/image-fx'
                }
            });

            if (!response.ok) {
                throw new Error('Session refresh failed');
            }

            const sessionData = await response.json();

            await this.sessionManager.updateToken(
                sessionData.access_token,
                new Date(sessionData.expires).getTime()
            );

            console.log('Session refreshed successfully');
        } catch (error) {
            console.error('Failed to refresh session:', error);
            // Clear invalid session
            this.sessionManager.clearSession();
            throw error;
        }
    }

    /**
     * Get current cookies
     */
    getCookies(): string | null {
        return this.sessionManager.getCookies();
    }

    /**
     * Check if authenticated
     */
    isAuthenticated(): boolean {
        return this.sessionManager.hasStoredSession();
    }

    /**
     * Logout and clear session
     */
    logout(): void {
        this.sessionManager.clearSession();
    }

    /**
     * Get user info
     */
    getUserInfo() {
        return this.sessionManager.getMetadata();
    }
}
```

### Step 3: Create ImageFX Service with Persistent Session

Create `src/services/ImageFXServicePersistent.ts`:

```typescript
import { ImageFX } from '@rohitaryal/imagefx-api';
import { PersistentAccountManager } from '../auth/PersistentAccountManager';

export class ImageFXServicePersistent {
    private accountManager: PersistentAccountManager;
    private imageFX: ImageFX | null = null;
    private isInitializing = false;

    constructor() {
        this.accountManager = new PersistentAccountManager();
    }

    /**
     * Initialize the service
     * Uses stored credentials if available
     */
    async initialize(): Promise<void> {
        if (this.isInitializing) {
            throw new Error('Initialization already in progress');
        }

        this.isInitializing = true;

        try {
            const { cookies } = await this.accountManager.initialize();
            this.imageFX = new ImageFX(cookies);
            console.log('ImageFX service initialized');
        } finally {
            this.isInitializing = false;
        }
    }

    /**
     * Generate image with automatic session management
     */
    async generateImage(prompt: string, options?: any): Promise<any[]> {
        if (!this.imageFX) {
            // Auto-initialize if not ready
            await this.initialize();
        }

        if (!this.imageFX) {
            throw new Error('Failed to initialize ImageFX service');
        }

        try {
            return await this.imageFX.generateImage(prompt);
        } catch (error) {
            // If auth fails, try to re-initialize
            if (error instanceof Error && error.message.includes('Authentication')) {
                console.log('Authentication error, reinitializing...');
                this.accountManager.logout();
                await this.initialize();
                return await this.imageFX.generateImage(prompt);
            }
            throw error;
        }
    }

    /**
     * Check if service is ready
     */
    isReady(): boolean {
        return this.imageFX !== null;
    }

    /**
     * Check if user is authenticated
     */
    isAuthenticated(): boolean {
        return this.accountManager.isAuthenticated();
    }

    /**
     * Get user information
     */
    getUserInfo() {
        return this.accountManager.getUserInfo();
    }

    /**
     * Logout user and clear session
     */
    logout(): void {
        this.accountManager.logout();
        this.imageFX = null;
    }

    /**
     * Force re-authentication
     */
    async reAuthenticate(): Promise<void> {
        this.logout();
        await this.initialize();
    }
}
```

### Step 4: Integrate with Main Process

Update `main.ts`:

```typescript
import { app, BrowserWindow, ipcMain } from 'electron';
import { ImageFXServicePersistent } from './services/ImageFXServicePersistent';
import path from 'path';

let mainWindow: BrowserWindow | null = null;
let imageFXService: ImageFXServicePersistent;

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

    // Initialize service
    imageFXService = new ImageFXServicePersistent();

    // Auto-initialize on startup (silent, uses stored credentials)
    try {
        if (imageFXService.isAuthenticated()) {
            await imageFXService.initialize();
            console.log('ImageFX ready with stored credentials');
        }
    } catch (error) {
        console.log('No valid stored credentials, user will need to login');
    }
}

// IPC Handlers
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

ipcMain.handle('imagefx:isAuthenticated', () => {
    return imageFXService.isAuthenticated();
});

ipcMain.handle('imagefx:getUserInfo', () => {
    return imageFXService.getUserInfo();
});

ipcMain.handle('imagefx:generate', async (event, prompt: string, options?: any) => {
    try {
        const images = await imageFXService.generateImage(prompt, options);
        
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

ipcMain.handle('imagefx:logout', () => {
    imageFXService.logout();
    return { success: true };
});

ipcMain.handle('imagefx:reAuthenticate', async () => {
    try {
        await imageFXService.reAuthenticate();
        return { success: true };
    } catch (error) {
        return { 
            success: false, 
            error: error instanceof Error ? error.message : 'Unknown error' 
        };
    }
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

### Step 5: Usage in Renderer

```typescript
// Check if already authenticated
async function checkAuth() {
    const isAuthenticated = await window.imageFX.isAuthenticated();
    
    if (isAuthenticated) {
        console.log('Already authenticated!');
        const userInfo = await window.imageFX.getUserInfo();
        console.log('User:', userInfo.userEmail);
        // App is ready to use
    } else {
        // Show login prompt
        showLoginButton();
    }
}

// Login (only needed first time or after logout)
async function login() {
    const result = await window.imageFX.initialize();
    if (result.success) {
        console.log('Logged in successfully!');
        // Now you can generate images
    }
}

// Generate images (works automatically even after app restart)
async function generateImage(prompt: string) {
    const result = await window.imageFX.generate(prompt);
    if (result.success) {
        return result.images;
    }
    throw new Error(result.error);
}

// Initialize on app start
checkAuth();
```

## Advanced Features

### Session Health Monitoring

```typescript
class SessionHealthMonitor {
    private checkInterval: NodeJS.Timeout | null = null;

    start(onSessionInvalid: () => void) {
        this.checkInterval = setInterval(async () => {
            const isAuthenticated = await window.imageFX.isAuthenticated();
            if (!isAuthenticated) {
                onSessionInvalid();
                this.stop();
            }
        }, 60000); // Check every minute
    }

    stop() {
        if (this.checkInterval) {
            clearInterval(this.checkInterval);
        }
    }
}
```

### Offline Mode Support

```typescript
class OfflineImageCache {
    private cache: Map<string, string> = new Map();

    async generateWithCache(prompt: string): Promise<string> {
        // Check cache first
        if (this.cache.has(prompt)) {
            return this.cache.get(prompt)!;
        }

        // Generate new image
        const result = await window.imageFX.generate(prompt);
        const imagePath = result.images[0].path;

        // Cache for offline use
        this.cache.set(prompt, imagePath);

        return imagePath;
    }
}
```

## Security Best Practices

1. **Encryption**: Use hardware-backed encryption where available
2. **Key Management**: Store encryption keys in OS keychain
3. **Session Validation**: Regularly validate stored sessions
4. **Secure Deletion**: Properly clear session data on logout
5. **Audit Logging**: Log authentication events

## Troubleshooting

### Issue: Session refresh fails
**Solution**: Implement fallback to re-authentication if refresh fails.

### Issue: Encrypted data can't be decrypted
**Solution**: Version your encryption scheme and handle migration.

### Issue: Multiple accounts
**Solution**: Extend SessionManager to support multiple profiles.

## Production Checklist

- [ ] Encryption key properly secured
- [ ] Session refresh logic tested
- [ ] Error handling for all edge cases
- [ ] Automatic cleanup of expired sessions
- [ ] User logout flow implemented
- [ ] Account switching support
- [ ] Session migration strategy

## Next Steps

- Implement multi-account support
- Add session analytics and monitoring
- Integrate with system keychain
- Add biometric authentication
- Implement session backup/restore

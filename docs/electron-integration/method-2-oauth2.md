# Method 2: OAuth2 with Google Identity Services

## Overview

This method implements a proper OAuth2 flow using Google's official Identity Services. This is the most secure and production-ready approach, providing proper token management with refresh tokens for long-term authentication.

## Pros & Cons

### ✅ Advantages
- Most secure approach
- Official Google authentication method
- Refresh tokens allow long-term sessions
- Professional user experience
- Follows OAuth2 best practices
- No cookie extraction needed

### ⚠️ Disadvantages
- Requires Google Cloud project setup
- More complex initial configuration
- Need to manage OAuth credentials
- Requires handling redirect URIs in Electron

## Prerequisites

```bash
npm install electron
npm install @rohitaryal/imagefx-api
npm install electron-store  # For secure token storage
```

## Google Cloud Setup

### Step 1: Create Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select an existing one
3. Enable the required APIs:
   - Go to "APIs & Services" > "Library"
   - Search for and enable "Google Identity"

### Step 2: Create OAuth2 Credentials

1. Go to "APIs & Services" > "Credentials"
2. Click "Create Credentials" > "OAuth client ID"
3. Select "Desktop app" as application type
4. Name it (e.g., "StoryFramer Desktop")
5. Download the credentials JSON file
6. Note your:
   - Client ID: `YOUR_CLIENT_ID.apps.googleusercontent.com`
   - Client Secret: `YOUR_CLIENT_SECRET`

### Step 3: Configure OAuth Consent Screen

1. Go to "OAuth consent screen"
2. Choose "External" (or "Internal" if using Workspace)
3. Fill in app information:
   - App name: "StoryFramer"
   - User support email
   - Developer contact information
4. Add scopes (if needed for future features)
5. Add test users during development

## Implementation

### Step 1: Store OAuth Credentials Securely

Create `.env.example`:

```bash
GOOGLE_CLIENT_ID=your_client_id_here.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_client_secret_here
```

**⚠️ Important**: Add `.env` to `.gitignore` and never commit credentials!

### Step 2: Create OAuth2 Manager

Create `src/auth/OAuth2Manager.ts`:

```typescript
import { BrowserWindow } from 'electron';
import { URLSearchParams } from 'url';
import Store from 'electron-store';

interface OAuth2Tokens {
    access_token: string;
    refresh_token?: string;
    expires_at: number;
    token_type: string;
    scope?: string;
}

interface StoreSchema {
    oauth_tokens: OAuth2Tokens;
}

export class OAuth2Manager {
    private clientId: string;
    private clientSecret: string;
    private redirectUri = 'http://localhost:5173/oauth/callback'; // Electron app will intercept
    private store: Store<StoreSchema>;

    // Google OAuth2 endpoints
    private authEndpoint = 'https://accounts.google.com/o/oauth2/v2/auth';
    private tokenEndpoint = 'https://oauth2.googleapis.com/token';

    constructor(clientId: string, clientSecret: string) {
        this.clientId = clientId;
        this.clientSecret = clientSecret;
        this.store = new Store<StoreSchema>({
            name: 'oauth-tokens',
            encryptionKey: 'your-encryption-key-here' // Use a proper key management solution
        });
    }

    /**
     * Start OAuth2 authentication flow
     */
    async authenticate(): Promise<string> {
        // Check if we have valid tokens
        const tokens = this.store.get('oauth_tokens');
        if (tokens && this.isTokenValid(tokens)) {
            return this.getCookiesFromToken(tokens.access_token);
        }

        // Check if we can refresh
        if (tokens?.refresh_token) {
            try {
                const newTokens = await this.refreshAccessToken(tokens.refresh_token);
                return this.getCookiesFromToken(newTokens.access_token);
            } catch (error) {
                console.log('Failed to refresh token, starting new auth flow');
            }
        }

        // Start new OAuth2 flow
        return this.startAuthFlow();
    }

    /**
     * Start the OAuth2 authorization flow
     */
    private async startAuthFlow(): Promise<string> {
        return new Promise((resolve, reject) => {
            const authUrl = this.buildAuthUrl();

            const authWindow = new BrowserWindow({
                width: 600,
                height: 800,
                show: false,
                webPreferences: {
                    nodeIntegration: false,
                    contextIsolation: true
                }
            });

            authWindow.loadURL(authUrl);
            authWindow.show();

            // Intercept the redirect
            authWindow.webContents.on('will-redirect', async (event, url) => {
                if (url.startsWith(this.redirectUri)) {
                    event.preventDefault();
                    authWindow.close();

                    try {
                        const code = this.extractAuthCode(url);
                        const tokens = await this.exchangeCodeForTokens(code);
                        this.store.set('oauth_tokens', tokens);
                        const cookies = await this.getCookiesFromToken(tokens.access_token);
                        resolve(cookies);
                    } catch (error) {
                        reject(error);
                    }
                }
            });

            authWindow.on('closed', () => {
                reject(new Error('Auth window was closed by user'));
            });
        });
    }

    /**
     * Build the OAuth2 authorization URL
     */
    private buildAuthUrl(): string {
        const params = new URLSearchParams({
            client_id: this.clientId,
            redirect_uri: this.redirectUri,
            response_type: 'code',
            scope: 'openid email profile',
            access_type: 'offline', // Get refresh token
            prompt: 'consent' // Force consent to get refresh token
        });

        return `${this.authEndpoint}?${params.toString()}`;
    }

    /**
     * Extract authorization code from redirect URL
     */
    private extractAuthCode(url: string): string {
        const urlParams = new URL(url);
        const code = urlParams.searchParams.get('code');
        
        if (!code) {
            throw new Error('No authorization code found in redirect');
        }

        return code;
    }

    /**
     * Exchange authorization code for access and refresh tokens
     */
    private async exchangeCodeForTokens(code: string): Promise<OAuth2Tokens> {
        const params = new URLSearchParams({
            code,
            client_id: this.clientId,
            client_secret: this.clientSecret,
            redirect_uri: this.redirectUri,
            grant_type: 'authorization_code'
        });

        const response = await fetch(this.tokenEndpoint, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded'
            },
            body: params.toString()
        });

        if (!response.ok) {
            const error = await response.text();
            throw new Error(`Token exchange failed: ${error}`);
        }

        const data = await response.json();

        return {
            access_token: data.access_token,
            refresh_token: data.refresh_token,
            expires_at: Date.now() + (data.expires_in * 1000),
            token_type: data.token_type,
            scope: data.scope
        };
    }

    /**
     * Refresh access token using refresh token
     */
    private async refreshAccessToken(refreshToken: string): Promise<OAuth2Tokens> {
        const params = new URLSearchParams({
            client_id: this.clientId,
            client_secret: this.clientSecret,
            refresh_token: refreshToken,
            grant_type: 'refresh_token'
        });

        const response = await fetch(this.tokenEndpoint, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded'
            },
            body: params.toString()
        });

        if (!response.ok) {
            throw new Error('Failed to refresh token');
        }

        const data = await response.json();

        const tokens: OAuth2Tokens = {
            access_token: data.access_token,
            refresh_token: refreshToken, // Keep the same refresh token
            expires_at: Date.now() + (data.expires_in * 1000),
            token_type: data.token_type,
            scope: data.scope
        };

        this.store.set('oauth_tokens', tokens);
        return tokens;
    }

    /**
     * Get cookies for ImageFX API using access token
     * This requires making a request to Google's session endpoint
     */
    private async getCookiesFromToken(accessToken: string): Promise<string> {
        // First, establish a session with Google using the access token
        const response = await fetch('https://labs.google/fx/api/auth/session', {
            headers: {
                'Authorization': `Bearer ${accessToken}`,
                'Origin': 'https://labs.google',
                'Referer': 'https://labs.google/fx/tools/image-fx'
            }
        });

        if (!response.ok) {
            throw new Error('Failed to establish ImageFX session');
        }

        // Extract Set-Cookie headers
        const cookies = response.headers.get('set-cookie');
        
        if (!cookies) {
            // If no cookies in response, we need to extract from session data
            const sessionData = await response.json();
            
            // Create a cookie string from session data
            // Note: This is a simplified version; actual implementation may vary
            return this.createCookieString(sessionData);
        }

        return this.parseCookieHeader(cookies);
    }

    /**
     * Create cookie string from session data
     */
    private createCookieString(sessionData: any): string {
        // This is a helper method that constructs the cookie string
        // Implementation depends on the actual session data structure
        const cookies: string[] = [];
        
        if (sessionData.access_token) {
            // Note: In practice, you might need to set up a browser session
            // and extract actual cookies. This is a simplified example.
            return `access_token=${sessionData.access_token}`;
        }

        return cookies.join('; ');
    }

    /**
     * Parse Set-Cookie header into cookie string
     */
    private parseCookieHeader(setCookieHeader: string): string {
        const cookies = setCookieHeader.split(',').map(cookie => {
            const parts = cookie.split(';')[0].trim();
            return parts;
        });

        return cookies.join('; ');
    }

    /**
     * Check if token is valid and not expired
     */
    private isTokenValid(tokens: OAuth2Tokens): boolean {
        return tokens.expires_at > Date.now() + (5 * 60 * 1000); // 5 min buffer
    }

    /**
     * Clear stored tokens (logout)
     */
    clearTokens(): void {
        this.store.delete('oauth_tokens');
    }

    /**
     * Check if user is authenticated
     */
    isAuthenticated(): boolean {
        const tokens = this.store.get('oauth_tokens');
        return tokens !== undefined && this.isTokenValid(tokens);
    }
}
```

### Step 3: Create Enhanced ImageFX Service

Create `src/services/ImageFXServiceOAuth.ts`:

```typescript
import { ImageFX } from '@rohitaryal/imagefx-api';
import { OAuth2Manager } from '../auth/OAuth2Manager';

export class ImageFXServiceOAuth {
    private oauth2Manager: OAuth2Manager;
    private imageFX: ImageFX | null = null;
    private cookies: string = '';

    constructor(clientId: string, clientSecret: string) {
        this.oauth2Manager = new OAuth2Manager(clientId, clientSecret);
    }

    /**
     * Initialize the service with OAuth2 authentication
     */
    async initialize(): Promise<void> {
        try {
            this.cookies = await this.oauth2Manager.authenticate();
            this.imageFX = new ImageFX(this.cookies);
        } catch (error) {
            throw new Error(`Failed to initialize ImageFX: ${error instanceof Error ? error.message : 'Unknown error'}`);
        }
    }

    /**
     * Generate an image with automatic token refresh
     */
    async generateImage(prompt: string): Promise<any[]> {
        if (!this.imageFX) {
            throw new Error('Service not initialized');
        }

        try {
            return await this.imageFX.generateImage(prompt);
        } catch (error) {
            // Try to refresh session and retry once
            if (error instanceof Error && error.message.includes('Authentication')) {
                console.log('Session expired, refreshing...');
                await this.initialize();
                return await this.imageFX.generateImage(prompt);
            }
            throw error;
        }
    }

    /**
     * Logout user
     */
    logout(): void {
        this.oauth2Manager.clearTokens();
        this.imageFX = null;
        this.cookies = '';
    }

    /**
     * Check if user is authenticated
     */
    isAuthenticated(): boolean {
        return this.oauth2Manager.isAuthenticated();
    }
}
```

### Step 4: Update Main Process

Update your `main.ts`:

```typescript
import { app, BrowserWindow, ipcMain } from 'electron';
import { ImageFXServiceOAuth } from './services/ImageFXServiceOAuth';
import path from 'path';
import * as dotenv from 'dotenv';

dotenv.config();

let mainWindow: BrowserWindow | null = null;
let imageFXService: ImageFXServiceOAuth;

async function createWindow() {
    const clientId = process.env.GOOGLE_CLIENT_ID!;
    const clientSecret = process.env.GOOGLE_CLIENT_SECRET!;

    if (!clientId || !clientSecret) {
        throw new Error('OAuth2 credentials not configured');
    }

    imageFXService = new ImageFXServiceOAuth(clientId, clientSecret);

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
        await imageFXService.initialize();
        return { success: true };
    } catch (error) {
        return { success: false, error: error instanceof Error ? error.message : 'Unknown error' };
    }
});

ipcMain.handle('imagefx:logout', () => {
    imageFXService.logout();
    return { success: true };
});

ipcMain.handle('imagefx:isAuthenticated', () => {
    return imageFXService.isAuthenticated();
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
        return { success: false, error: error instanceof Error ? error.message : 'Unknown error' };
    }
});

app.whenReady().then(createWindow);
```

### Step 5: Usage in Renderer

```typescript
// Check authentication status on app start
async function checkAuth() {
    const isAuthenticated = await window.imageFX.isAuthenticated();
    
    if (!isAuthenticated) {
        // Show login button
        document.getElementById('login-btn')?.addEventListener('click', async () => {
            const result = await window.imageFX.login();
            if (result.success) {
                console.log('Logged in successfully!');
                // Update UI
            }
        });
    } else {
        // User is already authenticated
        console.log('Already authenticated!');
    }
}

// Generate image
async function generateImage(prompt: string) {
    const result = await window.imageFX.generate(prompt);
    if (result.success) {
        return result.images;
    }
    throw new Error(result.error);
}
```

## Configuration File Example

Create `config/oauth.json`:

```json
{
    "redirectUri": "http://localhost:5173/oauth/callback",
    "scopes": ["openid", "email", "profile"],
    "accessType": "offline"
}
```

## Security Best Practices

1. **Never hardcode credentials** - Use environment variables or secure configuration
2. **Encrypt token storage** - Use electron-store with encryption
3. **Handle token refresh** - Implement automatic refresh before expiry
4. **Secure IPC communication** - Validate all IPC messages
5. **Use HTTPS only** - Never transmit tokens over HTTP

## Troubleshooting

### Issue: "Redirect URI mismatch"
**Solution**: Ensure the redirect URI in your code matches exactly what's configured in Google Cloud Console.

### Issue: Refresh token not received
**Solution**: Make sure to include `access_type: 'offline'` and `prompt: 'consent'` in the auth URL.

### Issue: Token refresh fails
**Solution**: Refresh tokens can be revoked. Implement fallback to full re-authentication.

## Production Checklist

- [ ] OAuth2 credentials configured in Google Cloud
- [ ] Redirect URIs properly configured
- [ ] Token storage encrypted
- [ ] Error handling implemented
- [ ] Token refresh logic tested
- [ ] Logout functionality working
- [ ] Security review completed

## Next Steps

- Implement token refresh scheduling
- Add multi-account support
- Combine with session persistence (Method 4)
- Add comprehensive error logging

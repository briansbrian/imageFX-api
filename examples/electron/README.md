# ImageFX Electron Example

This is a complete example Electron application demonstrating how to integrate ImageFX for seamless image generation within your desktop app.

## Features

- ✅ Multiple authentication methods
- ✅ Persistent session management
- ✅ Automatic session refresh
- ✅ Clean UI for testing
- ✅ Complete TypeScript implementation

## Quick Start

```bash
# Install dependencies
npm install

# Build the app
npm run build

# Run the app
npm start
```

## Project Structure

```
examples/electron/
├── src/
│   ├── main.ts                 # Main process entry point
│   ├── preload.ts              # Preload script for IPC
│   ├── auth/
│   │   ├── GoogleAuthManager.ts       # Method 1: Hidden browser auth
│   │   ├── OAuth2Manager.ts           # Method 2: OAuth2 flow
│   │   ├── CookieImportManager.ts     # Method 3: Cookie import
│   │   ├── PersistentAccountManager.ts # Method 4: Persistent session
│   │   └── SecureSessionManager.ts    # Session encryption & storage
│   ├── services/
│   │   └── ImageFXService.ts   # ImageFX service wrapper
│   └── renderer/
│       ├── index.html          # Main UI
│       └── renderer.ts         # Renderer process logic
├── package.json
├── tsconfig.json
└── README.md
```

## Authentication Methods

This example includes all four authentication methods. Choose the one that fits your needs:

### Method 1: Hidden Browser (Default)
- Simple, built-in Google login
- Recommended for most applications

### Method 2: OAuth2
- Most secure, production-ready
- Requires Google Cloud project setup
- See: `docs/electron-integration/method-2-oauth2.md`

### Method 3: Cookie Import
- Good for development/testing
- Users manually import cookies
- See: `docs/electron-integration/method-3-cookie-import.md`

### Method 4: Persistent Session
- Best user experience
- One-time authentication
- Automatic session refresh
- See: `docs/electron-integration/method-4-persistent-session.md`

## Usage in Your App

### Basic Image Generation

```typescript
// In your renderer process
async function generateSceneFrame(prompt: string) {
    const result = await window.imageFX.generate(prompt, {
        aspectRatio: 'LANDSCAPE',
        count: 1
    });

    if (result.success) {
        return result.images[0].path;
    }
}
```

### Integration with Scene Nodes

```typescript
class SceneNode {
    firstFrameImage: string | null = null;
    lastFrameImage: string | null = null;

    async generateFirstFrame(description: string) {
        const prompt = `First frame: ${description}, cinematic, high quality`;
        this.firstFrameImage = await generateSceneFrame(prompt);
    }

    async generateLastFrame(description: string) {
        const prompt = `Last frame: ${description}, cinematic, high quality`;
        this.lastFrameImage = await generateSceneFrame(prompt);
    }
}
```

## Configuration

### For OAuth2 (Method 2)

Create `.env` file:

```bash
GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_client_secret
```

### For Custom Settings

Edit `src/config.ts`:

```typescript
export const config = {
    authMethod: 'persistent', // 'hidden-browser' | 'oauth2' | 'cookie-import' | 'persistent'
    imageSavePath: 'generated-images',
    defaultModel: 'IMAGEN_3_5',
    defaultAspectRatio: 'LANDSCAPE'
};
```

## Building for Production

```bash
# Install electron-builder
npm install --save-dev electron-builder

# Build for current platform
npm run build
electron-builder

# Build for specific platforms
electron-builder --windows
electron-builder --mac
electron-builder --linux
```

## Security Notes

⚠️ **Important**: This example uses basic encryption for demonstration. For production:

1. Use hardware-backed encryption (macOS Keychain, Windows Credential Manager)
2. Implement proper key management
3. Add security auditing and logging
4. Follow Electron security best practices
5. Never commit credentials or cookies to git

## Troubleshooting

### Issue: "Module not found"
```bash
npm install
npm run build
```

### Issue: Authentication fails
- Check internet connection
- Ensure you're logged into Google in a browser
- Clear stored session and try again

### Issue: Images not saving
- Check file permissions
- Verify save path exists
- Check available disk space

## Learn More

- [Full Documentation](../../docs/electron-integration/README.md)
- [ImageFX API Docs](../../README.md)
- [Electron Documentation](https://www.electronjs.org/docs)

## License

MIT

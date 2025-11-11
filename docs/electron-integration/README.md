# ImageFX Integration for Electron Applications

This guide provides multiple methods for seamlessly integrating Google ImageFX into your Electron-based application (like StoryFramer). Each method addresses the challenge of authenticating with Google and maintaining a session for image generation without displaying the ImageFX UI.

## Overview

Google ImageFX requires authentication through a Google account. For Electron apps, we need to:
1. Authenticate the user with their Google account
2. Extract and manage session cookies
3. Use those cookies to generate images via the ImageFX API
4. Maintain the session for continued use

## Available Methods

We provide **four different approaches**, each with different trade-offs:

### [Method 1: Hidden BrowserWindow with Cookie Extraction](./method-1-hidden-browser.md)
**Best for**: Quick implementation, minimal setup
- Uses Electron's BrowserWindow in hidden mode
- Automatically extracts cookies after Google authentication
- Simple session management
- ⚠️ Requires one-time login per session

### [Method 2: OAuth2 with Google Identity Services](./method-2-oauth2.md)
**Best for**: Production apps, best security practices
- Uses official Google OAuth2 flow
- Secure token management with refresh tokens
- Professional authentication experience
- ✅ Most secure and maintainable approach

### [Method 3: Cookie Import from External Browser](./method-3-cookie-import.md)
**Best for**: Development, testing, or power users
- Import cookies from user's browser via extension
- Minimal auth implementation in your app
- Good for rapid prototyping
- ⚠️ Requires user to manually export cookies

### [Method 4: Persistent Session with Auto-Refresh](./method-4-persistent-session.md)
**Best for**: Seamless user experience
- Stores encrypted session data locally
- Automatic session refresh before expiry
- One-time authentication per installation
- ✅ Best user experience after initial setup

## Quick Start

1. Choose the method that best fits your needs
2. Follow the detailed guide in the respective method documentation
3. Install required dependencies
4. Implement the authentication flow
5. Start generating images!

## Common Prerequisites

All methods require:

```bash
npm install @rohitaryal/imagefx-api
```

For Electron-specific integration:

```bash
npm install electron
npm install electron-store  # For session persistence (Methods 2 & 4)
```

## Architecture Overview

```
┌─────────────────────┐
│   Electron App      │
│  (StoryFramer)      │
└──────────┬──────────┘
           │
           ├─► Authentication Layer
           │   ├─ Hidden BrowserWindow (Method 1)
           │   ├─ OAuth2 Flow (Method 2)
           │   ├─ Cookie Import (Method 3)
           │   └─ Persistent Session (Method 4)
           │
           ├─► Session Manager
           │   ├─ Cookie Storage
           │   ├─ Token Refresh
           │   └─ Session Validation
           │
           └─► ImageFX API Integration
               ├─ Image Generation
               ├─ Prompt Management
               └─ Image Storage
```

## Security Considerations

⚠️ **Important Security Notes:**

1. **Never commit credentials** to your repository
2. **Encrypt stored cookies** if using persistent storage
3. **Use Electron's security best practices**:
   - Enable context isolation
   - Disable Node integration in renderers
   - Use secure IPC communication
4. **Handle token expiration** gracefully
5. **Clear session data** on app uninstall

## Comparison Matrix

| Feature | Method 1 | Method 2 | Method 3 | Method 4 |
|---------|----------|----------|----------|----------|
| Ease of Implementation | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Security | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| User Experience | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Maintenance | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| Session Persistence | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Production Ready | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |

## Need Help?

- Check the troubleshooting section in each method's documentation
- Review the example implementations in `/examples/electron/`
- Open an issue on GitHub for specific problems

## License

This integration guide is part of the imageFX-api project and follows the same license terms.

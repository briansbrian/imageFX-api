# Complete Guide: Integrating ImageFX into Electron Applications

## Introduction

This comprehensive guide shows you how to integrate Google ImageFX into your Electron-based application (like StoryFramer) to generate images seamlessly without relying on the official ImageFX website interface.

**⚠️ Important Note**: This repository (imagefx-api) serves as a **reference implementation** showing how authentication and API communication work. The methods described in this guide are designed to help you implement your own direct integration using current best practices, not to depend on this npm package. You should research and use the most up-to-date authentication and API communication methods available.

## Table of Contents

1. [Overview](#overview)
2. [Why This Approach?](#why-this-approach)
3. [Architecture](#architecture)
4. [Authentication Methods](#authentication-methods)
5. [Implementation Roadmap](#implementation-roadmap)
6. [StoryFramer-Specific Integration](#storyframer-specific-integration)
7. [Best Practices](#best-practices)
8. [Advanced Topics](#advanced-topics)

## Overview

The challenge: You want to use ImageFX's powerful image generation within your Electron app without showing the ImageFX UI or requiring manual website interaction.

The solution: This guide provides **four different authentication and session management approaches**, each suited to different needs:

- **Method 1**: Hidden BrowserWindow - Simple, fast setup
- **Method 2**: OAuth2 - Production-ready, most secure
- **Method 3**: Cookie Import - Quick for development
- **Method 4**: Persistent Session - Best user experience

## Why This Approach?

### The Problem
- ImageFX requires Google account authentication
- Official API doesn't exist (free service)
- Need to maintain active session
- Want seamless integration without browser UI

### The Solution
1. Authenticate user once with Google
2. Extract and securely store session credentials
3. Use credentials with imagefx-api package
4. Auto-refresh sessions to maintain access
5. Generate images directly in your app

### Benefits for StoryFramer
- Generate first/last frame images for scene nodes
- No manual image imports needed
- Seamless workflow integration
- Free image generation (using ImageFX)
- Complete creative control

## Architecture

### High-Level Flow

```
┌─────────────────────────────────────────────────┐
│         Your Electron App (StoryFramer)        │
│                                                  │
│  ┌────────────────────────────────────────┐    │
│  │   Main Process (Node.js)               │    │
│  │                                         │    │
│  │  ┌──────────────────────────────────┐  │    │
│  │  │  Authentication Manager          │  │    │
│  │  │  • Hidden Browser Auth           │  │    │
│  │  │  • OAuth2 Flow                   │  │    │
│  │  │  • Cookie Management             │  │    │
│  │  │  • Session Persistence           │  │    │
│  │  └──────────────────────────────────┘  │    │
│  │                                         │    │
│  │  ┌──────────────────────────────────┐  │    │
│  │  │  ImageFX Service                 │  │    │
│  │  │  • Image Generation              │  │    │
│  │  │  • Prompt Management             │  │    │
│  │  │  • Result Handling               │  │    │
│  │  └──────────────────────────────────┘  │    │
│  │                                         │    │
│  │  ┌──────────────────────────────────┐  │    │
│  │  │  Session Manager                 │  │    │
│  │  │  • Secure Storage                │  │    │
│  │  │  • Auto-Refresh                  │  │    │
│  │  │  • Expiry Handling               │  │    │
│  │  └──────────────────────────────────┘  │    │
│  └────────────────────────────────────────┘    │
│                     ↕ IPC                       │
│  ┌────────────────────────────────────────┐    │
│  │   Renderer Process (UI)                │    │
│  │                                         │    │
│  │  • Scene Node Editor                   │    │
│  │  • Generate Frame Images               │    │
│  │  • Display Generated Results           │    │
│  └────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
         ↕ HTTPS (via imagefx-api)
┌─────────────────────────────────────────────────┐
│         Google ImageFX Service                  │
│         (labs.google/fx/tools/image-fx)        │
└─────────────────────────────────────────────────┘
```

### Data Flow

```
User Action → Renderer Process → IPC → Main Process
    → Auth Manager → Session Manager → ImageFX API
    → Google ImageFX → Response → Process Image
    → Save to Disk → Return Path → Update UI
```

## Authentication Methods

### Quick Comparison

| Criteria | Method 1 | Method 2 | Method 3 | Method 4 |
|----------|----------|----------|----------|----------|
| **Setup Time** | 30 min | 2 hours | 15 min | 1 hour |
| **User Experience** | Good | Excellent | Fair | Excellent |
| **Security** | Good | Excellent | Fair | Very Good |
| **Maintenance** | Low | Medium | Very Low | Medium |
| **Production Ready** | Yes | Yes | No | Yes |
| **Session Persistence** | No | Yes | Manual | Yes |
| **Auto-Refresh** | No | Yes | No | Yes |

### Method Selection Guide

**Choose Method 1** if:
- You need quick implementation
- Simple authentication is acceptable
- Session per app-launch is fine
- You're building an MVP or prototype

**Choose Method 2** if:
- Building a production application
- Security is paramount
- You can set up Google Cloud project
- Need official OAuth2 compliance

**Choose Method 3** if:
- In development/testing phase
- Building for technical users
- Want minimal code
- Need rapid prototyping

**Choose Method 4** if:
- Want best user experience
- Session persistence is critical
- Can manage encryption keys
- Building for end users

## Implementation Roadmap

### Phase 1: Setup (Day 1)

1. **Install Dependencies**
   ```bash
   npm install @rohitaryal/imagefx-api
   npm install electron-store
   npm install crypto-js
   ```

2. **Choose Authentication Method**
   - Review [method comparison](./electron-integration/README.md)
   - Select based on your requirements

3. **Set Up Project Structure**
   ```
   src/
   ├── main.ts
   ├── preload.ts
   ├── auth/
   │   └── [Your chosen auth manager]
   └── services/
       └── ImageFXService.ts
   ```

### Phase 2: Authentication (Day 1-2)

1. **Implement Authentication**
   - Follow your chosen method's guide
   - Test authentication flow
   - Handle error cases

2. **Test Session Management**
   - Verify cookie extraction
   - Test session persistence
   - Validate token refresh

### Phase 3: Integration (Day 2-3)

1. **Create ImageFX Service**
   - Wrap imagefx-api functionality
   - Add error handling
   - Implement retry logic

2. **Set Up IPC Communication**
   - Define IPC channels
   - Implement handlers in main process
   - Expose API in preload script

3. **Build UI Components**
   - Authentication status display
   - Image generation controls
   - Progress indicators

### Phase 4: StoryFramer Integration (Day 3-4)

1. **Scene Node Integration**
   - Add image generation to nodes
   - Connect to frame properties
   - Handle async generation

2. **UI/UX Polish**
   - Loading states
   - Error messages
   - Success feedback

3. **Testing**
   - Test all authentication flows
   - Test image generation
   - Test error scenarios

### Phase 5: Production Prep (Day 4-5)

1. **Security Audit**
   - Review credential storage
   - Check encryption implementation
   - Validate IPC security

2. **Performance Optimization**
   - Implement caching
   - Optimize session refresh
   - Handle rate limiting

3. **Documentation**
   - User guide
   - Troubleshooting docs
   - API documentation

## StoryFramer-Specific Integration

### Scene Node with Image Generation

```typescript
interface SceneNode {
    id: string;
    name: string;
    firstFrame: string | null;  // Path to generated image
    lastFrame: string | null;   // Path to generated image
    description: string;
}

class SceneNodeManager {
    private imageFX: ImageFXService;

    constructor() {
        this.imageFX = new ImageFXService();
    }

    async generateSceneFrames(node: SceneNode): Promise<void> {
        // Generate first frame
        const firstPrompt = this.createFramePrompt(
            node.description, 
            'establishing shot, beginning of scene'
        );
        
        const firstFrameResult = await this.imageFX.generate(firstPrompt, {
            aspectRatio: 'LANDSCAPE',
            model: 'IMAGEN_3_5'
        });

        node.firstFrame = firstFrameResult.images[0].path;

        // Generate last frame
        const lastPrompt = this.createFramePrompt(
            node.description,
            'final shot, conclusion of scene'
        );

        const lastFrameResult = await this.imageFX.generate(lastPrompt, {
            aspectRatio: 'LANDSCAPE',
            model: 'IMAGEN_3_5'
        });

        node.lastFrame = lastFrameResult.images[0].path;
    }

    private createFramePrompt(description: string, context: string): string {
        return `Cinematic frame from a storyboard: ${description}. 
                ${context}. Professional cinematography, 
                detailed scene, high quality render.`;
    }

    async generateVariations(node: SceneNode, count: number = 4): Promise<string[]> {
        const prompt = this.createFramePrompt(node.description, 'key frame');
        
        const result = await this.imageFX.generate(prompt, {
            count: count,
            aspectRatio: 'LANDSCAPE'
        });

        return result.images.map(img => img.path);
    }
}
```

### React Component Example

```tsx
import React, { useState } from 'react';

interface SceneNodeProps {
    node: SceneNode;
    onUpdate: (node: SceneNode) => void;
}

export function SceneNodeComponent({ node, onUpdate }: SceneNodeProps) {
    const [generating, setGenerating] = useState(false);
    const [error, setError] = useState<string | null>(null);

    const handleGenerateFrames = async () => {
        setGenerating(true);
        setError(null);

        try {
            // Generate first frame
            const firstResult = await window.imageFX.generate(
                `First frame: ${node.description}, cinematic establishing shot`,
                { aspectRatio: 'LANDSCAPE' }
            );

            // Generate last frame
            const lastResult = await window.imageFX.generate(
                `Last frame: ${node.description}, cinematic closing shot`,
                { aspectRatio: 'LANDSCAPE' }
            );

            if (firstResult.success && lastResult.success) {
                onUpdate({
                    ...node,
                    firstFrame: firstResult.images[0].path,
                    lastFrame: lastResult.images[0].path
                });
            } else {
                throw new Error('Generation failed');
            }
        } catch (err) {
            setError(err instanceof Error ? err.message : 'Unknown error');
        } finally {
            setGenerating(false);
        }
    };

    return (
        <div className="scene-node">
            <h3>{node.name}</h3>
            <p>{node.description}</p>

            <div className="frames">
                <div className="frame">
                    <h4>First Frame</h4>
                    {node.firstFrame ? (
                        <img src={`file://${node.firstFrame}`} alt="First frame" />
                    ) : (
                        <div className="placeholder">No image</div>
                    )}
                </div>

                <div className="frame">
                    <h4>Last Frame</h4>
                    {node.lastFrame ? (
                        <img src={`file://${node.lastFrame}`} alt="Last frame" />
                    ) : (
                        <div className="placeholder">No image</div>
                    )}
                </div>
            </div>

            <button 
                onClick={handleGenerateFrames}
                disabled={generating}
            >
                {generating ? 'Generating...' : 'Generate Frames'}
            </button>

            {error && <div className="error">{error}</div>}
        </div>
    );
}
```

### Batch Processing for Multiple Nodes

```typescript
class BatchImageGenerator {
    private queue: Array<{node: SceneNode, type: 'first' | 'last'}> = [];
    private processing = false;

    async addToQueue(node: SceneNode, type: 'first' | 'last') {
        this.queue.push({ node, type });
        
        if (!this.processing) {
            await this.processQueue();
        }
    }

    private async processQueue() {
        this.processing = true;

        while (this.queue.length > 0) {
            const item = this.queue.shift();
            if (!item) break;

            try {
                const prompt = this.createPrompt(item.node, item.type);
                const result = await window.imageFX.generate(prompt);

                if (result.success) {
                    if (item.type === 'first') {
                        item.node.firstFrame = result.images[0].path;
                    } else {
                        item.node.lastFrame = result.images[0].path;
                    }
                }

                // Rate limiting: wait 2 seconds between generations
                await new Promise(resolve => setTimeout(resolve, 2000));
            } catch (error) {
                console.error('Batch generation error:', error);
            }
        }

        this.processing = false;
    }

    private createPrompt(node: SceneNode, type: 'first' | 'last'): string {
        const context = type === 'first' 
            ? 'establishing shot, beginning' 
            : 'closing shot, ending';
        
        return `${node.description}, ${context}, cinematic, high quality`;
    }
}
```

## Best Practices

### Security

1. **Never Hardcode Credentials**
   ```typescript
   // ❌ Bad
   const clientId = '123456-abc.apps.googleusercontent.com';
   
   // ✅ Good
   const clientId = process.env.GOOGLE_CLIENT_ID;
   ```

2. **Encrypt Stored Data**
   ```typescript
   import CryptoJS from 'crypto-js';
   
   function encryptCredentials(data: string): string {
       const key = getSecureKey();
       return CryptoJS.AES.encrypt(data, key).toString();
   }
   ```

3. **Use Context Isolation**
   ```typescript
   new BrowserWindow({
       webPreferences: {
           contextIsolation: true,
           nodeIntegration: false,
           sandbox: true
       }
   });
   ```

### Performance

1. **Cache Generated Images**
   ```typescript
   class ImageCache {
       private cache = new Map<string, string>();
       
       async getOrGenerate(prompt: string): Promise<string> {
           if (this.cache.has(prompt)) {
               return this.cache.get(prompt)!;
           }
           
           const result = await generateImage(prompt);
           this.cache.set(prompt, result);
           return result;
       }
   }
   ```

2. **Implement Request Queuing**
   - Avoid overwhelming the API
   - Rate limit requests
   - Show progress to users

3. **Optimize Session Refresh**
   - Refresh before expiry
   - Use exponential backoff on failures
   - Cache valid sessions

### Error Handling

```typescript
async function safeImageGeneration(prompt: string): Promise<Result> {
    try {
        return await window.imageFX.generate(prompt);
    } catch (error) {
        if (error.message.includes('Authentication')) {
            // Handle auth errors
            await reAuthenticate();
            return await window.imageFX.generate(prompt);
        }
        
        if (error.message.includes('Rate limit')) {
            // Handle rate limiting
            await delay(5000);
            return await window.imageFX.generate(prompt);
        }
        
        // Log and show user-friendly error
        console.error('Generation failed:', error);
        showErrorToUser('Failed to generate image. Please try again.');
        throw error;
    }
}
```

## Advanced Topics

### Multi-Account Support

```typescript
class MultiAccountManager {
    private accounts = new Map<string, ImageFXService>();
    private activeAccount: string | null = null;

    async addAccount(email: string): Promise<void> {
        const service = new ImageFXService();
        await service.initialize();
        this.accounts.set(email, service);
    }

    switchAccount(email: string): void {
        if (this.accounts.has(email)) {
            this.activeAccount = email;
        }
    }

    getActiveService(): ImageFXService {
        if (!this.activeAccount) {
            throw new Error('No active account');
        }
        return this.accounts.get(this.activeAccount)!;
    }
}
```

### Prompt Templates

```typescript
const promptTemplates = {
    storyboard: {
        firstFrame: (desc: string) => 
            `Cinematic storyboard first frame: ${desc}, establishing shot, professional cinematography`,
        lastFrame: (desc: string) => 
            `Cinematic storyboard last frame: ${desc}, closing shot, emotional resolution`,
        keyFrame: (desc: string) => 
            `Cinematic storyboard key frame: ${desc}, important moment, dramatic lighting`
    },
    
    concept: {
        character: (desc: string) => 
            `Character concept art: ${desc}, full body, detailed design, professional illustration`,
        environment: (desc: string) => 
            `Environment concept art: ${desc}, wide shot, detailed background, atmospheric`
    }
};
```

### Analytics and Monitoring

```typescript
class ImageFXAnalytics {
    private metrics = {
        generationsSuccess: 0,
        generationsFailed: 0,
        averageGenerationTime: 0,
        authRefreshes: 0
    };

    trackGeneration(success: boolean, duration: number) {
        if (success) {
            this.metrics.generationsSuccess++;
            this.updateAverageTime(duration);
        } else {
            this.metrics.generationsFailed++;
        }
    }

    trackAuthRefresh() {
        this.metrics.authRefreshes++;
    }

    getMetrics() {
        return { ...this.metrics };
    }
}
```

## Resources

### Documentation
- [Method 1: Hidden Browser](./electron-integration/method-1-hidden-browser.md)
- [Method 2: OAuth2](./electron-integration/method-2-oauth2.md)
- [Method 3: Cookie Import](./electron-integration/method-3-cookie-import.md)
- [Method 4: Persistent Session](./electron-integration/method-4-persistent-session.md)

### Example Code
- [Complete Electron Example](../examples/electron/)
- [Basic Integration Example](../examples/1-generate-and-save.ts)

### External Resources
- [Electron Documentation](https://www.electronjs.org/docs)
- [Google OAuth2 Setup](https://developers.google.com/identity/protocols/oauth2)
- [Electron Security](https://www.electronjs.org/docs/tutorial/security)

## Support

- GitHub Issues: [Report bugs or request features](https://github.com/rohitaryal/imageFX-api/issues)
- Discussions: [Ask questions](https://github.com/rohitaryal/imageFX-api/discussions)

## License

This guide and example code are provided under the same license as the imageFX-api project.

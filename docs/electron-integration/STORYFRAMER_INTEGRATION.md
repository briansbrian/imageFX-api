# StoryFramer-Specific Integration Guide

Complete guide for integrating ImageFX into StoryFramer for automated scene frame generation.

## Overview

StoryFramer is a node-based storyboarding application where:
- Each scene is represented as a node
- Nodes have connections for workflow
- Each scene node has first frame and last frame properties
- Currently relies on manual image imports

**Goal**: Generate first and last frame images automatically using ImageFX without manual imports.

## Architecture for StoryFramer

```
┌──────────────────────────────────────────────────┐
│              StoryFramer Application              │
│                                                   │
│  ┌─────────────────────────────────────────┐    │
│  │         Scene Graph / Canvas            │    │
│  │  ┌─────────┐    ┌─────────┐             │    │
│  │  │ Scene 1 │───▶│ Scene 2 │             │    │
│  │  │ First ▓ │    │ First ▓ │             │    │
│  │  │ Last  ▓ │    │ Last  ▓ │             │    │
│  │  └─────────┘    └─────────┘             │    │
│  └─────────────────────────────────────────┘    │
│                     │                             │
│                     ▼                             │
│  ┌─────────────────────────────────────────┐    │
│  │      ImageFX Integration Layer          │    │
│  │  • Scene → Prompt Converter             │    │
│  │  • Batch Generator                      │    │
│  │  • Cache Manager                        │    │
│  │  • Progress Tracker                     │    │
│  └─────────────────────────────────────────┘    │
│                     │                             │
│                     ▼                             │
│  ┌─────────────────────────────────────────┐    │
│  │      Authentication Manager             │    │
│  │  (Choose Method 1, 2, or 4)             │    │
│  └─────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

## Recommended Setup

### Phase 1: Quick Start (Method 1)
Start with Method 1 (Hidden Browser) for fastest time-to-market.

### Phase 2: Production (Method 4)
Upgrade to Method 4 (Persistent Session) for best user experience.

## Implementation

### 1. Scene Node Data Structure

```typescript
interface SceneNode {
    id: string;
    name: string;
    description: string;
    firstFrame: {
        imagePath: string | null;
        prompt: string | null;
        generating: boolean;
    };
    lastFrame: {
        imagePath: string | null;
        prompt: string | null;
        generating: boolean;
    };
    // Existing properties
    connections: string[];
    position: { x: number; y: number };
}
```

### 2. ImageFX Service Integration

```typescript
// src/services/StoryFramerImageFX.ts
import { ImageFX } from '@rohitaryal/imagefx-api';
import { PersistentAccountManager } from '../auth/PersistentAccountManager';
import { app } from 'electron';
import * as path from 'path';

export class StoryFramerImageFX {
    private accountManager: PersistentAccountManager;
    private imageFX: ImageFX | null = null;
    private generationQueue: Map<string, Promise<any>> = new Map();

    constructor() {
        this.accountManager = new PersistentAccountManager();
    }

    async initialize(): Promise<void> {
        const { cookies } = await this.accountManager.initialize();
        this.imageFX = new ImageFX(cookies);
    }

    /**
     * Generate both frames for a scene
     */
    async generateSceneFrames(scene: SceneNode): Promise<{
        firstFrame: string;
        lastFrame: string;
    }> {
        if (!this.imageFX) {
            throw new Error('ImageFX not initialized');
        }

        // Generate prompts
        const firstPrompt = this.createFirstFramePrompt(scene);
        const lastPrompt = this.createLastFramePrompt(scene);

        // Generate both in parallel
        const [firstResult, lastResult] = await Promise.all([
            this.generateFrame(scene.id + '_first', firstPrompt),
            this.generateFrame(scene.id + '_last', lastPrompt)
        ]);

        return {
            firstFrame: firstResult,
            lastFrame: lastResult
        };
    }

    /**
     * Generate a single frame (first or last)
     */
    async generateSingleFrame(
        sceneId: string, 
        description: string, 
        type: 'first' | 'last'
    ): Promise<string> {
        if (!this.imageFX) {
            throw new Error('ImageFX not initialized');
        }

        const prompt = type === 'first' 
            ? this.createFirstFramePrompt({ description } as any)
            : this.createLastFramePrompt({ description } as any);

        return await this.generateFrame(`${sceneId}_${type}`, prompt);
    }

    /**
     * Generate variations of a frame
     */
    async generateVariations(
        description: string, 
        count: number = 4
    ): Promise<string[]> {
        if (!this.imageFX) {
            throw new Error('ImageFX not initialized');
        }

        const prompt = `Cinematic storyboard frame: ${description}, 
                        professional cinematography, detailed scene, high quality`;

        const images = await this.imageFX.generateImage(prompt);
        
        const savePath = path.join(
            app.getPath('userData'), 
            'storyframer', 
            'variations'
        );

        return images.map(img => img.save(savePath));
    }

    /**
     * Create prompt for first frame
     */
    private createFirstFramePrompt(scene: SceneNode | { description: string }): string {
        return `Cinematic storyboard establishing shot: ${scene.description}. 
                First frame of the scene, setting up the environment and mood.
                Professional cinematography, detailed composition, 
                clear establishing shot, high quality render, 
                storyboard style, cinematic lighting`;
    }

    /**
     * Create prompt for last frame
     */
    private createLastFramePrompt(scene: SceneNode | { description: string }): string {
        return `Cinematic storyboard closing shot: ${scene.description}. 
                Final frame of the scene, showing resolution and transition point.
                Professional cinematography, detailed composition,
                clear concluding moment, high quality render,
                storyboard style, cinematic lighting`;
    }

    /**
     * Internal method to generate and cache
     */
    private async generateFrame(cacheKey: string, prompt: string): Promise<string> {
        // Check if already generating
        if (this.generationQueue.has(cacheKey)) {
            return await this.generationQueue.get(cacheKey)!;
        }

        // Create generation promise
        const generationPromise = (async () => {
            try {
                const images = await this.imageFX!.generateImage(prompt);
                
                const savePath = path.join(
                    app.getPath('userData'),
                    'storyframer',
                    'scenes'
                );

                return images[0].save(savePath);
            } finally {
                this.generationQueue.delete(cacheKey);
            }
        })();

        this.generationQueue.set(cacheKey, generationPromise);
        return await generationPromise;
    }

    /**
     * Check if authenticated
     */
    isAuthenticated(): boolean {
        return this.accountManager.isAuthenticated();
    }
}
```

### 3. Main Process Integration

```typescript
// main.ts
import { app, BrowserWindow, ipcMain } from 'electron';
import { StoryFramerImageFX } from './services/StoryFramerImageFX';
import path from 'path';

let mainWindow: BrowserWindow | null = null;
let imageFXService: StoryFramerImageFX;

async function createWindow() {
    mainWindow = new BrowserWindow({
        width: 1400,
        height: 900,
        webPreferences: {
            preload: path.join(__dirname, 'preload.js'),
            contextIsolation: true,
            nodeIntegration: false
        }
    });

    mainWindow.loadFile('index.html');

    // Initialize ImageFX service
    imageFXService = new StoryFramerImageFX();

    // Auto-initialize if authenticated
    if (imageFXService.isAuthenticated()) {
        try {
            await imageFXService.initialize();
            console.log('ImageFX ready');
        } catch (error) {
            console.error('Failed to initialize ImageFX:', error);
        }
    }
}

// IPC Handlers
ipcMain.handle('imagefx:init', async () => {
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

ipcMain.handle('imagefx:generateSceneFrames', async (event, scene: SceneNode) => {
    try {
        const frames = await imageFXService.generateSceneFrames(scene);
        return { success: true, frames };
    } catch (error) {
        return { 
            success: false, 
            error: error instanceof Error ? error.message : 'Unknown error' 
        };
    }
});

ipcMain.handle('imagefx:generateSingleFrame', async (
    event, 
    sceneId: string, 
    description: string, 
    type: 'first' | 'last'
) => {
    try {
        const imagePath = await imageFXService.generateSingleFrame(
            sceneId, 
            description, 
            type
        );
        return { success: true, imagePath };
    } catch (error) {
        return { 
            success: false, 
            error: error instanceof Error ? error.message : 'Unknown error' 
        };
    }
});

ipcMain.handle('imagefx:generateVariations', async (
    event, 
    description: string, 
    count: number
) => {
    try {
        const variations = await imageFXService.generateVariations(description, count);
        return { success: true, variations };
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

app.whenReady().then(createWindow);
```

### 4. Preload Script

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('storyframerImageFX', {
    init: () => ipcRenderer.invoke('imagefx:init'),
    generateSceneFrames: (scene: any) => 
        ipcRenderer.invoke('imagefx:generateSceneFrames', scene),
    generateSingleFrame: (sceneId: string, description: string, type: 'first' | 'last') =>
        ipcRenderer.invoke('imagefx:generateSingleFrame', sceneId, description, type),
    generateVariations: (description: string, count: number) =>
        ipcRenderer.invoke('imagefx:generateVariations', description, count),
    isAuthenticated: () => ipcRenderer.invoke('imagefx:isAuthenticated')
});
```

### 5. React Component for Scene Nodes

```tsx
// components/SceneNode.tsx
import React, { useState } from 'react';

interface SceneNodeComponentProps {
    node: SceneNode;
    onUpdate: (node: SceneNode) => void;
}

export function SceneNodeComponent({ node, onUpdate }: SceneNodeComponentProps) {
    const [generating, setGenerating] = useState(false);
    const [progress, setProgress] = useState('');

    const handleGenerateBoth = async () => {
        setGenerating(true);
        setProgress('Generating frames...');

        try {
            const result = await window.storyframerImageFX.generateSceneFrames(node);

            if (result.success) {
                onUpdate({
                    ...node,
                    firstFrame: {
                        ...node.firstFrame,
                        imagePath: result.frames.firstFrame
                    },
                    lastFrame: {
                        ...node.lastFrame,
                        imagePath: result.frames.lastFrame
                    }
                });
                setProgress('✓ Complete!');
            } else {
                setProgress('✗ Failed: ' + result.error);
            }
        } catch (error) {
            setProgress('✗ Error: ' + (error as Error).message);
        } finally {
            setGenerating(false);
        }
    };

    const handleGenerateSingle = async (type: 'first' | 'last') => {
        setGenerating(true);
        setProgress(`Generating ${type} frame...`);

        try {
            const result = await window.storyframerImageFX.generateSingleFrame(
                node.id,
                node.description,
                type
            );

            if (result.success) {
                onUpdate({
                    ...node,
                    [type + 'Frame']: {
                        ...node[type + 'Frame'],
                        imagePath: result.imagePath
                    }
                });
                setProgress('✓ Complete!');
            } else {
                setProgress('✗ Failed: ' + result.error);
            }
        } catch (error) {
            setProgress('✗ Error: ' + (error as Error).message);
        } finally {
            setGenerating(false);
        }
    };

    return (
        <div className="scene-node">
            <div className="scene-header">
                <h3>{node.name}</h3>
                <p className="description">{node.description}</p>
            </div>

            <div className="frames-container">
                <div className="frame first-frame">
                    <h4>First Frame</h4>
                    {node.firstFrame.imagePath ? (
                        <img 
                            src={`file://${node.firstFrame.imagePath}`} 
                            alt="First frame"
                        />
                    ) : (
                        <div className="placeholder">
                            <span>No image</span>
                        </div>
                    )}
                    <button 
                        onClick={() => handleGenerateSingle('first')}
                        disabled={generating}
                    >
                        Generate First Frame
                    </button>
                </div>

                <div className="frame last-frame">
                    <h4>Last Frame</h4>
                    {node.lastFrame.imagePath ? (
                        <img 
                            src={`file://${node.lastFrame.imagePath}`} 
                            alt="Last frame"
                        />
                    ) : (
                        <div className="placeholder">
                            <span>No image</span>
                        </div>
                    )}
                    <button 
                        onClick={() => handleGenerateSingle('last')}
                        disabled={generating}
                    >
                        Generate Last Frame
                    </button>
                </div>
            </div>

            <div className="actions">
                <button 
                    className="generate-both"
                    onClick={handleGenerateBoth}
                    disabled={generating}
                >
                    {generating ? 'Generating...' : 'Generate Both Frames'}
                </button>
            </div>

            {progress && (
                <div className="progress-status">
                    {progress}
                </div>
            )}
        </div>
    );
}
```

### 6. Batch Processing for Multiple Scenes

```typescript
// components/BatchGenerator.tsx
import React, { useState } from 'react';

interface BatchGeneratorProps {
    scenes: SceneNode[];
    onScenesUpdated: (scenes: SceneNode[]) => void;
}

export function BatchGenerator({ scenes, onScenesUpdated }: BatchGeneratorProps) {
    const [processing, setProcessing] = useState(false);
    const [progress, setProgress] = useState({ current: 0, total: 0 });

    const handleBatchGenerate = async () => {
        setProcessing(true);
        setProgress({ current: 0, total: scenes.length });

        const updatedScenes = [...scenes];

        for (let i = 0; i < scenes.length; i++) {
            const scene = scenes[i];
            
            try {
                const result = await window.storyframerImageFX.generateSceneFrames(scene);

                if (result.success) {
                    updatedScenes[i] = {
                        ...scene,
                        firstFrame: {
                            ...scene.firstFrame,
                            imagePath: result.frames.firstFrame
                        },
                        lastFrame: {
                            ...scene.lastFrame,
                            imagePath: result.frames.lastFrame
                        }
                    };
                }
            } catch (error) {
                console.error(`Failed to generate frames for scene ${scene.id}:`, error);
            }

            setProgress({ current: i + 1, total: scenes.length });

            // Rate limiting: wait 2 seconds between scenes
            if (i < scenes.length - 1) {
                await new Promise(resolve => setTimeout(resolve, 2000));
            }
        }

        onScenesUpdated(updatedScenes);
        setProcessing(false);
    };

    return (
        <div className="batch-generator">
            <h3>Batch Frame Generation</h3>
            <p>Generate frames for {scenes.length} scenes</p>

            <button 
                onClick={handleBatchGenerate}
                disabled={processing || scenes.length === 0}
            >
                {processing ? 'Processing...' : 'Generate All Frames'}
            </button>

            {processing && (
                <div className="progress">
                    <div className="progress-bar">
                        <div 
                            className="progress-fill"
                            style={{ 
                                width: `${(progress.current / progress.total) * 100}%` 
                            }}
                        />
                    </div>
                    <span>
                        {progress.current} of {progress.total} completed
                    </span>
                </div>
            )}
        </div>
    );
}
```

## Workflow Integration

### Typical User Flow

1. **Setup (One-time)**:
   - User installs StoryFramer
   - On first launch, prompted to authenticate with Google
   - Authentication stored securely for future use

2. **Creating Scenes**:
   - User creates scene nodes in the canvas
   - Adds description for each scene
   - Connects nodes to form story flow

3. **Generating Frames**:
   - Option 1: Generate individual frames as needed
   - Option 2: Generate both frames for a scene
   - Option 3: Batch generate all scenes at once

4. **Reviewing and Editing**:
   - Generated frames appear in scene nodes
   - User can regenerate if not satisfied
   - Manual image import still available as fallback

## Prompt Engineering for StoryFramer

### Best Practices

```typescript
// Good prompt structure for storyboards
const promptTemplate = `
Cinematic storyboard ${frameType}: ${description}.
${frameContext}.
Professional cinematography, detailed composition,
${specificDetails},
storyboard style, cinematic lighting, high quality render
`;

// Examples:
const firstFrame = `
Cinematic storyboard establishing shot: hero enters dark cave.
First frame showing the cave entrance and surrounding forest.
Professional cinematography, detailed composition,
wide angle, dramatic shadows, mysterious atmosphere,
storyboard style, cinematic lighting, high quality render
`;

const lastFrame = `
Cinematic storyboard closing shot: hero emerges victorious.
Final frame showing triumph and resolution.
Professional cinematography, detailed composition,
medium shot, warm lighting, triumphant mood,
storyboard style, cinematic lighting, high quality render
`;
```

### Context-Aware Prompts

```typescript
function createContextualPrompt(
    scene: SceneNode, 
    type: 'first' | 'last',
    previousScene?: SceneNode,
    nextScene?: SceneNode
): string {
    let prompt = `Cinematic storyboard ${type === 'first' ? 'establishing' : 'closing'} shot: ${scene.description}. `;

    if (type === 'first' && previousScene) {
        prompt += `Continuing from previous scene where ${previousScene.description}. `;
    }

    if (type === 'last' && nextScene) {
        prompt += `Transitioning to next scene where ${nextScene.description}. `;
    }

    prompt += `Professional cinematography, detailed scene, high quality render, storyboard style`;

    return prompt;
}
```

## Performance Optimization

### Image Caching

```typescript
class StoryFramerImageCache {
    private cache = new Map<string, string>();
    
    getCachedImage(sceneId: string, type: 'first' | 'last'): string | null {
        return this.cache.get(`${sceneId}_${type}`) || null;
    }
    
    cacheImage(sceneId: string, type: 'first' | 'last', path: string): void {
        this.cache.set(`${sceneId}_${type}`, path);
    }
    
    clearCache(): void {
        this.cache.clear();
    }
}
```

### Progress Tracking

```typescript
interface GenerationProgress {
    sceneId: string;
    type: 'first' | 'last' | 'both';
    status: 'queued' | 'generating' | 'complete' | 'error';
    progress: number; // 0-100
    error?: string;
}

class ProgressTracker {
    private listeners: Set<(progress: GenerationProgress) => void> = new Set();
    
    subscribe(callback: (progress: GenerationProgress) => void): () => void {
        this.listeners.add(callback);
        return () => this.listeners.delete(callback);
    }
    
    notify(progress: GenerationProgress): void {
        this.listeners.forEach(callback => callback(progress));
    }
}
```

## Testing Checklist

- [ ] Authentication flow works smoothly
- [ ] Single frame generation successful
- [ ] Both frames generation works
- [ ] Batch processing handles multiple scenes
- [ ] Error handling for failed generations
- [ ] Progress indicators update correctly
- [ ] Generated images display properly
- [ ] Session persists between app restarts
- [ ] Performance acceptable with 10+ scenes
- [ ] Memory usage stays reasonable

## Troubleshooting

### Common Issues

**Frames take too long to generate**:
- Check network connection
- Verify ImageFX service status
- Consider reducing batch size

**Authentication fails**:
- Clear stored credentials
- Re-authenticate
- Check Google account status

**Images not displaying**:
- Verify file paths are correct
- Check file permissions
- Ensure image files were saved

## Next Steps

1. Implement basic single-frame generation
2. Add batch processing
3. Implement progress tracking
4. Add image caching
5. Polish UI/UX
6. Add export functionality
7. Implement undo/redo for generations

## Resources

- [Main Integration Guide](../ELECTRON_INTEGRATION_GUIDE.md)
- [Quick Start](./QUICK_START.md)
- [Troubleshooting](./TROUBLESHOOTING.md)
- [Methods Comparison](./METHODS_COMPARISON.md)

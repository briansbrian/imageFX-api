# ImageFX API Research Guide

## Important: This Repository is a Reference Only

This repository (`imagefx-api`) demonstrates **one approach** to accessing ImageFX. For production Electron applications, you should research and implement your own integration using the most current methods and tools available.

## Why Research Your Own Implementation?

1. **API Changes**: Google's internal APIs can change without notice
2. **Best Practices**: New authentication and security patterns emerge
3. **Optimization**: Current methods may be more efficient than what's documented here
4. **Compliance**: Latest security and privacy requirements
5. **Features**: New ImageFX features and capabilities

## Research Tools and Methods

### 1. Browser DevTools (Primary Method)

The most reliable way to understand how ImageFX works:

**Steps**:
1. Open https://labs.google/fx/tools/image-fx in Chrome/Edge
2. Open DevTools (F12)
3. Go to **Network** tab
4. Clear logs (Ctrl+L)
5. Generate an image
6. Observe the requests made

**What to Look For**:
- Authentication endpoints
- Session management URLs
- Image generation API endpoints
- Request headers (cookies, tokens)
- Request body structure
- Response format

**Example Findings** (as of documentation):
```
POST https://aisandbox-pa.googleapis.com/v1:runImageFx
Headers:
  - Authorization: Bearer <token>
  - Cookie: <google cookies>
  - Content-Type: application/json
  
Body:
{
  "prompt": "...",
  "numImages": 1,
  "model": "IMAGEN_3_5",
  "aspectRatio": "IMAGE_ASPECT_RATIO_SQUARE"
}

Response:
{
  "imagePanels": [{
    "generatedImages": [{
      "image": {
        "imageBytes": "<base64>",
        "mediaKey": "...",
        ...
      }
    }]
  }]
}
```

### 2. Context7 / AI Code Assistants

Use AI-powered research tools:

```
Prompt Example:
"Research the current ImageFX API structure from labs.google.
What are the authentication endpoints, generation endpoints,
and request/response formats?"
```

**Benefits**:
- Can aggregate information from multiple sources
- May have up-to-date documentation
- Can explain API changes over time

### 3. MCP Servers

Model Context Protocol servers can provide real-time API information:

```javascript
// Example: Using an MCP server for API research
const mcpClient = new MCPClient();
const apiInfo = await mcpClient.query({
  service: 'google-imagefx',
  query: 'authentication-flow'
});
```

**MCP Servers to Consider**:
- Google API MCP servers
- Web scraping MCP servers
- Documentation aggregation servers

### 4. Reverse Engineering Tools

**Puppeteer/Playwright** for automated research:

```javascript
const { chromium } = require('playwright');

async function researchImageFXAPI() {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  
  // Intercept network requests
  context.route('**/*', route => {
    const request = route.request();
    console.log('URL:', request.url());
    console.log('Method:', request.method());
    console.log('Headers:', request.headers());
    console.log('Body:', request.postData());
    route.continue();
  });
  
  const page = await context.newPage();
  await page.goto('https://labs.google/fx/tools/image-fx');
  
  // Interact with the page to trigger API calls
  // ...
  
  await browser.close();
}
```

### 5. GitHub / Community Research

Search for:
- Similar implementations
- Issue discussions
- Pull requests
- Community documentation

**Search terms**:
- "ImageFX API"
- "labs.google reverse engineering"
- "Google Imagen API unofficial"
- "ImageFX authentication"

## Implementation Checklist

When building your own integration:

### Authentication Research
- [ ] Identify current authentication flow
- [ ] Document required cookies
- [ ] Find session token endpoints
- [ ] Test token refresh mechanism
- [ ] Verify expiration handling

### API Endpoints Research
- [ ] Image generation endpoint URL
- [ ] Request body structure
- [ ] Required headers
- [ ] Response format
- [ ] Error handling

### Session Management Research
- [ ] Cookie structure and requirements
- [ ] Token refresh intervals
- [ ] Session expiration times
- [ ] Re-authentication triggers

### Security Research
- [ ] Current encryption standards
- [ ] Credential storage best practices
- [ ] OAuth2 latest recommendations
- [ ] CORS and security headers

## Current API Structure (Reference)

**Note**: This may be outdated. Always verify with your own research.

### Authentication Endpoints

```typescript
// Session endpoint
GET/POST https://labs.google/fx/api/auth/session
Headers: {
  Cookie: "<google cookies>"
}
Response: {
  access_token: string,
  expires: string,
  user: { email: string, ... }
}

// OAuth endpoints (if available)
GET https://accounts.google.com/o/oauth2/v2/auth
POST https://oauth2.googleapis.com/token
```

### Image Generation Endpoints

```typescript
// Main generation endpoint
POST https://aisandbox-pa.googleapis.com/v1:runImageFx
Headers: {
  Authorization: "Bearer <token>",
  Cookie: "<cookies>",
  Content-Type: "application/json"
}
Body: {
  prompt: string,
  numImages: number,
  model: "IMAGEN_3" | "IMAGEN_3_1" | "IMAGEN_3_5",
  aspectRatio: "IMAGE_ASPECT_RATIO_SQUARE" | "IMAGE_ASPECT_RATIO_PORTRAIT" | "IMAGE_ASPECT_RATIO_LANDSCAPE",
  seed?: number
}

// Fetch existing image
GET https://labs.google/fx/api/trpc/media.fetchMedia?input=...
```

### Image to Text Endpoints

```typescript
POST https://labs.google/fx/api/trpc/backbone.captionImage
Body: {
  json: {
    captionInput: {
      candidatesCount: number,
      mediaInput: {
        mediaCategory: "MEDIA_CATEGORY_SUBJECT",
        rawBytes: string // base64 data URL
      }
    }
  }
}
```

## Monitoring for Changes

### Set Up Monitoring

1. **Automated Testing**:
```javascript
// Run periodically to detect API changes
async function testAPIEndpoints() {
  const endpoints = [
    'https://labs.google/fx/api/auth/session',
    'https://aisandbox-pa.googleapis.com/v1:runImageFx'
  ];
  
  for (const endpoint of endpoints) {
    try {
      const response = await fetch(endpoint);
      console.log(`${endpoint}: ${response.status}`);
    } catch (error) {
      console.error(`${endpoint} failed:`, error);
    }
  }
}
```

2. **Error Tracking**:
```javascript
class APIMonitor {
  trackError(endpoint: string, error: any) {
    // Log to your monitoring service
    console.error(`API Error [${endpoint}]:`, error);
    
    // Alert if error rate increases
    if (this.errorRate > threshold) {
      this.alert('API may have changed!');
    }
  }
}
```

3. **Version Detection**:
```javascript
// Check for API version changes
async function detectAPIVersion() {
  const response = await fetch('https://labs.google/fx/tools/image-fx');
  const html = await response.text();
  
  // Look for version indicators in HTML/JS
  const versionMatch = html.match(/version["\s:]+([0-9.]+)/i);
  
  if (versionMatch && versionMatch[1] !== lastKnownVersion) {
    console.warn('ImageFX version changed:', versionMatch[1]);
  }
}
```

## Best Practices

### 1. Abstract Your API Layer

```typescript
interface ImageGenerationAPI {
  authenticate(): Promise<AuthToken>;
  generateImage(prompt: string, options?: any): Promise<GeneratedImage[]>;
  refreshSession(): Promise<void>;
}

class ImageFXAPIClient implements ImageGenerationAPI {
  // Implementation can be swapped easily when API changes
}
```

### 2. Configuration-Driven

```typescript
const apiConfig = {
  endpoints: {
    session: 'https://labs.google/fx/api/auth/session',
    generate: 'https://aisandbox-pa.googleapis.com/v1:runImageFx',
    caption: 'https://labs.google/fx/api/trpc/backbone.captionImage'
  },
  headers: {
    origin: 'https://labs.google',
    referer: 'https://labs.google/fx/tools/image-fx'
  },
  models: ['IMAGEN_3', 'IMAGEN_3_1', 'IMAGEN_3_5']
};
```

### 3. Comprehensive Error Handling

```typescript
class APIError extends Error {
  constructor(
    message: string,
    public endpoint: string,
    public statusCode?: number,
    public response?: any
  ) {
    super(message);
  }
}

async function callAPI(endpoint: string, options: RequestInit) {
  try {
    const response = await fetch(endpoint, options);
    
    if (!response.ok) {
      throw new APIError(
        'API call failed',
        endpoint,
        response.status,
        await response.text()
      );
    }
    
    return await response.json();
  } catch (error) {
    // Log for debugging
    logAPIError(endpoint, error);
    throw error;
  }
}
```

### 4. Testing Infrastructure

```typescript
// Mock API for testing
class MockImageFXAPI implements ImageGenerationAPI {
  async authenticate() {
    return { token: 'mock-token', expires: Date.now() + 3600000 };
  }
  
  async generateImage(prompt: string) {
    return [{ imageData: 'mock-data', mediaID: 'mock-id' }];
  }
}

// Use in tests
const api = process.env.NODE_ENV === 'test' 
  ? new MockImageFXAPI() 
  : new ImageFXAPIClient();
```

## Community Resources

### Stay Updated

- **GitHub Issues**: Watch this repo for API change reports
- **Discord/Slack**: Join communities discussing ImageFX
- **Stack Overflow**: Search for "ImageFX API" questions
- **Google Groups**: AI/ML tool discussions
- **Twitter/X**: Follow Google AI announcements

### Contributing Back

If you discover API changes or improvements:

1. Document your findings
2. Share with the community
3. Update this repository (if appropriate)
4. Help others avoid the same research

## Example: Complete Research Session

```bash
# 1. Open ImageFX in browser with DevTools
# 2. Generate an image and observe network traffic
# 3. Document findings:

## Authentication Flow (Verified: 2024-11-11)
- GET https://labs.google/fx/tools/image-fx
- Redirects to Google OAuth if not logged in
- POST https://labs.google/fx/api/auth/session
  - Response includes access_token, expires timestamp
  
## Image Generation (Verified: 2024-11-11)
- POST https://aisandbox-pa.googleapis.com/v1:runImageFx
- Headers:
  - Authorization: Bearer {token}
  - Cookie: {google cookies}
- Body includes: prompt, numImages, model, aspectRatio
- Response includes base64 image data

## Changes from Previous Version
- Model names updated (IMAGEN_3_5 added)
- New aspectRatio option added
- Response format unchanged

## Next Review: 2024-12-11
```

## Conclusion

This repository provides a **reference** for how ImageFX integration works. Your production implementation should:

1. ✅ Research current API structure
2. ✅ Implement based on latest findings
3. ✅ Monitor for changes
4. ✅ Use modern tools (Context7, MCP servers)
5. ✅ Abstract your API layer for easy updates
6. ✅ Test thoroughly
7. ✅ Share findings with the community

**Remember**: The code examples in this repository show patterns and approaches, not necessarily the current API structure. Always verify with your own research!

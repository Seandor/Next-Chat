# Edge Runtime GET Request with Content-Type Header Fix

## Problem

SearXNG plugin search requests were failing with 500 errors on Azure deployment, but working locally.

**Error**: `AxiosError: Request failed with status code 500`

## Root Cause

The issue occurs when a GET request is sent with a `Content-Type: application/json` header to an API route using Edge Runtime.

### Why it happens

1. **Axios default behavior**: Axios sets `Content-Type: application/json` for all requests by default, even GET requests
2. **Edge Runtime strictness**: When Edge Runtime receives a GET request with `Content-Type: application/json`, it attempts to parse the request body
3. **No body on GET**: GET requests don't have a body, causing Edge Runtime to fail before the handler code executes
4. **Result**: 500 Internal Server Error

### Why it works locally but fails on Azure

| Environment | Runtime | Behavior |
|-------------|---------|----------|
| Local (`yarn dev`) | Node.js | Lenient, ignores Content-Type on GET |
| Azure (standalone) | Edge Runtime (for API routes with `runtime = "edge"`) | Strict, fails on Content-Type + GET |

## Solution

### Client-side fix (Primary)

In `app/store/plugin.ts`, added `transformRequest` to remove `Content-Type` header when there's no request body:

```typescript
const api = new OpenAPIClientAxios({
  definition: yaml.load(plugin.content) as any,
  axiosConfigDefaults: {
    adapter: (window.__TAURI__ ? adapter : ["xhr"]) as any,
    baseURL,
    headers,
    // Remove Content-Type for GET/HEAD requests to avoid Edge Runtime issues
    transformRequest: [
      (data, headers) => {
        if (headers && !data) {
          delete headers["Content-Type"];
        }
        return data;
      },
    ],
  },
});
```

### Server-side fix (Defensive)

In `app/api/proxy.ts`, added defensive handling for GET/HEAD requests:

```typescript
const isBodylessMethod = req.method === "GET" || req.method === "HEAD";

// Skip content-type header for bodyless methods
const skipHeaders = isBodylessMethod
  ? ["connection", "host", "origin", "referer", "cookie", "content-type"]
  : ["connection", "host", "origin", "referer", "cookie"];

// Don't pass body for bodyless methods
const fetchOptions: RequestInit = {
  headers,
  method: req.method,
  body: isBodylessMethod ? undefined : req.body,
  // ...
};
```

**Note**: The server-side fix alone doesn't solve the problem because the Edge Runtime error occurs *before* the handler code executes. It serves as a defensive layer.

## Testing

### Browser test (should work)
Use the SearXNG plugin in the NextChat UI - searches should return results.

### curl test (will still fail)
```bash
# This will fail with 500 because curl sends Content-Type directly
curl "https://nextchat.siyuan0.tech/api/proxy/search?q=test&format=json" \
  -H "X-Base-URL: https://searxng.yellowmushroom-1df91506.eastasia.azurecontainerapps.io" \
  -H "Content-Type: application/json"

# This will succeed (no Content-Type header)
curl "https://nextchat.siyuan0.tech/api/proxy/search?q=test&format=json" \
  -H "X-Base-URL: https://searxng.yellowmushroom-1df91506.eastasia.azurecontainerapps.io"
```

## Alternative Solutions (Not Implemented)

1. **Remove Edge Runtime**: Change `export const runtime = "edge"` to use Node.js runtime instead. This would fix curl tests but may affect performance/compatibility with other providers.

2. **Middleware**: Add Next.js middleware to strip Content-Type from GET requests before they reach the API route.

## Related Files

- `app/store/plugin.ts` - Client-side axios configuration
- `app/api/proxy.ts` - Server-side proxy handler
- `app/api/[provider]/[...path]/route.ts` - API route with Edge Runtime config

## References

- [Next.js Edge Runtime](https://nextjs.org/docs/app/api-reference/edge)
- [Axios transformRequest](https://axios-http.com/docs/req_config)

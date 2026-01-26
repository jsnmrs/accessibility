# Plan: Migrating from Axios to Native Fetch

## Overview

This document outlines a comprehensive plan for replacing the axios HTTP client library with the native `fetch` API. The native fetch API is now available in Node.js (v18+) and all modern browsers, making axios no longer necessary for most use cases.

## Current State Analysis

In this repository, axios is a **transitive dependency** of the `evaluatory` package (v4.1.0). If custom HTTP client code were to be added, or if this plan is applied to another codebase using axios directly, the following migration steps apply.

---

## Migration Plan

### Phase 1: Inventory & Assessment

#### Step 1.1: Identify All Axios Usage
```bash
# Find all files importing/requiring axios
grep -r "require('axios')" --include="*.js" --include="*.ts"
grep -r "from 'axios'" --include="*.js" --include="*.ts"
grep -r 'from "axios"' --include="*.js" --include="*.ts"
```

#### Step 1.2: Document Usage Patterns
Create an inventory of:
- [ ] GET requests
- [ ] POST requests
- [ ] PUT/PATCH requests
- [ ] DELETE requests
- [ ] Request interceptors
- [ ] Response interceptors
- [ ] Custom axios instances
- [ ] Request/response transformers
- [ ] Timeout configurations
- [ ] Cancel tokens / AbortController usage
- [ ] Base URL configurations
- [ ] Default headers

### Phase 2: Create Fetch Wrapper (Optional)

For complex codebases, create a wrapper to maintain consistent API patterns.

#### Step 2.1: Basic Fetch Wrapper

```javascript
// lib/http-client.js

class HttpClient {
  constructor(baseURL = '', defaultHeaders = {}) {
    this.baseURL = baseURL;
    this.defaultHeaders = {
      'Content-Type': 'application/json',
      ...defaultHeaders
    };
  }

  async request(url, options = {}) {
    const fullURL = this.baseURL + url;
    const config = {
      ...options,
      headers: {
        ...this.defaultHeaders,
        ...options.headers
      }
    };

    const response = await fetch(fullURL, config);

    if (!response.ok) {
      const error = new Error(`HTTP ${response.status}: ${response.statusText}`);
      error.response = response;
      error.status = response.status;
      throw error;
    }

    const contentType = response.headers.get('content-type');
    if (contentType && contentType.includes('application/json')) {
      return response.json();
    }
    return response.text();
  }

  get(url, options = {}) {
    return this.request(url, { ...options, method: 'GET' });
  }

  post(url, data, options = {}) {
    return this.request(url, {
      ...options,
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  put(url, data, options = {}) {
    return this.request(url, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(data)
    });
  }

  patch(url, data, options = {}) {
    return this.request(url, {
      ...options,
      method: 'PATCH',
      body: JSON.stringify(data)
    });
  }

  delete(url, options = {}) {
    return this.request(url, { ...options, method: 'DELETE' });
  }
}

export default HttpClient;
```

### Phase 3: API Conversion Reference

#### Step 3.1: Simple GET Request

**Axios:**
```javascript
const response = await axios.get('/api/users');
const data = response.data;
```

**Fetch:**
```javascript
const response = await fetch('/api/users');
const data = await response.json();
```

#### Step 3.2: GET Request with Query Parameters

**Axios:**
```javascript
const response = await axios.get('/api/users', {
  params: { page: 1, limit: 10 }
});
```

**Fetch:**
```javascript
const params = new URLSearchParams({ page: 1, limit: 10 });
const response = await fetch(`/api/users?${params}`);
const data = await response.json();
```

#### Step 3.3: POST Request with JSON Body

**Axios:**
```javascript
const response = await axios.post('/api/users', {
  name: 'John',
  email: 'john@example.com'
});
```

**Fetch:**
```javascript
const response = await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'John', email: 'john@example.com' })
});
const data = await response.json();
```

#### Step 3.4: Custom Headers

**Axios:**
```javascript
const response = await axios.get('/api/data', {
  headers: {
    'Authorization': 'Bearer token123',
    'X-Custom-Header': 'value'
  }
});
```

**Fetch:**
```javascript
const response = await fetch('/api/data', {
  headers: {
    'Authorization': 'Bearer token123',
    'X-Custom-Header': 'value'
  }
});
```

#### Step 3.5: Error Handling

**Axios:**
```javascript
try {
  const response = await axios.get('/api/data');
  return response.data;
} catch (error) {
  if (error.response) {
    // Server responded with error status
    console.error('Status:', error.response.status);
    console.error('Data:', error.response.data);
  } else if (error.request) {
    // Request made but no response
    console.error('No response received');
  } else {
    // Request setup error
    console.error('Error:', error.message);
  }
}
```

**Fetch:**
```javascript
try {
  const response = await fetch('/api/data');

  if (!response.ok) {
    // Server responded with error status (fetch doesn't throw on HTTP errors)
    const errorData = await response.json().catch(() => ({}));
    console.error('Status:', response.status);
    console.error('Data:', errorData);
    throw new Error(`HTTP ${response.status}`);
  }

  return await response.json();
} catch (error) {
  if (error.name === 'TypeError') {
    // Network error (no response received)
    console.error('Network error:', error.message);
  } else {
    console.error('Error:', error.message);
  }
}
```

#### Step 3.6: Request Timeout

**Axios:**
```javascript
const response = await axios.get('/api/data', {
  timeout: 5000 // 5 seconds
});
```

**Fetch:**
```javascript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch('/api/data', {
    signal: controller.signal
  });
  clearTimeout(timeoutId);
  return await response.json();
} catch (error) {
  if (error.name === 'AbortError') {
    throw new Error('Request timeout');
  }
  throw error;
}
```

#### Step 3.7: Request Cancellation

**Axios (Cancel Token - deprecated):**
```javascript
const source = axios.CancelToken.source();
axios.get('/api/data', { cancelToken: source.token });
source.cancel('Request cancelled');
```

**Axios (AbortController) / Fetch:**
```javascript
const controller = new AbortController();

fetch('/api/data', { signal: controller.signal });
// or
axios.get('/api/data', { signal: controller.signal });

// To cancel:
controller.abort();
```

#### Step 3.8: Uploading Files (FormData)

**Axios:**
```javascript
const formData = new FormData();
formData.append('file', fileInput.files[0]);

const response = await axios.post('/api/upload', formData, {
  headers: { 'Content-Type': 'multipart/form-data' }
});
```

**Fetch:**
```javascript
const formData = new FormData();
formData.append('file', fileInput.files[0]);

// Note: Don't set Content-Type header - browser sets it with boundary
const response = await fetch('/api/upload', {
  method: 'POST',
  body: formData
});
```

#### Step 3.9: Interceptors Replacement

**Axios Interceptors:**
```javascript
axios.interceptors.request.use(config => {
  config.headers.Authorization = `Bearer ${getToken()}`;
  return config;
});

axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response.status === 401) {
      redirectToLogin();
    }
    return Promise.reject(error);
  }
);
```

**Fetch Wrapper with Interceptor-like Behavior:**
```javascript
async function fetchWithAuth(url, options = {}) {
  // Request "interceptor"
  const headers = {
    ...options.headers,
    'Authorization': `Bearer ${getToken()}`
  };

  const response = await fetch(url, { ...options, headers });

  // Response "interceptor"
  if (response.status === 401) {
    redirectToLogin();
    throw new Error('Unauthorized');
  }

  return response;
}
```

### Phase 4: Migration Steps

#### Step 4.1: Update Node.js Version
Ensure Node.js v18+ is being used for native fetch support.

```json
// package.json
{
  "engines": {
    "node": ">=18.0.0"
  }
}
```

#### Step 4.2: Create Feature Branch
```bash
git checkout -b feature/axios-to-fetch-migration
```

#### Step 4.3: Migrate Files Incrementally
1. Start with utility/helper files
2. Move to service/API layer files
3. Update components/controllers last
4. Run tests after each file migration

#### Step 4.4: Update Tests
- Update mocks from axios mocks to fetch mocks
- Consider using `msw` (Mock Service Worker) for API mocking

**Jest Fetch Mock Example:**
```javascript
// Before (axios)
jest.mock('axios');
axios.get.mockResolvedValue({ data: { id: 1 } });

// After (fetch)
global.fetch = jest.fn(() =>
  Promise.resolve({
    ok: true,
    json: () => Promise.resolve({ id: 1 })
  })
);
```

#### Step 4.5: Remove Axios Dependency
```bash
npm uninstall axios
```

#### Step 4.6: Update Documentation
- Update README with new HTTP client patterns
- Document any custom fetch wrapper API

### Phase 5: Testing Checklist

- [ ] All GET requests work correctly
- [ ] All POST/PUT/PATCH requests send correct body
- [ ] Error handling catches HTTP errors (4xx, 5xx)
- [ ] Network errors are handled properly
- [ ] Timeouts work as expected
- [ ] Request cancellation works
- [ ] File uploads work correctly
- [ ] Authentication headers are sent
- [ ] Query parameters are correctly encoded
- [ ] Response parsing (JSON, text, blob) works

### Phase 6: Key Differences to Remember

| Feature | Axios | Fetch |
|---------|-------|-------|
| Response data | `response.data` | `await response.json()` |
| HTTP errors | Throws automatically | Must check `response.ok` |
| Request body | `{ data: obj }` | `{ body: JSON.stringify(obj) }` |
| Query params | `{ params: obj }` | Use `URLSearchParams` |
| Timeout | `{ timeout: ms }` | Use `AbortController` |
| Interceptors | Built-in | Custom wrapper needed |
| Progress | Built-in | Use Streams API |

---

## Rollback Plan

If issues arise during migration:

1. Keep axios in `devDependencies` until fully migrated
2. Use feature flags to toggle between axios and fetch
3. Maintain the ability to revert individual files via git

---

## Benefits of Migration

1. **Reduced bundle size** - No external HTTP library needed
2. **Native browser/Node.js support** - fetch is built-in
3. **Fewer dependencies** - Less maintenance burden
4. **Standards-compliant** - Uses web platform standards
5. **Better tree-shaking** - Only import what you use

---

## When to Keep Axios

Consider keeping axios if you need:
- Automatic request/response transformation
- Built-in progress events for uploads/downloads
- Older browser support without polyfills
- Simpler API for complex interceptor chains

---

## Resources

- [MDN Fetch API Documentation](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Node.js Fetch Documentation](https://nodejs.org/dist/latest/docs/api/globals.html#fetch)
- [AbortController Documentation](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)

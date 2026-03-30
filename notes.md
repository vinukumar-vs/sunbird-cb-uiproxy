# Technical Debt & Optimization Tasks

## 📋 Overview
This document outlines identified technical debt items, performance optimizations, and code cleanup tasks for the Sunbird CB UI Proxy application.

---

## 🧹 Code Cleanup Tasks

### 1. Logging Dependencies
**Issue:** Redundant logging library  
**Description:** The Morgan logging middleware can be removed since we've migrated to Pino logger for all logging operations.

**Action Required:**
- Remove Morgan dependency from package.json
- Remove Morgan middleware configuration
- Verify all logging is handled by Pino

---

### 2. Middleware Optimizations

#### 2.1 Compression Middleware
**Location:** `server.ts -> configureMiddleware()`  
**Issue:** Inefficient compression logic  
**Current State:** All responses are compressed regardless of size  
**Optimization:** Only compress responses greater than 5KB to improve performance

**Benefits:**
- Reduced CPU overhead for small responses
- Better performance for frequent small API calls

#### 2.2 Health Check Endpoint
**Issue:** Incorrect HTTP method implementation  
**Current State:** Health check is implemented as middleware function via `app.use`  
**Required Change:** Convert to proper **GET** method endpoint

**Implementation:**
```javascript
// Change from: app.use('/health', healthMiddleware)
// To: app.get('/health', healthController)
```

#### 2.3 Static Route Optimization & Duplicate Logic Removal
**Location:** `apiWhiteList.ts` and `server.ts`  
**Issue:** Multiple performance bottlenecks in static file handling

**Current Problems:**
1. **Redundant Route Validation**: Static route checks are performed multiple times across different middleware layers
2. **Inefficient Static File Serving**: Static files are processed through all middleware instead of being served directly
3. **Performance Overhead**: Each static file request goes through authentication and validation checks unnecessarily

**Root Cause Analysis:**
| Component | Current Behavior | Impact |
|-----------|-----------------|---------|
| `apiWhiteListLogger()` | Checks for static routes in `app.all('*')` | ⚠️ First redundant check |
| `isAllowed()` method | Duplicates the same static route check | ⚠️ Second redundant check |
| `shouldAllow()` function | Additional validation layer for same routes | ⚠️ Third redundant check |
| Static file serving | Routes through middleware instead of `express.static` | 🔴 Major performance impact |

**Recommended Solution:**

**Phase 1: Implement Express Static Middleware**
```javascript
// Add this at the beginning of middleware configuration (server.ts)
// This should be placed BEFORE session and other middleware to avoid unnecessary processing

// Static assets with optimized caching
app.use('/assets', express.static(path.join(__dirname, 'assets'), {
  maxAge: '1d',        // 1 day cache for production assets
  etag: true,          // Enable ETag headers for caching
  lastModified: true,  // Enable Last-Modified headers
  immutable: true      // Mark assets as immutable for better caching
}));

// Resource files
app.use('/resource', express.static(path.join(__dirname, 'resource'), {
  maxAge: '1h',        // Shorter cache for dynamic resources
  etag: true,
  lastModified: true
}));
```

**Phase 2: Remove Redundant Validation Code**
```javascript
// REMOVE this entire block from apiWhiteList.ts -> isAllowed()
if (shouldAllow(req) || _.includes(REQ_URL, '/resource') || _.includes(REQ_URL, '/eclogin')) {
    logDebug('Path : ' + REQ_URL + ' is in excluded list.')
    next()
}
```

**Phase 3: Update Middleware Order**
```javascript
// Optimized middleware configuration order in server.ts
class Server {
  private configureMiddleware() {
    // 1. Static files FIRST (bypasses all other middleware)
    this.setupStaticRoutes();
    
    // 2. Then configure other middleware
    const sessionConfig = getSessionConfig();
    this.app.use(expressSession(sessionConfig));
    this.app.use(express.urlencoded({ extended: false, limit: '50mb' }));
    this.app.use(express.json({ limit: '50mb' }));
    this.setCookie();
    
    // 3. API middleware last
    this.app.all('*', apiWhiteListLogger());
  }
  
  private setupStaticRoutes() {
    // Static routes configuration here
  }
}
```

**Implementation Benefits:**
- ⚡ **Performance**: Static files served directly, skipping middleware stack
- 🔄 **Caching**: Proper HTTP caching headers for better browser performance  
- 🧹 **Code Cleanup**: Removes ~30 lines of redundant validation code
- 📊 **Monitoring**: Cleaner request logs (static files won't appear in API logs)
- 🛡️ **Security**: Static files bypass authentication middleware appropriately

**Action Items:**
1. Add `express.static` middleware configuration at the top of middleware stack
2. Remove redundant validation logic from `apiWhiteList.ts`
3. Update `excludePath` list in `checkIsStaticRoute()` method
4. Test static file serving performance before and after changes
5. Update middleware order documentation


#### 2.4 File Upload Configuration
**Location:** `server.ts -> configureMiddleware() -> this.app.use(fileUpload())`  
**Issue:** Suboptimal file upload configuration

**Current Issues:**
- No file size limits configured
- Using memory store (inefficient for large files)

**Recommended Changes:**
- Implement Busboy properties for max file size limits
- Switch from memory store to temporary file store
- Reference: [express-fileupload documentation](https://www.npmjs.com/package/express-fileupload)

**Benefits:**
- Better memory management
- Configurable file size limits
- Improved handling of large file uploads

---

### 3. Authentication & Security

#### 3.1 Keycloak Protection Inconsistency
**Location:** `authoringApi()` method  
**Issue:** Inconsistent Keycloak protection implementation

**Current State:**
```javascript
private authoringApi() {
    if (this.keycloak) {
        this.app.use('/authSearchApi', this.keycloak.protect, authSearch)  // ✅ Protected
        this.app.use('/authApi', authApi)                                   // ❌ Unprotected
    }
}
```

**Question:** Why is `/authApi` not protected by `keycloak.protect`?  
**Action Required:** 
- Investigate if this is intentional or oversight
- Document the reason or apply protection consistently

---

## 🗂️ Unused Configurations

### Configuration Cleanup
The following configuration variables are defined but not actively used in the application:

| Configuration | Status | Action Required |
|---------------|---------|-----------------|
| `APP_CONFIGURATIONS` | Not in use | Remove or document purpose |
| `APP_LOGS` | Not in use | Remove or document purpose |

**Recommendation:** 
- Audit codebase to confirm these are truly unused
- Remove unused configurations to reduce complexity
- Document any configurations that are reserved for future use

---

## 📈 Priority Matrix

| Task | Priority | Impact | Effort |
|------|----------|--------|--------|
| Remove duplicate static route checks | High | Medium | Low |
| Fix health endpoint method | High | Low | Low |
| Optimize compression middleware | Medium | Medium | Medium |
| Review Keycloak protection | High | High | Low |
| Update file upload config | Medium | Medium | Medium |
| Remove unused configs | Low | Low | Low |
| Remove Morgan dependency | Low | Low | Low |

---

## 📝 Notes
- All changes should be tested thoroughly in development environment
- Consider backward compatibility when modifying middleware
- Document any breaking changes for deployment teams


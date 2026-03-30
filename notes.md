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

#### 2.3 Static Route Validation (Duplicate Logic)
**Location:** `apiWhiteList.ts`  
**Issue:** Redundant static route checks causing performance overhead

**Problem Analysis:**
1. `apiWhiteListLogger()` method checks for static routes in `app.all('*')`
2. Same check is duplicated in `app.all('*', isAllowed())`
3. `shouldAllow()` check in `isAllowed()` method is redundant

**Solution:**
- Remove duplicate `shouldAllow()` check from `isAllowed()` method
- Consolidate static route validation logic
- Add `/resource` and `/eclogin` directly to `checkIsStaticRoute() -> excludePath` list

**Code to Remove:**
```javascript
// Remove this entire condition from apiWhiteList.ts -> isAllowed()
if (shouldAllow(req) || _.includes(REQ_URL, '/resource') || _.includes(REQ_URL, '/eclogin')) {
    logDebug('Path : ' + REQ_URL + ' is in excluded list.')
    next()
}
```

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


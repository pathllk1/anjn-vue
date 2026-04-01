# Authentication Cookie Persistence Bug Report

## Bug Summary
Users are required to log in again when closing and reopening the browser, despite the application using cookies for authentication. This indicates that authentication cookies are not persisting across browser sessions as expected.

## Severity
**High** - This bug significantly impacts user experience by forcing users to re-authenticate every time they close their browser.

## Environment
- **Frontend**: Vue.js 3 (beta) with TypeScript
- **State Management**: Pinia
- **Build Tool**: Vite
- **UI Framework**: Nuxt UI
- **API Proxy**: Vite dev server proxy to localhost:3000

## Root Cause Analysis

### 1. Authentication Flow Issues

#### Problem Identified:
The application relies solely on server-side session cookies but has several critical issues:

1. **Missing Persistent Cookie Configuration**: The API configuration in `vite.config.ts` includes cookie rewriting but doesn't ensure cookies have proper persistence attributes.

2. **Incomplete Client-side Initialization**: The `App.vue` component calls `auth.fetchUser()` on mount, but this only checks if the user is currently authenticated, not if persistent authentication should be restored.

3. **No "Remember Me" Implementation**: While the login form has a "Remember me" checkbox (`form.remember`), this value is not being used in the login request to set persistent cookie flags.

### 2. Technical Details

#### Files Involved:
- `src/stores/auth.ts` - Authentication state management
- `src/utils/api.ts` - API client with cookie handling
- `src/views/LoginView.vue` - Login form with unused "Remember me" feature
- `src/App.vue` - App initialization and user fetching
- `vite.config.ts` - Proxy configuration

#### Key Issues:

**In `src/views/LoginView.vue`:**
```typescript
// Line 24-28: Login request doesn't use the "remember" flag
const response = await api.post('/auth/login', {
  username: form.username,
  password: form.password,
  device_id: window.navigator.userAgent // Missing: remember: form.remember
})
```

**In `src/utils/api.ts`:**
```typescript
// Line 54: Credentials are included, but no cookie persistence control
credentials: 'include',
```

**In `src/App.vue`:**
```typescript
// Line 57: Only fetches current user, no persistent session restoration
auth.fetchUser()
```

### 3. Cookie Configuration Issues

The Vite proxy configuration in `vite.config.ts` has cookie rewriting but may not be sufficient for persistent cookies:

```typescript
proxy: {
  '/api': {
    target: 'http://localhost:3000',
    changeOrigin: true,
    secure: false,
    cookieDomainRewrite: "localhost",
    cookiePathRewrite: "/",
  }
}
```

## Expected Behavior
1. When users check "Remember me" and log in, authentication cookies should persist across browser sessions
2. When users reopen the browser, they should remain logged in (if "Remember me" was selected)
3. The `auth.fetchUser()` call should successfully restore the user session from persistent cookies

## Current Behavior
1. Users log in successfully
2. Closing and reopening the browser requires re-authentication
3. The "Remember me" checkbox has no effect on cookie persistence
4. Session cookies are being used instead of persistent cookies

## Potential Solutions

### 1. Implement "Remember Me" Functionality
- Pass the `remember` flag to the login API
- Configure server-side to set appropriate cookie attributes based on this flag

### 2. Cookie Attributes Configuration
Ensure server sets cookies with proper attributes:
- `Expires` or `Max-Age` for persistence
- `HttpOnly` for security
- `Secure` in production
- `SameSite` for CSRF protection

### 3. Enhanced Client-side Initialization
- Improve the authentication initialization flow
- Add better error handling for session restoration
- Consider adding a loading state during authentication check

### 4. Development Environment Considerations
The Vite proxy may interfere with cookie persistence in development. Consider:
- Testing in a production-like environment
- Ensuring proxy configuration preserves all cookie attributes

## Testing Steps to Reproduce
1. Open the application in a browser
2. Log in with valid credentials (check "Remember me")
3. Verify successful login and user session
4. Close the browser completely
5. Reopen the browser and navigate to the application
6. Observe that user is logged out and needs to log in again

## Impact Assessment
- **User Experience**: Poor - Users must re-authenticate frequently
- **Security**: Medium - Session-only cookies are more secure but less convenient
- **Development**: Medium - Requires both frontend and potential backend changes

## Recommended Priority
**High Priority** - This is a core functionality issue that significantly impacts user experience and adoption.

## Related Issues
- Consider implementing refresh token mechanism for better security and persistence
- Evaluate if localStorage/sessionStorage should be used as fallback
- Review CSRF protection implementation with persistent cookies

## Notes
This bug report focuses on the client-side implementation. The server-side cookie configuration may also need to be updated to properly handle persistent authentication cookies based on the "remember me" preference.

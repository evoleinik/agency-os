# Authentication Patterns

**For projects requiring login/auth during testing.**

## Persistent Auth Files

Store authenticated state in JSON files to avoid re-login on every test:

```
auth-admin.json     # Admin user session
auth-user.json      # Regular user session
auth-director.json  # Super admin session
```

## Playwright Usage

```typescript
// Setup: Save auth state after login
await page.context().storageState({ path: 'auth-admin.json' });

// Test: Reuse auth state
const context = await browser.newContext({
  storageState: 'auth-admin.json'
});
```

## Browser State Management

- **Don't close the browser** when done - next test reuses it
- **Assume inherited state** - you may already be logged in
- **Navigate explicitly** - go to your target page, don't assume fresh start
- **Check auth on failure** - if redirect to login, re-authenticate

## Test Account Pattern

Keep test credentials in environment or config (never in prompts):

```bash
# .env.test
TEST_ADMIN_EMAIL=admin@example.com
TEST_ADMIN_PASSWORD=test123
TEST_USER_EMAIL=user@example.com
TEST_USER_PASSWORD=test123
```

## Auth File Structure

```json
{
  "cookies": [...],
  "origins": [
    {
      "origin": "http://localhost:3000",
      "localStorage": [...]
    }
  ]
}
```

## Security Notes

- Add `auth-*.json` to `.gitignore`
- Use test accounts only (never real credentials)
- Rotate test passwords periodically
- Don't commit auth files to public repos

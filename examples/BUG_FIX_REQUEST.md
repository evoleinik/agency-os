# BUG FIX REQUEST v1

## Source
QA_REPORT (Failed)

## Bug Description
The profile form accepts empty name input and submits to server, causing a 500 error. Client-side validation is missing.

## Steps to Reproduce
1. Navigate to /profile
2. Clear the name field completely
3. Click the Save button
4. Observe: Form submits, server returns 500 error

## Expected vs Actual
- **Expected**: Form shows validation error "Name is required" and prevents submission
- **Actual**: Form submits with empty name, server crashes with 500

## Console Error
```
POST /api/profile 500 (Internal Server Error)
Error: name cannot be null
    at ProfileController.update (profile.controller.ts:45)
```

## Suggested Fix Area
- File: src/components/ProfileForm.tsx
- Add validation to the name field before form submission
- Consider using a form library like react-hook-form with zod validation

## Related Files
- src/components/ProfileForm.tsx (form component)
- src/pages/api/profile.ts (API endpoint - may also need validation)

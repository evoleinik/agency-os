# QA REPORT
STATUS: FAIL

## Summary
Tested the user profile page. Found 2 bugs that need fixing.

## Pages Tested
| Page | Snapshot | Interactions | Verdict |
|------|----------|--------------|---------|
| /profile | Yes | 5 | FAIL |
| /profile (mobile) | Yes | 3 | FAIL |

## Tests Performed

### Profile Display
- [x] User name displays correctly
- [x] Email displays correctly
- [x] Join date formatted properly
- [x] Bio field shows current value

### Profile Editing
- [x] Name field accepts valid input
- [ ] **BUG: Name field accepts empty input (no validation)**
- [x] Bio field accepts up to 500 characters
- [x] Save button submits form
- [x] Success message displays after save
- [x] Changes persist after page refresh

### Avatar Upload
- [x] Upload button opens file picker
- [x] JPG files upload successfully
- [x] PNG files upload successfully
- [ ] **BUG: Files over 2MB are accepted (should be rejected)**
- [x] Preview shows after successful upload

## Bugs Found

### Bug 1: Empty name accepted
- **Location**: /profile form
- **Expected**: Validation error when name is empty
- **Actual**: Form submits with empty name, server returns 500
- **Steps to reproduce**:
  1. Go to /profile
  2. Clear the name field
  3. Click Save
  4. Observe 500 error in console

### Bug 2: Oversized avatar upload accepted
- **Location**: /profile avatar upload
- **Expected**: Files over 2MB rejected with error message
- **Actual**: Large files upload and cause server timeout
- **Steps to reproduce**:
  1. Go to /profile
  2. Click upload avatar
  3. Select a 5MB image file
  4. Observe upload starts but times out

## Recommendation
Fix client-side validation for both issues before re-testing.

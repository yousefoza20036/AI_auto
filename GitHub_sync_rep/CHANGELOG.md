# Changelog

## v1.2 — Multi-file uploads + true idempotent sync

### Added
- Optional file upload field on the sync form (README, screenshots, any other file)
- Automatic file routing: `README*` → project root, images → `screenshots/`, everything else → project root
- Real byte-level content comparison — the sync now skips the commit entirely when a file hasn't actually changed

### Fixed
- GitHub node's error-output branch was silently dropping binary data on new-file uploads, causing `Cannot read properties of undefined (reading 'file')`
- Content comparison was reading a binary-storage placeholder value (`"filesystem-v2"`) instead of real file bytes, so it always concluded files had changed — even identical re-uploads produced a new commit every time
- Index-based file matching could silently mismatch results when uploading a batch with both new and already-existing files; switched to n8n's pairedItem-aware `itemMatching()`
- Removed a leftover error-swallowing setting on the file-update step so genuine failures (auth, network) surface instead of disappearing

## v1.1 — GitHub sync core

### Added
- One-click, form-triggered sync of a selected n8n workflow to GitHub
- Automatic create-vs-update detection — checks GitHub before committing, no manual SHA handling

## v1.0 — Initial workflow

### Added
- Manual JSON export → GitHub commit replaced with an n8n-native automation

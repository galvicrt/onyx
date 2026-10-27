---
"sit-onyx": minor
---

feat(OnyxFileUpload): support importing a file from a URL, a custom `hint` and duplicate detection

- `importUrl`: shows an "Import from URL" button that downloads the file and adds it to the selection (only `http:` and `https:` URLs)
- `hint`: plain-text hint below the file list, `{filename}` is replaced with the most recently added file name
- files with identical content are now marked with a "Duplicate file" warning

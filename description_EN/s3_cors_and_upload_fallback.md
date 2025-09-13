# S3 CORS and Local Upload Fallback

## Goal
Provide a server-side fallback when presigned PUT is blocked by CORS in local/multi-origin setups.

## Recommended CORS
Add allowed origins/headers/methods and expose ETag; restrict in production.

## Fallback endpoint
- `POST /api/character/upload-direct`
  - multipart/form-data: `file` (required), `filename` (optional)
  - Auth: same as other character APIs (session headers)
  - Writes to S3 via SDK and returns `{ key }`

## Frontend behavior
`function/character/publish.ts`:
1) Try presign + PUT
2) On failure, call `/api/character/upload-direct`
3) Then call `/api/character/finalize-publish`

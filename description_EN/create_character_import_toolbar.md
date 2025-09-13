## Create Character - Import Toolbar

### Component
- `components/create-character/ImportToolbar.tsx`
- Two buttons: Import File (`.png`/`.json`) and Clear

### Behavior
- PNG import: parse PNG tEXt chunks, update page JSON, cache image blob `create_character_uploaded.png`
- JSON import: read text + `JSON.parse`
- Clear: restore default JSON and delete cached image blob

### Integration
- `app/create-character/page.tsx`: handlers to update `characterJson`, manage `selectedImage`, and PNG export via `writeCharacterToPng`

## Create Character Page: JSON Preview and PNG Export

### Default Character JSON
- Template at `lib/defaults/character-card.json` (enabled by `resolveJsonModule`)

### JSON Preview Module
- `components/create-character/JsonPreviewPanel.tsx`
- Fantasy theme styles
- Pretty-print, copy, download

### PNG Export Flow
- `utils/character-parser.ts` → `writeCharacterToPng(file, data)` writes JSON into PNG `tEXt` chunks

### Page Integration
- `app/create-character/page.tsx`
- Top `ImportToolbar` (import/clear), right column has JSON preview + PNG export, publish card exists

### CharacterCardInfo
- Maps fields like name/intro/tags/creator/version

## Create Character Page: JSON Preview and PNG Export

This document summarizes how the `/create-character` page implements the JSON preview module and PNG character card export.

### Default Character JSON
- The default character card schema is stored at `lib/defaults/character-card.json` and can be imported directly on the frontend thanks to `resolveJsonModule` in `tsconfig.json`.

### JSON Preview Module
- Component: `components/JsonPreviewPanel.tsx`.
- Visual style: Uses `session-card` background (same pattern as the character market search panel) from `app/styles/fantasy-ui.css`.
- Icon: `components/icons/JsonIcon.tsx`, exported via `components/icons/index.ts`.
- Features:
  - Pretty-printed JSON view
  - Copy to clipboard
  - Download JSON file
  - Optional fullscreen trigger (plumbed prop)

### PNG Export Flow
- Utility: `utils/character-parser.ts` provides `writeCharacterToPng(file, data)` which embeds character data into PNG `tEXt` chunks (`chara` and `ccv3`).
- On `/create-character`, users select a base PNG. Clicking "下载 PNG 角色卡" writes the current JSON into the PNG and downloads it.

### Page Integration
- File: `app/create-character/page.tsx`.
- Imports default JSON and holds it in local state; passes it to `JsonPreviewPanel`.
- Provides Import Toolbar with two actions: Import File and Clear.
- Import logic:
  - PNG → `parseCharacterCard` then update state; cache image via `setBlob("create_character_uploaded.png")`.
  - JSON → read text + `JSON.parse` to update state.
- Download section handled by `DownloadCharacterCard`; PNG export uses `writeCharacterToPng`.
- Layout: page content sits in a transparent container with `max-w-6xl` and height `calc(100vh - 140px)`; two-column split with scrolling on the left pane; right pane hosts the JSON preview.
 - Layout updated: container uses `max-w-7xl` and responsive paddings; 2/3 (left) vs 1/3 (right). Right column stacks `JsonPreviewPanel` on top of `DownloadCharacterCard`; on small screens, the right column moves below the left.

### CharacterCardInfo Component
- Path: `components/create-character/CharacterCardInfo.tsx`.
- Fields mapping:
  - Name: `name`
  - Community intro: `data.extensions.custom_fields.user_description`
  - Tags (max 3): `data.tags` (source list from env: `NEXT_PUBLIC_TAG_CN` or `NEXT_PUBLIC_TAG_EN`)
  - Creator: `creator`
  - Version: `character_version`

### Notes
- The documentation in this directory reflects implementation, but always verify behavior by checking referenced files.



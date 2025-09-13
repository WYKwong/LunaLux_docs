# Settings Page Overview

## Overview
- Dedicated `/settings` page mirroring `components/SettingsDropdown.tsx` core features

## Route & Rendering
- Path: `app/settings/page.tsx`
- Client component

## Features
- Language switch via `useLanguage` (zh/en)
- Inline model settings form `components/ModelSettingsInline.tsx` (user-configured models)
- Sound toggle (`useSoundContext`)
- Reset tour (`useTour.resetTour()`)
- Plugin manager (optional button exposure)

## Layout Events
- `MainLayout.tsx` listens `openModelSidebar` / `toggleModelSidebar`
- `/settings` dispatches `openModelSidebar`

## Sidebar
- Add `Settings` item in `components/sidebar/sidebarConfig.tsx` → `/settings`
- Icon: `components/icons/SettingsIcon.tsx`

## i18n
- Re-use existing keys: `common.settings`, `common.switchToEnglish`, `common.switchToChinese`, `modelSettings.title`, `common.soundOn/off`, `tour.resetTour`

## Styling
- Dark fantasy theme; hover glow; smooth transitions

## Tests
- Page toggles language, opens model sidebar, persists sound, resets tour, and highlights sidebar item

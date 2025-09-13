# User Page Implementation

## Overview
- Dedicated `/user` page replaces the old `AccountModal` to provide a richer profile experience

## Features
- Display: username (editable), email, userId, status
- Wallet: badges and balances (local cache)
- Invite profile: code, total invites, unissued count
- Coupon redeem panel: calls `POST /api/coupon/redeem`, updates local wallet/ledger
- Redirect unauthenticated users

## Tech
- Client component, hooks-based state, framer-motion animations, i18n
- Consistent background styles

## Sidebar & Mobile
- Sidebar menu config driven; mobile nav links to `/user`

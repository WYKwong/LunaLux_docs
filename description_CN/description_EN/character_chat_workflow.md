# Character Chat Workflow (End-to-end)

## Overview
This document explains the flow from user input on the character page to receiving the LLM response, powered by a node-based workflow.

## Architecture
Frontend UI → API (Function) → Workflow Engine → Nodes → LLM Services

## 1) Character Page
- File: `app/character/page.tsx`
- Manages state, user input, and triggers chat requests.

## 2) CharacterChatPanel
- Handles input, send-lock, streaming-complete signals, and rendering.
- Enter sends; Shift+Enter for newline (i18n placeholder).

## 3) API Layer (Dialogue)
- Entry: `function/dialogue/chat.ts`
- Flow:
  - Build `DialogueWorkflowParams`
  - `DialogueWorkflow.execute()` returns structured output
  - Async post-processing persists dialogue tree (does not block UI)

## 3.3) Official Model Billing & Forwarding
- File: `app/api/official-model/submit/route.ts`
- Validates session, approves billing (free/star/luna) and calls official LLM in parallel.
- Updates wallet and, when applicable, `Metadata.DialogueCount/TotalLunaCoinUsed`.
- If character not found in tables, returns `deactivateBinding=true` while still returning content.

## 3.4) Local Binding + Server Counters
- Local `marketBinding` (IndexedDB) controls whether to attach community `characterId` on the official path.
- Server increments `DialogueCount` always; adds `TotalLunaCoinUsed` only on luna billing.

## 4) Workflow Engine
- Definition: `lib/workflow/examples/DialogueWorkflow.ts`
- Nodes: userInput → pluginMessage → preset → context → worldBook → llm → regex → plugin → output
- Engine: `BaseWorkflow` orchestrates ENTRY → MIDDLE → EXIT; AFTER nodes can run in background.

## 5) Node Highlights
- PresetNode/PresetNodeTools: build prompt framework using character info and enabled preset
- LLMNode/LLMNodeTools: create LLM client and invoke
- RegexNode: strip thinking, extract `next_prompts/events`, run regex

## 6) Prompt System
- Prompt presets with prompt/cot/structure blocks; multiple keys supported.

## 7) Persistence
- Async post-processing adds nodes to dialogue tree and updates events when present.

References: `lib/nodeflow/**`, `lib/workflow/**`, `function/dialogue/**`

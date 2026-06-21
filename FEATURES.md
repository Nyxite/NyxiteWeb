# Nyxite Web — Features

Next.js + shadcn/ui client. Primary surface for anonymous guest access.

## Editing

- Markdown, handwritten ink, and plain-text editing and viewing

## Organization

- Project and folder navigation

## Collaboration

- Live collaborative editing
- Anonymous guest access via share links (opens in browser, no account required)

## Version history

- Version history and diffs

## Sharing

- Share link creation and management

## Authentication

- Keycloak login with TOTP

## Open questions

- Guest session model: how anonymous link sessions are issued, scoped, and expired (web owns this flow)
- CRDT on web: Yjs (the JS reference implementation) interoperability with the server's Ycs — verify wire compatibility
- In-browser ink editing (pointer events / canvas) and parity with the native clients
- Read-only vs editable guest links and the per-link permission UI

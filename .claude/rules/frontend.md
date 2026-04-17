---
paths:
  - "apps/web/**"
---

# Frontend Rules

- Mobile-first: design for small screens first, then scale up
- All components must be keyboard-navigable and screen reader friendly
- Never render user content as raw HTML — use the framework's safe text rendering mechanism
- Image optimization, code splitting, unnecessary re-render prevention
- All images through the framework's image optimization
- UI text, button labels, error messages, SEO content: Turkish

## Identity display (always enforced on the frontend)

- Username is the primary public identifier. Topic cards, comment cards, notification actors, vote lists, and every other "author next to content" surface render **username only**. Never render firstName or lastName in these contexts
- On another user's profile page only (not cards), display the given name in full plus the family name abbreviated to a single initial followed by a period. Example: a user named "Ahmet Demir" appears as `"Ahmet D."` to everyone else
- The owner of the profile sees their own full surname on their own profile page; other users never do
- Admin dashboard surfaces may display the full legal name for moderation / KVKK compliance — that data lives behind admin endpoints only. Non-admin frontends must not import or render admin response shapes
- Never request a new "full name" field on a shared DTO to sidestep these rules — if a card is rendering `{firstName} {lastName}`, the bug is structural. Fix it by switching to `UserPublicCardSchema` (username only). Mirror requirement applies to backend response shaping (`.claude/rules/security.md` → "Identity Display Rules")

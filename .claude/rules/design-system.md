---
paths:
  - "apps/web/**"
---

# Design System

Overall feel: Social, warm, friendly. A design that creates community feeling and welcomes users.

## Color Palette

Every color below must define both a light and a dark variant. Dark variants follow the same semantic role but use shades adapted for dark surfaces.

- **Primary:** Purple/Lavender tone — brand color, buttons, active elements, links
- **Secondary:** Warm gray — text, background variations
- **Accent:** Light purple tint — highlighting, hover states, selected elements
- **Success:** Green — successful operations, "Haklı" vote emphasis
- **Danger:** Red/coral — errors, warnings, "Haksız" vote emphasis
- **Background:** Light → warm white (not cold white); Dark → deep neutral (not pure black)
- **Surface:** Light → light gray/white; Dark → elevated dark neutral — for cards and elevated areas
- **Text primary:** Light → dark gray (not pure black); Dark → off-white (not pure white)
- **Text secondary:** Medium gray — helper text, dates, placeholders
- **Theme:** Light, dark, and system preference. System preference is the default; user choice overrides it and is persisted.

## Typography

- **Font family:** Nunito (Google Fonts) — rounded, friendly, readable
- **Headings:** Bold (700), graduated sizes from large to small
- **Body text:** Regular (400), readable line height
- **Small text:** Dates, helper info — with secondary color
- **Button text:** SemiBold (600)

## Spacing

- **Base unit:** 4px (Tailwind default)
- **Component inner padding:** 12-16px
- **Gap between cards:** 16px
- **Page margin:** Mobile 16px, desktop 24-32px
- **Between sections:** 24-32px

## Border & Radius

- **Default radius:** 8px — cards, buttons, inputs
- **Small elements (badge, chip):** 6px
- **Avatar:** full (circle)
- **Border:** Thin (1px), light gray — card and input borders

## Component Rules

- **Cards:** Light shadow + border, warm white background, 8px radius
- **Buttons:** Primary (purple fill, white text), secondary (outline), ghost (text only)
- **Inputs:** Border, 8px radius, primary color border on focus
- **Avatar:** Circle, show initials if no system avatar
- **Vote buttons:** "Haklı" (green tone), "Haksız" (red/coral tone) — fill color when selected, outline when unselected
- **Anonymous card:** Distinct background tone to differentiate from normal cards

## Icons

- **UI icons:** Lucide React (shadcn/ui default) — for navigation, buttons, actions, status indicators
- **User emojis:** Twemoji — for emojis added by users in topics and comments
- Never use emoji characters for UI icons — always import from icon library
- Icon sizes must be consistent: 16px (small), 20px (default), 24px (large)

## Responsive Behavior

- **Mobile (< 640px):** Single column, bottom navigation bar, full-width cards
- **Tablet (640-1024px):** Two-column grid possible, sidebar hidden
- **Desktop (> 1024px):** Centered content area + side margins, max-width 1200px container

## Animation

- **Transition duration:** 150-200ms (fast, smooth)
- **Easing:** ease-in-out
- **Hover:** Subtle scale or shadow increase
- **Page transitions:** Fade-in
- **Avoid unnecessary animations** — performance and accessibility first

## Rank Badge

- The `<RankBadge />` component is the canonical rendering of any rank-related visual. Do not render rank name text without the badge wrapper — rank identity is icon + color + name together
- **Colors:** each rank carries a semantic color token seeded on the `Rank` row. The 8 current tokens are `zinc`, `violet`, `blue`, `indigo`, `emerald`, `amber`, `rose`, and `gold`. Keep contrast AA compliant on both light and dark surfaces
- **Size variants:** `"sm"` (compact for lists), `"md"` (default for user cards and notifications), `"lg"` (profile header)
- **Progress surface:** `<RankProgress />` uses the same tokens to tint the filled portion of the progress bar; summit state renders with a gold accent

## Avatar Grid

- Locked avatars use a translucent overlay (~0.35 opacity) plus a centered Lucide `Lock` icon
- Hovering (pointer-over), receiving keyboard focus, or long-pressing on touch devices removes the overlay so the viewer can preview the locked avatar in full color; leaving the card restores the overlay
- Locked avatars are not activatable — clicking/tapping reveals a tooltip stating the required rank; no network request is fired
- Selected avatar carries a primary-tone ring
- Every card exposes an `aria-label` describing its state (available / selected / locked + rank requirement). The grid container uses `role="radiogroup"` so assistive tech announces the selection model correctly
- System-category avatars (the "Silinmiş Kullanıcı" placeholder) are never rendered in user-facing grids — they are a backend-only rendering concern

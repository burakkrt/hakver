---
paths:
  - "apps/web/**"
---

# Design System

Overall feel: Social, warm, friendly. A design that creates community feeling and welcomes users.

## Color Palette

- **Primary:** Purple/Lavender tone — brand color, buttons, active elements, links
- **Secondary:** Warm gray — text, background variations
- **Accent:** Light purple tint — highlighting, hover states, selected elements
- **Success:** Green — successful operations, "Haklı" vote emphasis
- **Danger:** Red/coral — errors, warnings, "Haksız" vote emphasis
- **Background:** Light, warm white (not cold white) — easy on the eyes
- **Surface:** Light gray/white for cards and elevated areas
- **Text primary:** Dark gray (not pure black)
- **Text secondary:** Medium gray — helper text, dates, placeholders
- **Theme:** Light mode only

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

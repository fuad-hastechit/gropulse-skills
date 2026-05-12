---
description: Add full multilanguage support (i18next + react-i18next, DB-persisted language, auto-detection, language switcher) to a Gropulse Shopify app
---

Invoke the gropulse-skills:translation skill.

Add full multilanguage i18n support to this Gropulse Shopify app:
- Install i18next + react-i18next
- Create app/locales/ with JSON files per language + barrel index (15 languages: en, zh-CN, zh-TW, fr, de, ja, es, pt-BR, pt-PT, it, hi, nl, vi, el, fi)
- Create app/config/translations/ for server-side/PDF flat label maps (10 languages)
- Add createI18nInstance factory in app/i18n.ts
- Wire I18nextProvider in app/routes/app.tsx — add appLanguage to loader, wrap Outlet
- Add app/services/i18n.server.ts — resolveLanguage (billing country → customer locale → shop default), getLabels, formatDateForLocale, getNumberLocale
- Add AppLanguageSelector component — instant client-side switch + Prisma persistence via settings action
- Add getAppLanguage to app/models/settings.server.ts
- Handle appLanguage tab in the settings route action

Critical placement rule: AppLanguageSelector goes as first child of <s-page> in each route — NOT inside <s-app-nav> (silently dropped) and NOT as a standalone sibling in app.tsx.

Follow the skill steps exactly. State assumptions before starting.

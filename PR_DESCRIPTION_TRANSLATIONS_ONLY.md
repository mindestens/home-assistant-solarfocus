# Add Comprehensive German Translations from Official Documentation

## Summary

This PR adds comprehensive German translations for Solarfocus entities and state values based on the official Solarfocus Modbus TCP documentation (DR-0180-DE / v13-251216).

## Type of Change

- [x] New feature (non-breaking change which adds functionality)
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] Breaking change

## Impact

- User interface texts for German users are significantly improved
- Entity names and state texts better match official Solarfocus terminology
- No breaking changes: entity IDs remain unchanged
- Existing automations and dashboards continue to work

---

## Motivation

Solarfocus is widely used in the DACH region. In this context, complete and terminology-accurate German labels improve usability and reduce ambiguity in Home Assistant.

Problems addressed:
1. Incomplete or inconsistent German naming in the UI
2. Missing German state translations for multiple entities
3. Terminology mismatches versus official Solarfocus register documentation

---

## Scope of This PR

This PR contains translation-focused updates only.

Included:
1. Expanded German translations in `custom_components/solarfocus/translations/de.json`
2. Improved entity labels and state texts based on official documentation

Not included:
1. No structural integration refactoring
2. No entity ID changes
3. No migration logic changes

---

## Translation Methodology

Source:
- Solarfocus ecomanager-touch Modbus TCP documentation
- Document ID: DR-0180-DE
- Version: v13-251216

Approach:
1. Keep terminology aligned with official register descriptions
2. Prefer consistency across related entities and states
3. Preserve backward compatibility at entity ID level

---

## Testing Performed

1. Verified Home Assistant starts normally after translation changes
2. Verified entity IDs remain stable
3. Verified German labels/states render in UI when language is set to German
4. Verified no functional regression in automations depending on existing entity IDs

---

## Breaking Changes

None.

- Entity IDs unchanged
- Existing automations unaffected
- Historical data preserved

---

## Documentation

- Translation updates are documented in this PR description
- Changelog update is recommended if this translation update is released to users

---

## Checklist

- [x] Translation changes are non-breaking
- [x] Entity IDs remain unchanged
- [x] Terminology aligned with official documentation
- [x] No unrelated refactoring included in this PR

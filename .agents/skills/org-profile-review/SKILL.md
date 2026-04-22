---
name: org-profile-review
description: Review the GitHub organization public profile for broken links, rendering problems, and branding inconsistencies.
allowed-tools: Read Grep Glob
---

# Organization Public Profile Review

Review changes to the public GitHub organization profile under `profile/`.

Current public terminology:

- Company name: Fields Education Inc.
- `joinfields.org`: general marketing site
- `fields.app`: Fields Classroom
- `fields.org`: Fields Marketplace

Focus on concrete, user-visible issues introduced by the change:

- incorrect or inconsistent company, product, or site naming
- broken links, bad email addresses, or mismatched destinations
- GitHub rendering problems in `profile/README.md`
- broken relative asset paths, theme-related readability issues, or inaccessible image usage
- SVG or image issues that would make the profile unreadable or misleading
- factual contradictions introduced by the change

Do not nitpick tone or suggest broad marketing rewrites unless the current change creates confusion or an inaccuracy.

For each finding:

- cite the specific file and line
- explain the concrete user impact
- suggest the smallest reasonable fix

If the change looks correct, report no findings.

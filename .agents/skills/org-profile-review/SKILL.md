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

For external links, especially social/profile links:

- do not report a link as wrong just because you believe another domain is more canonical
- alternate first-party domains and redirects are acceptable when the link text and URL path are consistent
- only flag a social/profile link when the problem is evident from the diff itself, for example:
  - the handle in the URL does not match the displayed account name
  - the destination is obviously for a different brand or person
  - the URL is malformed or clearly points to the wrong product/site
- if you cannot prove the external link is wrong from the changed content alone, do not report it

Do not nitpick tone or suggest broad marketing rewrites unless the current change creates confusion or an inaccuracy.

For each finding:

- cite the specific file and line
- explain the concrete user impact
- suggest the smallest reasonable fix

If the change looks correct, report no findings.

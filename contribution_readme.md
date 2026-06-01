# Contribution 1: [Security] XSS — concurrencyError.jsp renders clinical note in `<textarea>` without encoding

**Contribution Number:** 1  
**Student:** Nikayel Jamal  
**Issue:** https://github.com/carlos-emr/carlos/issues/2319  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose issue #2319, an XSS (cross-site scripting) vulnerability in the carlos
project, because it aligns with my interest in web security and has a clear,
bounded scope. The issue is labeled "good first issue," "help wanted," and
"Security," and rates the review effort as 2/5 — signals that it's an
appropriate first contribution. It is currently unassigned with no linked pull
requests, so it's available to claim.

I'm interested in this because:
1. The vulnerability is concrete and easy to understand: a clinical note is
   rendered directly into a `<textarea>` without HTML encoding, so a note
   containing a `</textarea>` tag can break out of the element and inject
   arbitrary HTML or scripts.
2. The fix is well-scoped — it centers on a single file and line — which makes
   it realistic to complete within the 3–4 week cycle.
3. The issue even points to a related fix (#2302 in `billingSettings.jsp`) and
   suggests the project's own `carlos:encode` taglib as the solution, giving me
   a clear pattern to follow and learn from.
4. I want to learn how a real Java/JSP web application handles output encoding
   and defends against stored XSS, and how this project structures its security
   fixes.

From reading the issue, I understand the problem is that untrusted clinical-note
content is output without encoding, allowing tag injection. My contribution will
HTML-encode that content so the note is displayed safely as text rather than
interpreted as markup, improving the application's security.

> _Note: This is in Java/JSP, which I'll be ramping up on with Claude Code during
> Phase II. The fix follows an established pattern in the codebase, which makes the
> language gap manageable._

---

## Understanding the Issue

### Problem Description

The `concurrencyError.jsp` page renders a clinical note (`bean.encounter`)
directly into a `<textarea>` element without HTML-encoding it first. Because the
content is untrusted (it can come from data stored in the database), a note that
contains a `</textarea>` sequence can prematurely close the textarea and inject
arbitrary HTML — a stored cross-site scripting (XSS) vulnerability.

### Expected Behavior

The clinical-note content should be HTML-encoded before being inserted into the
`<textarea>`, so that characters like `<` and `>` are displayed as text rather
than interpreted as markup. No injected tags or scripts should execute when the
concurrency-error page loads.

### Current Behavior

Line 62 outputs the value unencoded:
`<textarea><%= bean.encounter %></textarea>`. A stored note such as
`</textarea><img src=x onerror=alert(1)>` breaks out of the textarea and executes
script in the user's browser when the page is rendered.

### Affected Components

- **File:** `src/main/webapp/WEB-INF/jsp/encounter/concurrencyError.jsp` (line 62)
- **Related fix for reference:** #2302, a similar XSS fix in `billingSettings.jsp`
- **Suggested mechanism:** the project's own taglib —
  `<carlos:encode value='<%= bean.encounter %>' context="html"/>`

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]

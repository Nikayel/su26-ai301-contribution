# Contribution 1: [Security] XSS — concurrencyError.jsp renders clinical note in `<textarea>` without encoding

**Contribution Number:** 1  
**Student:** Nikayel Jamal  
**Issue:** https://github.com/carlos-emr/carlos/issues/2319  
**Status:** Phase II Complete

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

Because the full OSCAR EMR stack requires MySQL + Tomcat + a built WAR, I reproduced
the vulnerability using an isolated JSP on a standalone Tomcat 10.1.25. This approach
targets the exact vulnerable line in isolation and is sufficient to demonstrate the
broken-textarea payload executing in a browser.

**Tools used:** Java 23 (pre-installed), Apache Tomcat 10.1.25 (downloaded tarball),
`curl` for automated payload delivery.

**Setup steps:**
1. Downloaded `apache-tomcat-10.1.25.tar.gz` from the Apache archives and extracted
   it to `/tmp/xss-repro/`.
2. Created a one-file JSP webapp (`xss-demo/vulnerable.jsp`) that replicates the
   exact pattern from `concurrencyError.jsp` line 62:
   ```jsp
   <textarea name="encounterTextarea" wrap="hard" cols="99" rows="20"><%=encounterNote%></textarea>
   ```
   where `encounterNote` simulates `bean.encounter` (an attacker-controlled string
   from the database).
3. Started Tomcat (`bin/startup.sh`) and verified the normal case renders correctly.

### Steps to Reproduce

1. Deploy the vulnerable JSP to a Tomcat instance (as described in Environment Setup).
2. Visit the page with a benign note:
   ```
   http://localhost:8080/xss-demo/vulnerable.jsp
   ```
   Observe a normal textarea containing the default clinical note text — no issues.
3. Send the XSS payload as the `note` parameter (simulating a stored note retrieved
   from the database):
   ```
   http://localhost:8080/xss-demo/vulnerable.jsp?note=%3C%2Ftextarea%3E%3Cimg+src%3Dx+onerror%3Dalert(document.cookie)%3E%3Ctextarea%3E
   ```
   (URL-decoded payload: `</textarea><img src=x onerror=alert(document.cookie)><textarea>`)
4. **Observed result:** The textarea element is prematurely closed by `</textarea>`.
   The `<img>` tag is rendered as raw HTML outside the form element. In a browser,
   the `onerror` handler fires immediately, executing `alert(document.cookie)`. The
   raw HTML in the server response confirms the breakout:
   ```html
   <textarea name="encounterTextarea" ...></textarea><img src=x onerror=alert(document.cookie)><textarea></textarea>
   ```

### Reproduction Evidence

- **Branch:** https://github.com/Nikayel/su26-ai301-contribution/tree/main
- **My findings:** The `curl` response directly shows the textarea being closed early
  and the `<img>` tag emitted as free HTML. The OWASP classification is **Stored XSS**
  because in the real application `bean.encounter` is read from the database — a
  malicious note saved by any user with encounter-write access would activate on every
  clinician who hits the concurrency-error page while editing that record.
  The same `carlos:encode` taglib that fixed `billingSettings.jsp` in issue #2302 is
  the correct remedy here: it HTML-encodes `<`, `>`, `&`, `"` so the note content is
  displayed as text, never interpreted as markup.

---

## Solution Approach

### Analysis

The root cause is missing output encoding at the JSP template layer. JSP's `<%= expr %>`
emits the raw string value of the expression — it performs no escaping. When `bean.encounter`
contains `</textarea>`, the browser's HTML parser sees a valid closing tag and ends the
textarea element, interpreting everything that follows as regular page markup. This is a
classic **stored XSS via untrusted data in an HTML element context** (OWASP A03:2021).

The vulnerability has been in the file since at least the initial commit. The concurrency
error page is uncommon but reachable in any busy clinic where two users open the same
encounter simultaneously — making it a realistic attack surface.

### Proposed Solution

Add the `carlos` taglib declaration to `concurrencyError.jsp` and replace the raw `<%= %>`
with `<carlos:encode ... context="html"/>`. This is exactly the fix pattern used in
`billingSettings.jsp` (PR #2302) and is the project's established standard for HTML-context
output encoding.

The `carlos:encode` tag HTML-encodes the five sensitive characters (`<`, `>`, `&`, `"`, `'`)
so that `</textarea>` becomes `&lt;/textarea&gt;`, which the browser renders as visible text
inside the textarea rather than as a closing HTML tag.

### Implementation Plan

Using UMPIRE framework:

**Understand:** `concurrencyError.jsp` line 62 outputs `bean.encounter` (a DB-sourced
clinical note string) directly into a `<textarea>` with no HTML encoding. A note containing
`</textarea>` closes the element and injects arbitrary HTML/JS into the page.

**Match:** The codebase already has the solution: `billingSettings.jsp` was fixed identically
in issue #2302. That fix added `<%@ taglib uri="carlos" prefix="carlos" %>` at the top of
the file and replaced raw `<%= %>` outputs with `<carlos:encode value='...' context="html"/>`.

**Plan:**
1. Open `src/main/webapp/WEB-INF/jsp/encounter/concurrencyError.jsp` in the fork.
2. Add the taglib declaration near the other existing taglib declarations (after line ~33):
   ```jsp
   <%@ taglib uri="carlos" prefix="carlos" %>
   ```
3. Replace line 62:
   ```jsp
   <%-- BEFORE (vulnerable) --%>
   <textarea name='encounterTextarea' wrap="hard" cols="99" rows="20"><%=bean.encounter%></textarea>

   <%-- AFTER (fixed) --%>
   <textarea name='encounterTextarea' wrap="hard" cols="99" rows="20"><carlos:encode value='<%= bean.encounter %>' context="html"/></textarea>
   ```
4. Run `mvn -B -Pjspc package -DskipTests` (the maintainer-suggested verification command)
   to confirm the JSP compiles without errors.
5. Manually verify: deploy locally, insert a synthetic note containing
   `</textarea><img src=x onerror=alert(1)>`, trigger the concurrency-error page, and
   confirm the payload appears as escaped text inside the textarea rather than executing.

**Implement:** Link to branch/commits will be added here as work proceeds.

**Review checklist:**
- [ ] Only `concurrencyError.jsp` is modified — no changes to encounter save logic,
      session handling, or Struts actions (per maintainer guidance)
- [ ] Taglib URI matches the project's existing pattern (same `uri="carlos"` used elsewhere)
- [ ] `context="html"` is correct for textarea body content (vs. `htmlAttribute` for attributes)
- [ ] JSP compiles via `mvn -B -Pjspc package -DskipTests`
- [ ] Test data uses synthetic names (e.g., "Fake-name Fake-surname")

**Evaluate:** After the fix, the same XSS payload (`</textarea><img src=x onerror=alert(document.cookie)>`)
stored as a clinical note must render as escaped text (`&lt;/textarea&gt;...`) inside the
textarea, with no JavaScript executing and the textarea element remaining intact.

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

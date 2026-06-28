# Prompt Template

**Date:** 2026-06-28
**Author:** Andrei Siarheichyk — Architecture Team
**Project:** Meridian Retail
**Model:** Claude Opus 4.8
**DIAL location:** [\[Link\]](https://chat.lab.epam.com/share/jwS3V82EAsCAWTm4oJVrdvUVDsgu3vqnjQgB2kETNGgJMrNCXYLwZAeaRPKZwkJAJobnko31YCSdybeSMQRszu6hmWmTJNoYtJe5hkknRUMitLue2M32wY3JU3JNsJBDqLTPJW4SwXXBSVDFfujAKgxHN)
**Committed location:** [\[Repo path\]](https://github.com/a-sergeichik/epam-ai-run-2026/blob/main/kata-2/prompt.md)

---

## Purpose

Generates a numbered list of Gherkin-formatted acceptance criteria from a single user story, used by BAs and product squads during the Requirements / Backlog Refinement stage of the SDLC.

---

## Variable Placeholders

| Placeholder | Description | Example value |
|---|---|---|
| `{{user_story}}` | The full user story in "As a / I want / so that" form | As a store associate at a retail location, I want to look up a customer's account by email or phone number at the POS terminal, so that I can apply their loyalty points without asking them to re-register in store. |
| `{{output_language}}` | Natural language the criteria should be written in | English |
| `{{output_format}}` | Output format | Gherkin |

---

## Output Format Instruction

Return a numbered list of acceptance criteria. Each item must have a short bold title followed by a fenced gherkin code block using Scenario / Given / When / Then / And keywords. One scenario per criterion. Cover happy path, alternate paths, and at least one error/edge case. No preamble, no closing summary.
---

## Prompt Body

You are a senior Business Analyst writing testable acceptance criteria.

Draft acceptance criteria from the user story below.

<user_story>
{{user_story}}
</user_story>

Requirements:
- Write the criteria in {{output_language}}.
- Format each criterion as a {{output_format}} scenario.
- Produce a numbered list; give each criterion a short bold title above its gherkin block.
- Cover the happy path, meaningful alternate paths, and at least one error or edge case.
- Keep each scenario atomic (one observable outcome). Use concrete, realistic example values.
- Do not include any text before the first criterion or after the last.
---

## Test Run (Author)

**Input values used:**
- `{{user_story}}` =   As a store associate at a Meridian retail location,
                          I want to look up a customer's account by email or phone number at the POS terminal,
                          so that I can apply their loyalty points and access their purchase history without asking them to re-register in store.
- `{{output_language}}` = English
- `{{output_format}}` = Gherkin

**Output quality:** The out was usable as-is

---

## Peer Review

**Reviewer:** NA
**Date reviewed:** 
**Model used by reviewer:**

**Reviewer input values used:**
- `{{placeholder_1}}` = [value reviewer used]
- `{{placeholder_2}}` = [value reviewer used]

| Review question | Reviewer answer |
|---|---|
| Could you run the template without asking the author anything? | Yes / No — [one sentence] |
| Was the output format what you expected? | Yes / No — [one sentence] |
| Would you use this template on your own work? | Yes / No — [one sentence] |
| One concrete improvement suggestion | [One sentence] |

---

## Revision History

| Version | Date | Change | Author |
|---|---|---|---|
| 1.0 | 2026-06-28 | Initial commit | Andrei Siarheichyk |
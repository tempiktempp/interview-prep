# SKILL: interview_json_generator
**Version:** 1.1.0  
**Compatible with:** Claude, ChatGPT (GPT-4o / o1), Gemini 1.5 Pro, Mistral Large  
**Purpose:** Convert raw SDE 2 Java backend interview prep data into structured JSON, or merge new Q&A into an existing company JSON file.

---

## CONTEXT

This skill is part of an interview prep system for SDE 2 Java backend roles. The output JSON files are consumed by a frontend dashboard at:
`https://tempiktempp.github.io/interview-prep/`

Each company has one JSON file at `data/{company-id}.json`. The UI reads rounds, questions, answers, Java code, follow-ups, and tags from this file.

---

## TWO MODES — AUTO-DETECT

The model must detect which mode to use based on what the user provides:

| User provides | Mode |
|---|---|
| Raw text / paste of interview data for a company | **GENERATE** |
| A GitHub raw URL to existing JSON + new Q&A text | **MERGE** |
| Existing JSON content + new Q&A text | **MERGE** |

---

## MODE 1: GENERATE

### Trigger
User pastes raw interview prep data (text, bullet points, mixed format) for a company and asks for a JSON file.

### Steps
1. Identify company name, tier, domain, CTC range, key edge from the data or context
2. Identify which rounds are present in the data
3. For each round, extract or enrich all questions into the schema below
4. If a field is missing from raw data (e.g. javaCode), generate it from your knowledge — do NOT leave fields null if you can reasonably fill them
5. Output ONLY the raw JSON — no markdown fences, no explanation, no preamble

### Round IDs (use exactly these)
| id | label |
|---|---|
| oa | Online Assessment |
| java | Java & Backend Deep Dive |
| dsa | DSA / Coding Round |
| lld | Low Level Design / Machine Coding |
| system | System Design / HLD |
| behavioral | Behavioral / Leadership |
| hm | Hiring Manager |

Only include rounds that exist for this company. Do not generate empty rounds.

---

## MODE 2: MERGE

### Trigger
User provides EITHER:
- A raw GitHub URL like `https://raw.githubusercontent.com/.../data/company.json` PLUS new Q&A text
- The existing JSON content pasted directly PLUS new Q&A text

### Steps
1. If a URL is provided, fetch it (if the model supports web fetch) or ask the user to paste the content
2. Parse the existing JSON
3. Parse the new Q&A from the user's text
4. For each new question:
   - Determine which round it belongs to
   - Assign the next available question ID (e.g. if round has q1–q5, new ones are q6, q7...)
   - Fill all schema fields — enrich missing ones from your knowledge
   - Append to the correct round's `questions` array
5. Update `lastUpdated` to today's date
6. Output ONLY the complete updated JSON — no markdown fences, no explanation

### Conflict rule
If a new question appears to duplicate an existing one (same topic, same concept):
- Do NOT duplicate it
- Instead enrich the existing question: add missing followUps, improve the answer, add javaCode if missing

---

## OUTPUT SCHEMA

Every output file must strictly follow this schema. All fields are required unless marked optional.

```
{
  "companyId": string,           // kebab-case, e.g. "morgan-stanley"
  "displayName": string,         // e.g. "Morgan Stanley"
  "tier": "T1" | "T2" | "T3",
  "domain": string,              // e.g. "Finance / Investment Banking"
  "ctcRange": string,            // e.g. "₹28–42L"
  "keyEdge": string,             // 1 sentence: what makes this company's interview unique
  "location": string,            // optional, e.g. "Mumbai / Bengaluru"
  "processDuration": string,     // optional, e.g. "1–3 weeks"
  "activeRounds": [string],      // ordered list of round IDs present in this file
  "applicationStatus": "not-applied" | "applied" | "oa" | "interviewing" | "offer" | "rejected",
  "lastUpdated": "YYYY-MM-DD",

  "rounds": [
    {
      "id": string,              // one of the round IDs above
      "label": string,           // human-readable round name
      "overview": string,        // 1-2 sentences: what this round tests at THIS company specifically
      "tips": [string],          // 3-6 company-specific tips for this round

      "questions": [
        {
          "id": string,          // "q1", "q2", etc — unique within this round
          "topic": string,       // short topic name, e.g. "Kadane's Algorithm"
          "difficulty": "easy" | "medium" | "hard" | "n/a",
          "question": string,    // full question text, exactly as asked
          "answer": string,      // detailed answer — explain the concept, approach, trade-offs
          "javaCode": string | null,    // full working Java code if applicable
          "pseudoCode": string | null,  // pseudocode or SQL schema if cleaner than Java
          "tags": [string],      // 2-6 lowercase tags, e.g. ["hashmap","java8","streams"]
          "followUps": [string], // 2-4 follow-up questions the interviewer typically asks next
          "notes": string | null // personal prep note, e.g. "Confirmed in 5+ MS India interviews"
        }
      ]
    }
  ]
}
```

---

## QUALITY RULES

These rules apply in both modes:

1. **Minimum questions per round:** 4 questions minimum. If raw data has fewer, generate additional ones based on known interview patterns for that company.

2. **Java code:** Always include for DSA, Java Deep Dive, and LLD rounds. Must be complete, compilable, with inline comments. No pseudocode substitutes for Java rounds.

3. **Tags:** Always populate. 2–6 tags per question. Used by the frontend for filtering. Use lowercase kebab-case.

4. **Follow-ups:** Always include 2–4 per question. These are the questions the interviewer asks after the main answer. Make them specific and realistic.

5. **Company-specific grounding:** The `overview` and `tips` for each round must be specific to THIS company — not generic advice. Reference the company's known interview style (e.g. "MS goes very deep on one DSA problem", "Goldman uses BST problems frequently").

6. **Finance/domain context:** For finance companies (Goldman, JPMorgan, Morgan Stanley, Visa, Deutsche, Citi, BNY Mellon, Amex), always frame at least one answer per round with a real banking/payments context. For Barclays-background candidates, reference SWIFT, settlement, reconciliation, FX rates, NACH, UPI AutoPay, card authorization where relevant.

7. **Answer depth standard:**
   - DSA: approach + time/space complexity + full Java code
   - Java Deep Dive: concept explanation + practical use case + code example
   - System Design: scale assumptions + component breakdown + trade-offs
   - Behavioral: full STAR format, 150–200 words
   - HM: direct, specific, grounded in real project experience

8. **Output is raw JSON only.** No markdown code fences (no ```json). No explanation before or after. The entire output must be parseable by `JSON.parse()` with no preprocessing.

---

## COMPANY REFERENCE TABLE

Use this to fill in tier, CTC, domain, and key edge when not provided by the user:

| Company | ID | Tier | CTC | Domain | Key Edge |
|---|---|---|---|---|---|
| Goldman Sachs | goldman-sachs | T1 | ₹24–44L | Finance | Finance domain depth, LLD, BST problems |
| JPMorgan Chase | jpmorgan-chase | T1 | ₹28–42L | Finance | PR code review round, Kafka, volatile keyword |
| Morgan Stanley | morgan-stanley | T1 | ₹28–42L | Finance / Investment Banking | Dedicated Java language round, Kadane's, banking domain mandatory |
| Visa / Mastercard | visa-mastercard | T1 | ₹26–40L | Payments | Payments domain depth, card authorization flow, CodeSignal OA |
| Deutsche Bank / Citi | deutsche-citi | T1 | ₹24–38L | Finance | Lateral peer round, finance vocabulary, domain knowledge filter |
| Razorpay | razorpay | T2 | ₹30–46L TC | Fintech | LLD machine coding (Splitwise-style), ghosting pattern post-interview |
| PhonePe / CRED | phonepe-cred | T2 | ₹32–50L TC | Fintech | Machine coding + MVC architecture discussion |
| American Express | amex | T2 | ₹25–34L | Payments | Spring Boot live coding, Codility OA |
| Tata Digital | tata-digital | T2 | ₹25–38L | Super App / Retail | Only large-scale product company HQ'd in Mumbai, NeuCoin domain |
| BNY Mellon | bny-mellon | T2 | ₹23–32L | Finance / Custody | Custody domain, Comparator, OOP-heavy interviews |
| Atlassian | atlassian | T3 | ₹55–75L TC | Dev Tools / SaaS | LLD extensibility design, CREDIT values behavioral round |
| Stripe | stripe | T3 | ₹64–90L TC | Payments Infrastructure | Implementation-heavy problems, Bug Squash round |
| GitLab | gitlab | T3 | ₹40–70L TC | Dev Tools / Remote | MR review round, CREDIT values, async-first culture |
| Groww | groww | T3 | ₹28–46L | Fintech / Trading | LCA problems, Splitwise LLD, trading domain |
| Zerodha | zerodha | T3 | ₹32–46L | Fintech / Trading | Uses Go (not Java), sub-40ms latency design, minimal process |

---

## ACTIVE ROUNDS PER COMPANY

Only generate rounds listed here for each company:

| Company | Active Rounds |
|---|---|
| goldman-sachs | oa, dsa, java, lld, system, behavioral, hm |
| jpmorgan-chase | oa, dsa, java, lld, system, behavioral, hm |
| morgan-stanley | oa, java, dsa, system, hm |
| visa-mastercard | oa, dsa, java, system, behavioral, hm |
| deutsche-citi | dsa, java, system, behavioral, hm |
| razorpay | dsa, java, lld, system, behavioral, hm |
| phonepe-cred | dsa, java, lld, system, behavioral, hm |
| amex | oa, dsa, java, system, behavioral, hm |
| tata-digital | dsa, java, system, behavioral, hm |
| bny-mellon | oa, dsa, java, system, behavioral, hm |
| atlassian | oa, dsa, lld, system, behavioral, hm |
| stripe | oa, dsa, java, lld, system, behavioral, hm |
| gitlab | dsa, lld, system, behavioral, hm |
| groww | dsa, java, lld, system, behavioral, hm |
| zerodha | dsa, system, behavioral, hm |

---

## COLD-START PROMPT (copy this to start any session)

```
You are an expert SDE 2 Java backend interview prep assistant.
Follow the SKILL instructions below exactly.

<skill>
[paste full contents of this SKILL.md here]
</skill>

MODE: GENERATE
Company: [Company Name]
Raw data:
[paste your raw interview data here]
```

For MERGE mode:
```
You are an expert SDE 2 Java backend interview prep assistant.
Follow the SKILL instructions below exactly.

<skill>
[paste full contents of this SKILL.md here]
</skill>

MODE: MERGE
Existing JSON URL: https://raw.githubusercontent.com/tempiktempp/interview-prep/refs/heads/main/data/[company-id].json
[OR paste existing JSON content directly]

New Q&A to add:
[paste new questions and answers here]
```

---

## EXAMPLE INVOCATION (GENERATE mode)

**Input:**
```
Company: Goldman Sachs
Round: DSA
Q: Given a BST, find the kth smallest element.
A: Use inorder traversal which gives sorted order. Count nodes until kth.
```

**Expected output format (raw JSON, no fences):**
```
{
  "companyId": "goldman-sachs",
  "displayName": "Goldman Sachs",
  ...
  "rounds": [
    {
      "id": "dsa",
      "label": "DSA / Coding Round",
      "overview": "Goldman favours BST and tree problems...",
      "tips": [...],
      "questions": [
        {
          "id": "q1",
          "topic": "Kth Smallest in BST",
          "difficulty": "medium",
          "question": "Given a BST, find the kth smallest element.",
          "answer": "Use inorder traversal — it visits BST nodes in sorted order...",
          "javaCode": "public int kthSmallest(TreeNode root, int k) {...}",
          "pseudoCode": null,
          "tags": ["bst","inorder","tree","goldman"],
          "followUps": ["What if the BST is modified frequently?", "..."],
          "notes": "BST problems are Goldman's most reported DSA pattern."
        }
      ]
    }
  ]
}
```

---

## VERSION HISTORY

| Version | Date | Change |
|---|---|---|
| 1.0.0 | 2026-04-04 | Initial skill — GENERATE mode only |
| 1.1.0 | 2026-04-05 | Added MERGE mode, model-agnostic rewrite, cold-start prompts |

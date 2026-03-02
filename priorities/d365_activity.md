
---

## 📄 文件 2：`priorities/TEMPLATE.md`（加权优先级解码模板）

```markdown
## 🎯 Priority Log: [Project Name]
**Date**: YYYY-MM-DD  
**Source**: [Meeting/Email/Observation]  
**Status**: [🔍 Exploring / ✅ Validated / ❌ Archived]

---

### Surface Request (Verbatim)
> "I need to understand which team members are using D365 CRM and what actions they perform under what entities"
> "I need a set of indicators to measure the activity level of D365 CRM accounts"
> "I need a web dashboard that shows the activity level of D365 CRM accounts"

*Example: "I need to see which team members are using D365 CRM and what features they use"*

---

### Implicit Signals (Weighted Analysis)

| Signal | Weight | Your Observation | Evidence/Quote |
|--------|--------|-----------------|----------------|
| 🔁 Repeated keywords | 40% | | *"I need a summary as IT head", "business leadership visibility", "CRM license optimization"* |
| ❓ Skipped questions | 40% | | *Didn't have a clear idea about the set of indicators, only ask "the result of CRM accounts"* |
| 💰 Resource commitment | 30% | | *"IT head can do this alone", "no need to involve other teams"* |
| 😰 Expressed anxiety | 20% | | *"Worried about CRM license optimization without usage data"* |

**Signal Quality Check**:
- [ ] Are "repeated keywords" consistent with "skipped questions"? (If no, flag for clarification)
**一致**：强调「快」+ 跳过「细节」= 逻辑自洽，无需澄清

- [ ] Is there a gap between stated request and inferred priority? (Document why)
**小差距**：需要通过看板来研判续费的依据
---

### Inferred True Priority
> "What they really need is: ______"

*Example: "Not a perfect dashboard, but MVP evidence that proves D365 value to support renewal decisions"*

**Confidence Level**: [High / Medium / Low]  
**Rationale**: [Why you believe this inference]

---

### MVP Validation Plan
> "If this priority is correct, the smallest test would be: ______"

*Example: "Build a single-page report showing top 3 used features for 1 team, deliver in 3 days, measure stakeholder reaction"*

**Success Metric**: [How you'll know the priority was correct]  
**Timebox**: [Max time to spend on validation]

---

### Technical Feasibility Snapshot
| Component | Status | Notes |
|-----------|--------|-------|
| Data source | [✅ Available / ⚠️ Needs research / ❌ Blocked] | e.g., "D365 API docs reviewed" |
| AI capability | [✅ Prompt ready / ⚠️ Needs testing / ❌ Unclear] | e.g., "Azure OpenAI can summarize usage logs" |
| Delivery channel | [✅ Web / ✅ Email / ⚠️ Other] | e.g., "Simple React page on Azure App Service" |

**Overall Feasibility**: [✅ High / ⚠️ Medium / ❌ Low]

---

### AI Calibration (Optional but Recommended)
*Use `.ai/PROMPT_PRIORITY.md` to generate AI's priority guess, then compare.*

- [ ] AI's top priority hypothesis: [Paste output]
- [ ] Matched my inference? [✓ Yes / ✗ No]
- [ ] Hallucination check: Did AI assume unverified technical capabilities? [✓ Yes / ✗ No]
- [ ] Calibration note: *"Next time, check ______ first"*

---

### Reusability Assessment
**Similar historical scenarios**: [Link to `scenarios/*.md` if applicable]  
**Abstractable pattern**:  
> "When stakeholders need [X data] + [Y frequency] + [Z audience], consider [this approach]"

**Tags**: `#D365` `#Usage-Metrics` `#Leader-Reporting` `#MVP-First`

---

### Attachments & Links
- [ ] GitHub repo: `github.com/mcaserhn/[repo-name]`
- [ ] Prompt library: `.ai/PROMPT_*.md`
- [ ] Decision log: `decisions/[related-decision].md`
- [ ] Meeting notes: `meetings/[date].md`

---

*Last updated: YYYY-MM-DD | Next review: [Date for priority validation check]*
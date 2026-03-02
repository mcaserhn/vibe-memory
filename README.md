# 🧠 Vibe Memory System

> Personal knowledge system for AI-augmented full-stack development.  
> Focus: Scenario intuition × Decision calibration × Reusable patterns.

## 🎯 Core Philosophy
- **Record what matters**: Business priority signals > Code snippets
- **Calibrate, don't delegate**: Human reviews AI reasoning, not just output
- **Reuse by scenario**: Atomic steps → Verified components → Scenario cards

## 📁 Repository Structure
‘‘‘
vibe-memory/
├── .ai/ # AI prompt templates with your calibration weights
│ ├── PROMPT_PRIORITY.md # Business priority decoder (40/40/30/20 weights)
│ ├── PROMPT_MEETING.md # Meeting notes → repetition signal extractor
│ └── PROMPT_COMPONENT.md # Code → verified component generator
│
├── priorities/ # Business priority decoding logs (CORE)
│ ├── TEMPLATE.md # Weighted priority log template
│ └── *.md # One file per需求/项目
│
├── scenarios/ # Verified scenario cards (reusable assets)
│ ├── TEMPLATE.md # Scenario card with scorecard + decision log
│ └── .md # One file per completed project
│
├── decisions/ # Human vs AI decision audits
│ ├── TEMPLATE.md # Decision review with hallucination check
│ └── weekly_review_.md # Weekly calibration reflections
│
├── signals/ # Raw repetition signal logs
│ ├── TEMPLATE.md # "Repetitive manual step" observation template
│ └── *.md # One file per observed signal
│
├── components/ # Verified reusable components
│ ├── verified/ # Steps that passed real-world testing
│ └── drafts/ # Components pending validation
│
└── README.md # This file
’’’

## 🔑 Your Implicit Knowledge (Calibration Guide)

### Priority Signal Weights (Your Intuition Encoded)
| Signal | Weight | Why It Matters |
|--------|--------|---------------|
| 🔁 Repeated keywords | 40% | Stakeholders reveal true priority through repetition |
| ❓ Skipped questions | 40% | What they DON'T ask reveals what they truly value |
| 💰 Resource commitment | 30% | Willingness to invest signals genuine priority |
| 😰 Expressed anxiety | 20% | Fear points to accountability, not necessarily value |

### Decision Rule (Your Hallucination Shield)
> "Pause when AI scores high. Check: Is reasoning based on **verified facts** or **training data hallucination**?"

### Vibe Coding Definition (Your North Star)
> "Helping non-coders become advanced full-stack developers by:  
> 1. Identifying high-leverage business scenarios  
> 2. Using AI to bridge capability gaps  
> 3. Delivering complete systems (independent or simple integrations)"

## 🔄 Sustainable Daily Flow (30 min/day)
| Time | Action | Output |
|------|--------|--------|
| 🌅 5min morning | Confirm AI drafts in `.ai/` | Updated `priorities/*.md` |
| 🌤️ 15min during dev | Voice-log decisions → text | `decisions/*.md` entries |
| 🌙 10min evening | Review: "What did AI hallucinate? What did I correct?" | `decisions/weekly_review_*.md` |

## 🛡️ Risk Management
### Scenario Scorecard (Risk-Adjusted)
Base Score (0-10):
Complexity (1-4) + Flexibility (1-3) + Integration (1-3)
Risk Deductions:
Business rules unclear: -0 to -2
Unverified API/system: -0 to -2
Scope creep risk: -0 to -1
Final Decision:
≥8: ✅ Green light | 5-7: ⚠️ MVP first | ≤4: ❌ Pause/clarify


### Correction Protocols (Your Anti-Hallucination Tools)
1. **Doc-Feeding**: Force AI to code from official docs, not training data
2. **Step-Verify**: Break integrations into independently testable steps

## 🌍 Compliance & Localization
- **Hosting**: GitHub.com + Azure Singapore (avoid mainland China infrastructure)
- **Naming**: English variable names throughout
- **Content**: Zero references to mainland China entities, regulations, or markets
- **Team**: International collaboration ready (async-first documentation)

## 🚀 Quick Start
```bash
# Clone and start
git clone https://github.com/mcaserhn/vibe-memory
cd vibe-memory

# Fill your first priority log (3 min)
code priorities/d365_activity.md  # or your preferred editor

# Commit and push
git add . && git commit -m "feat: first priority log" && git push

📈 Evolution Protocol
This system evolves THROUGH use, not through design.
Week 1: Record only priorities/ + signals/ (minimal friction)
Week 2: Add decisions/ audits (calibrate AI)
Week 3: Extract first components/ (build reusable assets)
Month 2+: Review weekly_review_*.md to refine weights/prompts
"The best system is the one you actually use. Start small, calibrate often, scale intentionally."
# Pipeline: Text Analysis & Summary

## Project
- **Name**: backend1
- **Output folder**: backend1/
- **Description**: Analyze a piece of text to extract themes, sentiment, and key points, then synthesize into a concise summary with a recommendation.

---

## Initial State
```
raw_input: str   # The text to analyze (injected at session creation)
```

---

## Agents

### 1. AnalysisAgent
- **Type**: BaseAgent wrapping one LlmAgent
- **LlmAgent field name**: `analyzer_llm`
- **Reads**: `raw_input`
- **Writes**: `analysis`
- **Output schema**:
  - `themes: list[str]` — Main themes identified in the text
  - `sentiment: Literal["positive", "negative", "neutral", "mixed"]` — Overall sentiment
  - `key_points: list[str]` — Key facts or observations extracted from the text
- **Instruction summary**: Text analysis expert. Extract themes, sentiment, and key points from `{raw_input}`.

---

### 2. SummaryAgent
- **Type**: BaseAgent wrapping one LlmAgent
- **LlmAgent field name**: `summarizer_llm`
- **Reads**: `analysis`
- **Writes**: `summary`
- **Output schema**:
  - `summary: str` — Concise 2-3 sentence summary of the analysis findings
  - `recommendation: str` — Single clear, actionable recommendation based on the analysis
- **Instruction summary**: Synthesis expert. Read `{analysis}`, produce a concise summary and one actionable recommendation.

---

## Pipeline

```
Sequential:
  AnalysisAgent → SummaryAgent
```

State flow:
```
raw_input  →  AnalysisAgent  →  analysis
analysis   →  SummaryAgent   →  summary
```

---

## Test Input (for main.py)
```
"The new product launch exceeded expectations with 50,000 units sold in the first week.
Customer feedback has been overwhelmingly positive, praising the intuitive interface and
competitive pricing. However, supply chain issues caused delivery delays for 15% of orders,
leading to some negative reviews. The marketing campaign on social media generated 2M impressions
with a 4.5% conversion rate, well above the 2% industry average."
```

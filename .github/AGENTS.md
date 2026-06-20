# Workspace Custom Agents

This workspace defines a set of custom agents for trading strategy development, indicator validation, and code quality.

## Custom Agents

- `triad-trading.agent.md`
  - Coordinates all specialist agents.
  - Purpose: produce disciplined buy/sell indicator logic by combining devil's advocate, professional analyst, and emotion analyst perspectives.
  - Uses: `devils-advocate`, `professional-analyst`, `emotion-analyst`, `coding-skills`, `indicator-runner`.

- `devils-advocate.agent.md`
  - Plays the negative side to find worst-case failure modes, signal weaknesses, and defensive checks.

- `professional-analyst.agent.md`
  - Applies cold logic, market structure reasoning, and institutional trading principles to validate strategy quality.

- `emotion-analyst.agent.md`
  - Evaluates emotional bias and crowd behavior, then reframes strategy into disciplined, emotion-neutral terms.

- `coding-skills.agent.md`
  - Separate, code-focused agent that revises and edits indicator implementation based on other agents’ reasoning.
  - Works alongside the Indicator Runner to ensure the trading indicator is smooth and accurate.

- `indicator-runner.agent.md`
  - Translates strategy outputs into continuous indicator logic and validates signal behavior for chart integration.
  - Acts as the main synthesizer of final indicator output using inputs from the reasoning and coding agents.

- `market-scout.agent.md`
  - Continuously searches for market-related information tied to names in the workspace and builds a summary feed for the Newscaster.

- `newscaster.agent.md`
  - Converts market summaries into bullish/bearish overviews and a prioritized list of considerations for reasoning agents.

- `trading-strategy.agent.md`
  - Support agent for building custom trading strategy guidance and buy/sell marker design.

## Workflow

1. `triad-trading` runs the three reasoning agents together.
2. It integrates their outputs into a final recommendation.
3. `coding-skills` revises implementation details.
4. `indicator-runner` validates and updates the indicator logic.

## Notes

- The workspace instruction file `./.github/instructions/triad-trading.instructions.md` ensures this workflow is applied during trading-related code development.
- These agents are intended to reduce mistakes, plan for worst-case scenarios, and create a profitable, disciplined market indicator strategy.

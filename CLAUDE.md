# Edgewonk Trading Journal Connector

This project connects Edgewonk trading journal exports to Claude for analysis.

## How This Works

The user is a day trader who journals in Edgewonk. They export CSV files from
Edgewonk and place them in the `edgewonk-data/` folder. Claude reads those CSVs
and helps analyze trading performance.

## Edgewonk CSV Format

Edgewonk exports typically include these columns (names may vary slightly by version):

- **Trade #** — sequential trade number
- **Date/Time Open** — when the trade was entered
- **Date/Time Close** — when the trade was exited
- **Symbol/Instrument** — what was traded (e.g., ES, NQ, EURUSD)
- **Direction** — Long or Short
- **Entry Price** — price at entry
- **Exit Price** — price at exit
- **Stop Loss** — stop loss level
- **Take Profit** — take profit level
- **P&L** — profit or loss in currency
- **P&L (Pips/Points)** — profit or loss in pips or points
- **R-Multiple** — result expressed as a multiple of risk (R)
- **Commission** — trading fees
- **Setup** — the trade setup/strategy name (user-defined in Edgewonk)
- **Mistakes** — tagged mistakes (user-defined in Edgewonk)
- **Custom Tags** — any custom fields the trader added
- **Notes** — free-text trade notes
- **Emotions** — emotional state tags
- **Rating** — trade quality rating

## How to Analyze

When the user asks about their trading, always:

1. Read the CSV files in `edgewonk-data/`
2. Parse them as CSV (handle both comma and semicolon delimiters — Edgewonk uses both depending on locale)
3. Focus on actionable insights, not just statistics
4. Reference specific trades when making observations
5. Be encouraging but honest — the goal is improvement

## Common Analysis Requests

- **Performance summary**: Win rate, average R, profit factor, expectancy
- **Setup analysis**: Which setups are most/least profitable
- **Mistake patterns**: Which mistakes cost the most money
- **Time analysis**: Best/worst times of day, days of week
- **Emotional patterns**: How emotions correlate with results
- **Risk management**: Are stops being honored? Position sizing consistency?
- **Streak analysis**: Winning/losing streaks and behavior during them

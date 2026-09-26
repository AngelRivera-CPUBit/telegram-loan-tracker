# Personal Loan Tracker Bot

A Telegram bot + n8n automation to track money lent to other people, paired with a Python/pandas notebook that generates an aggregated financial summary from the same data. Built as a second hands-on automation project, extending the pattern used in the [expense tracker bot](https://github.com/AngelRivera-CPUBit/telegram-expense-tracker).

> **Note on data:** all names shown in this README and in the underlying spreadsheet are fictitious — this project tracks real personal loans, so example data was anonymized before publishing.

## The problem

Informally lending money to multiple people (family, friends) gets hard to track over time: who owes what, how much interest applies, and how much is expected back in total. This project turns that into a structured, queryable record instead of scattered mental notes.

## How it works — part 1: data capture (n8n)

```
Telegram message  →  n8n Telegram Trigger  →  Code node (parses text + calculates interest/total/payment)  →  Google Sheets (append row)  →  Telegram confirmation reply
```

1. **Telegram Trigger** — listens for any message sent to the bot.
2. **Code node (JavaScript)** — parses the message (`"Name Amount"`) and calculates:
   - `interest` — a flat 65% of the loaned amount.
   - `total` — amount + interest.
   - `weekly_payment` — total divided into 11 installments.
3. **Google Sheets** — appends the record as a new row.
4. **Telegram response** — confirms the registered loan with the calculated breakdown.

## How it works — part 2: analysis (Python + pandas)

A companion [Google Colab notebook](./analisis_prestamos.ipynb) reads the same spreadsheet (published as CSV) and produces an aggregated summary:

```python
import pandas as pd

df = pd.read_csv(url)
df = df.dropna(axis=1, how="all")  # drop empty helper columns

# Total lent per person (handles repeat borrowers)
por_persona = df.groupby("name")["amount"].sum()

# Total pending to collect, including interest
deuda_total = df["total"].sum()

# Total expected profit if everyone pays back in full
ganancia_total = df["profit"].sum()
```

This complements the spreadsheet's own running-total columns (built with `ARRAYFORMULA`) by answering a different question: not "what's the cumulative total row by row," but "what's the summary grouped by person, on demand."

## Example

**Input (sent to the bot):**
```
Juan 10000
```

**Bot reply:**
```
✅ Registrado: Juan - $10000
Interés: $6500
Total: $16500
Pago semanal: $1500
```

**Notebook output:**
```
Total pendiente de cobrar (con interés): $28050
Ganancia esperada total: $5525
```

## Tech stack

- **n8n** — workflow automation (capture + calculation + confirmation)
- **Telegram Bot API** — user input interface
- **Google Sheets** — data storage, plus native `ARRAYFORMULA` running totals
- **Python (pandas)** — aggregated analysis, run in Google Colab
- **JavaScript** — parsing and interest/payment calculation (n8n Code node)

## Setup

1. Create a bot via [@BotFather](https://t.me/botfather) and save the API token.
2. Build the n8n workflow: Telegram Trigger → Code (parse + calculate) → Google Sheets (append row) → Telegram (reply).
3. Publish the Google Sheet to the web as CSV (`File → Share → Publish to web`) to make it readable from an external notebook.
4. In Google Colab, load the data with `pandas.read_csv(<published-csv-url>)` and run the aggregation cells.

## Planned improvements

- Command to mark a loan as fully paid (status tracking).
- Automatic weekly reminder of upcoming payments due.
- Per-person payment history, not just the original loan amount.
- Move the pandas analysis into a scheduled n8n step, so the summary itself gets sent automatically instead of running the notebook manually.

## What I learned

- Applying the same automation pattern (trigger → process → store → confirm) to a second, different use case.
- Basic financial calculations inside a JavaScript automation node.
- Reading external data into Python with `pandas.read_csv`, and the difference between a **function** (`pd.read_csv()`) and a **method** (`df["total"].sum()`).
- Data aggregation with `groupby()` as the pandas equivalent of a manual accumulator pattern.
- Handling shared spreadsheet data safely — publishing only what's needed for analysis, keeping personally identifying data out of anything public.

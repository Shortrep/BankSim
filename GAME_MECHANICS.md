# Bank Sim mechanics

This document describes the mechanics currently implemented in the simulation.
Each call to `run_full_year` runs one game year.

## World and starting state

The default initial conditions create a year-0 economy with:

- Two banks: Bank Blue and Bank Red. Each starts with 50 cash.
- Eight people, Person A through Person H. Each starts aged 20 with yearly income of 10 and no cash.
- Four firms: Firm Blue, Firm Red, Firm Yellow, and Firm Green. Each starts with 10 cash and zero capital.
- Each bank has 1,000 shares outstanding. Every person initially owns 125 shares in each bank.

The initial-condition definitions are in `initial_conditions/initial_conditions.py`.

## Actors

### Banks

Banks hold cash and make three types of loans: business, personal, and home loans. They can fund themselves with deposits and fixed-maturity wholesale bonds. A bank CEO selects dividends, lending allocations, loan and deposit rates, broker instructions, and bond issuance offers each year.

The CEO's objective is to maximise shareholder value through expected future dividends. Each bank also has a separate loan-agent model that accepts or rejects applications within the CEO's allocated lending amount.

### People

People earn yearly income until retirement at age 60. They age by one year each simulation year and die at age 80. Dead people no longer make decisions or earn income.

Each person decides consumption, deposits, financial-market orders, a possible loan application, and housing actions. Their stated objectives are consumption smoothing, buying a good house, and retirement saving.

### Firms

Firms have cash and productive capital. They do not own houses, equities, or bonds. Each year they choose investment, labour expenditure, cash reserve, and an optional business-loan application.

Firm output follows Cobb-Douglas production:

```text
output = capital^0.5 × labour^0.5
```

Investment costs cash in the current year and becomes productive capital next year. It does not increase current-year output. Capital depreciates by 10% each year.

## Financial instruments

### Cash

Cash is the immediately spendable balance used for consumption, deposits, purchases, labour, investment, loan payments, and settlements.

### Bank equity

Each bank has exactly 1,000 shares. Equity holders receive any bank dividend pro rata. A CEO selecting total dividends `D` therefore pays `D / 1,000` per share.

Equities trade in a limit-order market between people. A trade requires a buyer and seller for the same bank, with bid price at least equal to ask price. Trades execute at the seller's ask price.

### Risk-free bonds

Risk-free bonds are zero-coupon instruments issued outside the bank sector. They pay face value only at maturity. The hard-coded annual risk-free curve is:

| Tenor | Annual rate |
| --- | --- |
| 1 year | 3.0% |
| 2 years | 3.2% |
| 3 years | 3.4% |
| 4 years | 3.6% |
| 5 years | 3.8% |
| 6 years | 4.0% |

For face value `F`, annual rate `r`, and tenor `t`, the price is:

```text
price = F / (1 + r)^t
```

Risk-free purchases and sales settle automatically at this price. At maturity, the holder receives face value without any bank being charged.

### Bank bonds

Bank bonds are also zero-coupon, fixed-maturity claims. A bank CEO can issue bonds by posting an offer with a tenor, quantity, and price per face-value unit. Issuance raises bank cash only when a person submits a compatible bid.

Secondary bank-bond trades require the same issuing bank and tenor, with a bid price at least equal to the ask price. Trades execute at the ask price. At maturity, the issuing bank must pay face value to the holder.

### Deposits

People may deposit cash with one bank at that bank's deposit rate. Deposits accrue interest annually, are liabilities of the bank, and can be withdrawn at any time.

A person's deposit decision specifies their total desired deposit at a chosen bank. Leaving the same bank and amount unchanged keeps the deposit in place. Switching, resizing, or withdrawing settles the old deposit first and then creates the new one if applicable.

### Loans

Every loan has one debtor, one bank creditor, an interest rate, a maturity year, and a fixed annual repayment. The repayment is calculated as an amortising annuity, so equal yearly payments repay all interest and principal by the end of the term:

```text
payment = principal × rate / (1 - (1 + rate)^(-years))
```

The first repayment occurs in the year after the loan is granted. A loan granted at the end of year 0 first repays in year 1.

Business and personal loans have one- to six-year terms. A home loan always has a 30-year term.

### Home-loan approvals and houses

A home-loan application creates an approval, not an immediate cash payment to the person. If the person buys a house, the bank pays up to the approved amount toward the purchase price and the purchased house becomes collateral. The buyer pays any price above the approved amount from cash. If the house costs less than the approved amount, the unused approval is not disbursed.

Houses have an integer quality and one owner. Only people and banks may own houses.

## Decisions and cash constraints

LLMs produce structured decisions for bank CEOs, loan agents, people, firms, and house sales.

For a person, immediate cash use includes consumption, an increase in deposits, equity bids, bond bids, and the cash component of a house bid. The simulation treats every bid as though it may execute in full. If the combined immediate requirement exceeds available cash, the person's full financial decision set is rejected and replaced with no financial action. A loan application does not provide spendable cash in that same year.

For a firm, investment plus labour may not exceed cash. If it does, the whole firm decision is replaced with no action.

Execution code also contains affordability guards so an invalid order cannot create negative cash if it bypasses decision validation.

## Year sequence

For game year `t`, the simulation proceeds in this order:

1. Reset the event log and save the prior bank settings.
2. Apply production and transfers:
   - pay person income and age people;
   - accrue deposit interest;
   - collect scheduled loan repayments;
   - settle maturing bonds;
   - pay dividends;
   - run firm production, depreciation, investment, and labour spending.
3. Resolve insolvencies caused by those transfers.
4. Ask each bank CEO for its decisions and publish rates.
5. Ask each living person whether to list a house, then ask for their full decisions.
6. Apply person consumption and deposit decisions.
7. Ask firms for their decisions.
8. Clear equity, fixed-income, and housing markets.
9. Ask each bank's loan agents to accept or reject applications.
10. Resolve any new insolvencies.
11. Write the year event log, record history, advance the game year, and save the next start-state pickle.

## Markets

### Equity market

For each bank, equity bids sort from highest to lowest and asks from lowest to highest. Matching continues while the best bid is at least the best ask. The seller's ask is the trade price. A seller cannot sell more shares than they own, and a buyer cannot spend more cash than they have.

### Fixed-income market

Risk-free orders settle automatically at the hard-coded curve. Bank-bond issuance and secondary trading use price-time matching by issuer and tenor. A buyer must have enough cash for the full matched quantity.

### Housing market

Each house clears against the highest compatible bid, at the seller's asking price. A buyer must cover the price from cash and any valid home-loan approval.

## Defaults and game over

If a bank cannot meet an obligation, the game ends. The event output records the failure reason and the end-state pickle is saved.

If a person or firm cannot meet an obligation or has negative cash:

1. Their deposits are withdrawn, if the bank can pay them.
2. All of their loans are destroyed.
3. All bonds they own are destroyed.
4. Their remaining cash is distributed among creditor banks in proportion to outstanding loan balances.
5. Their cash is set to zero.

## Save files and reports

Games are stored under `games/<game name>/`.

```text
year_0/game.pkl   initial state
year_0/output.txt events and decisions for year 0
year_1/game.pkl   state at the end of year 0 / start of year 1
```

Every subsequent year follows the same pattern. `run_game.py` starts a fresh game; `continue_game.py` loads the latest saved state and runs one further year.

Run:

```bash
python3 generate_game_report.py <game name>
```

to generate an offline `report.html` for the game's saved states and event logs.

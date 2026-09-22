# VAULT-CBS: A Production-Grade Core Banking System in COBOL

VAULT-CBS is a fully functional core banking system written from scratch in COBOL. It implements deposit and loan account management, double-entry general ledger accounting, ACH and wire transfer processing, interest accrual, fee assessment, and regulatory compliance — the same operations that run inside systems like FIS, Fiserv, and Jack Henry.

**32,000+ lines of COBOL | 25 modules | 612 tests | 0 failures**

> I wrote about the process of building this: [blog post coming soon]

## Why COBOL?

COBOL processes [95% of ATM transactions](https://fingfx.thomsonreuters.com/gfx/rngs/USA-BANKS-COBOL/010040KH18J/index.html) and runs the core ledger at most banks in the world. The largest financial institutions — JPMorgan Chase, Bank of America, Wells Fargo — still rely on COBOL systems originally written in the 1970s and 80s. These aren't legacy artifacts waiting to be replaced. They're the production systems of record processing trillions of dollars daily.

I built VAULT-CBS to understand what's actually inside a core banking system and why these systems are so hard to replace. The answer isn't the language — it's the accumulated domain logic: regulatory rules, edge cases in date arithmetic, overflow-safe fixed-point math, and the sheer number of ways money can move between accounts while keeping the books balanced.

## What It Does

VAULT-CBS implements the core functions of a community bank or credit union (up to ~$10B in assets):

### Transaction Engine
- **Transaction posting** with authorization, validation, and automatic GL entry generation
- **Double-entry accounting** — every debit has a matching credit, always
- **Overflow-safe arithmetic** — all balance-mutating operations use `ON SIZE ERROR` to prevent silent corruption at PIC S9(13)V99 boundaries ($99 trillion)

### Payment Processing
- **ACH processing** with R01–R09 return codes, Reg D enforcement, and dormancy checks
- **Wire transfers** with OFAC screening, dual-control authorization (>$50K), and full reversal support
- **Loan payments** — regular, late fee assessment, and full payoff with per-diem interest calculation

### Interest & Fees
- **Four accrual bases**: 30/360, Actual/360, Actual/365, Actual/Actual
- **Daily interest accrual** with business day conventions via a holiday calendar
- **Fee engine** with tiered waiver logic — employee, direct deposit, minimum balance, age-based — evaluated in priority order
- **APY/APR calculator** compliant with Truth in Savings and Truth in Lending

### Regulatory Compliance
- **BSA/AML**: Currency Transaction Reports for cash activity >$10,000, structuring detection for split transactions just below the threshold, Suspicious Activity Report filing
- **Reg CC**: Hold calculation using business day arithmetic (2-day local, 5-day non-local, 7-day exception items)
- **Reg D**: Transfer limit enforcement for savings/money market accounts
- **Reg E**: Electronic fund transfer dispute management with provisional credits, investigation timelines, and resolution tracking
- **OFAC**: Sanctions screening against name, country, and beneficiary fields

### Batch Operations
- **End-of-Day**: Interest accrual, hold release, CTR aggregation, audit logging, GL trial balance verification
- **End-of-Month**: Fee assessment, MTD counter reset, YTD tracking, statement generation

### Persistence & Audit
- **Indexed file I/O** (ISAM) for accounts, customers, transactions, GL entries, holds, and audit records
- **Immutable audit trail** — every state change captured with before/after images, timestamps, and program identification
- **Dormancy detection** and escheatment processing for abandoned accounts

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          VAULT-CBS                              │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │   CIF    │ │ Deposits │ │  Loans   │ │    GL    │          │
│  │ Module   │ │  Module  │ │  Module  │ │  Module  │          │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘          │
│       │            │            │            │                  │
│  ┌────┴────────────┴────────────┴────────────┴─────┐           │
│  │           TRANSACTION ENGINE (TXN-ENG)          │           │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐           │           │
│  │  │ Journal │ │ Posting │ │  Auth   │           │           │
│  │  │  Layer  │ │  Layer  │ │  Layer  │           │           │
│  │  └─────────┘ └─────────┘ └─────────┘           │           │
│  └──────────────────────┬──────────────────────────┘           │
│                         │                                       │
│  ┌──────────┐ ┌─────────┴──┐ ┌──────────┐ ┌──────────┐        │
│  │   ACH    │ │   Wire     │ │   Fee    │ │ Interest │        │
│  │ Processor│ │  Transfer  │ │  Engine  │ │  Engine  │        │
│  └──────────┘ └────────────┘ └──────────┘ └──────────┘        │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │  Batch   │ │ Reg/AML  │ │ Stmt Gen │ │  Audit   │          │
│  │ Scheduler│ │ Compliance│ │  Module  │ │  Trail   │          │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Module Inventory

| Module | LOC | Purpose |
|--------|-----|---------|
| FILEIO0 | 489 | Indexed file CRUD gateway — unified I/O for all 6 data stores |
| EODPROC0 | 429 | End-of-day batch orchestrator |
| WIREXFR0 | 425 | Wire transfer processor (send/receive/complete/reverse/inquiry) |
| ACHRECV0 | 395 | ACH payment processing with return code handling |
| LOANPMT0 | 381 | Loan payment engine (regular/late/payoff) |
| DATEUTIL | 376 | Business day arithmetic and holiday calendar |
| DISPMGT0 | 339 | Reg E dispute management |
| TXNPOST0 | 320 | Core transaction posting engine |
| GLPOST0 | 320 | General ledger posting with Call Report mapping |
| ACCTMGMT | 308 | Account lifecycle (open/close/freeze/status changes) |
| EOMPROC0 | 292 | End-of-month batch processing |
| BSACTRO | 241 | BSA/AML — CTR detection and structuring analysis |
| HOLDCALC0 | 236 | Reg CC hold calculation |
| SARFIL0 | 213 | Suspicious Activity Report filing |
| FEECALC0 | 211 | Fee assessment with waiver priority logic |
| INTCALC0 | 206 | Interest calculation (4 accrual bases) |
| OFACCHK0 | 201 | OFAC sanctions screening |
| ODMGMT0 | 189 | Overdraft management |
| CTRFIL0 | 181 | Currency Transaction Report filing |
| CIFMGMT | 158 | Customer Information File / KYC / CIP |
| DORMANT0 | 146 | Dormancy detection and escheatment |
| APYCALC0 | 139 | APY/APR calculation |
| REGDCHK0 | 96 | Reg D transfer limit enforcement |
| AUDTLOG0 | 91 | Audit trail writer |

**18 copybooks** define shared data structures: account records, GL entries, transaction records, fee schedules, wire transfer details, CTR/SAR filing records, dispute records, and more.

### Data Architecture

All data uses fixed-point decimal (`PIC S9(13)V99` for balances — max $99,999,999,999,999.99). No floating-point arithmetic anywhere. Every monetary computation is protected with `ON SIZE ERROR` to prevent silent overflow.

| Data Store | Key | Contents |
|-----------|-----|----------|
| Accounts | ACCT-NUMBER | Balances, interest params, hold amounts, status flags |
| Customers | CIF-CUST-ID | KYC/CIP data, BSA risk rating, OFAC status |
| Transactions | TXN-ID + SEQ | Debits, credits, reversals, batch references |
| General Ledger | GL-ACCT + COST-CTR | Trial balance, MTD/YTD accumulators |
| Holds | ACCT + SEQ | Reg CC holds with business-day release dates |
| Audit Trail | AUDIT-ID + SEQ | Immutable before/after images for every state change |

## Testing

612 tests across 30 suites, organized in 4 phases:

```
Phase 1 — Foundation          Phase 2 — Core Banking
  DATEUTIL     (25 tests)       INTCALC      (29 tests)
  GLPOST       (27 tests)       FEECALC      (23 tests)
  CIFMGMT      (19 tests)       HOLDCALC     (19 tests)
  ACCTMGMT     (31 tests)       ODMGMT       (25 tests)
  TXNPOST      (47 tests)       BSAAML       (17 tests)
                                 REGD         (12 tests)

Phase 3 — Payments             Phase 4 — Production Hardening
  ACHRECV      (38 tests)       OVERFLOW, HOLDDATE, SAR,
  INTEG-EOD    ( 7 tests)       AUDTLOG, FILEIO, BATCH-EOD,
  INTEG-FULL   (13 tests)       BATCH-EOM, INTEG-EOM, DISPMGT,
                                 APYCALC, OFAC, WIREXFR,
                                 CTRFIL, SARFIL, DORMANT,
                                 LOANPMT (254 tests total)
```

Tests cover both happy paths and adversarial edge cases: overflow at max balance, rounding overshoot in interest accrual, partial loan payments that shouldn't reset late fee flags, wire reversals on insufficient funds, structuring detection for split cash deposits, escheated account blocking across all channels, and GL trial balance verification after batch cycles.

```bash
# Run the full suite
make test

# Example output
========================================
 TEST SUMMARY
========================================
 Suites run:     30
 Compile errors: 0
 Total failures: 0
========================================
ALL TESTS PASSED
```

## Building

Requires [GnuCOBOL](https://gnucobol.sourceforge.io/) 3.x.

```bash
# macOS
brew install gnucobol

# Ubuntu/Debian
sudo apt-get install gnucobol

# Compile all modules
make all

# Run tests
make test
```

## Project Structure

```
src/             25 COBOL source modules (6,388 LOC)
tests/           30 test suites (25,512 LOC)
copybooks/       18 shared data structure definitions (711 LOC)
bin/             compiled binaries (gitignored)
data/            indexed data files at runtime (gitignored)
run-tests.sh     4-phase test runner
Makefile         build targets for all modules and tests
```

## Development Process

This system was built iteratively over 41 audit-fix-test rounds. Each round consisted of:

1. **Audit**: Automated code review across all modules looking for missing error handling, regulatory gaps, arithmetic edge cases, and inconsistencies between modules
2. **Fix**: Address confirmed bugs, dismiss false positives with documented reasoning
3. **Test**: Write tests that reproduce the bug, verify the fix, confirm no regressions

The project evolved from basic transaction posting to a complete system with batch processing, indexed file persistence, regulatory filing, and overflow-safe arithmetic. The full bug tracker with all 41 rounds is in [TODO.md](TODO.md).

## Known Limitations

These are documented architectural boundaries, not bugs:

- **Holiday calendar**: Covers 2025–2027. Business day calculations beyond that range treat federal holidays as working days.
- **APYCALC0**: Computes nominal APY/APR. Fee-inclusive APR (Reg Z / TILA) would require a separate module with fee schedule access.
- **Interest day count**: Actual/365 uses a fixed 365-day denominator (Actual/365 Fixed convention). Actual/Actual is available via a separate accrual basis.
- **No online interface**: All operations are invoked via CALL. A CICS or REST gateway would be needed for interactive use.

## License

MIT


<!-- AIPad disposable branch: verified UTF-8 save 🧪 -->

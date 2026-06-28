# ATM System

**Difficulty:** Foundation-Intermediate
**Category:** State Machine + Transaction Safety
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** State (ATM lifecycle), Strategy (transaction types)
**Asked at:** Amazon, Goldman Sachs, JPMorgan, Microsoft, Paytm

---

## Problem Statement

Design an ATM system. A user inserts their card, enters a PIN, and can perform transactions: check balance, withdraw cash, deposit, or transfer funds. The ATM must enforce state — a user cannot withdraw without first authenticating, and the machine must be in a specific state to dispense cash.

The system must handle insufficient funds, incorrect PINs (lockout after 3 attempts), and concurrent access to account balances.

---

## Actors

| Actor | Description |
|-------|-------------|
| Customer | Inserts card, authenticates, performs transactions |
| Bank System | Validates PINs, holds account balances, authorizes transactions |
| ATM Machine | Manages state, card reader, cash dispenser, receipt printer |
| ATM Technician | Restocks cash (out of scope for MVP) |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Customer inserts card — ATM reads card number
- FR2: Customer enters PIN — validated against bank system
- FR3: After 3 failed PIN attempts, card is locked (account blocked)
- FR4: Authenticated customer can check balance
- FR5: Authenticated customer can withdraw cash (if sufficient balance + ATM has cash)
- FR6: ATM dispenses cash and prints receipt on successful withdrawal
- FR7: Customer ejects card and session ends

**Secondary (implement if time allows):**
- FR8: Deposit cash/cheque
- FR9: Transfer funds to another account
- FR10: Change PIN
- FR11: ATM tracks available cash (can reject withdrawal if insufficient cash in machine)

---

## Non-Functional Requirements

- **Concurrency:** Multiple ATMs can access the same account simultaneously (same bank customer at two ATMs)
- **Atomicity:** Withdrawal must be atomic — debit account AND dispense cash together, or neither
- **Security:** Card locked after 3 wrong PINs
- **Persistence:** Account balances persist (assume Bank system handles this)

---

## Constraints and Assumptions

- ATM has limited cash (track available denominations)
- PIN is 4 digits; validated against bank backend
- Only one active session per card at a time (bank enforces this)
- Out of scope: network failure handling, card chip/NFC (assume magnetic stripe), multi-currency

---

## Good Clarifying Questions to Ask

1. What transaction types are needed — withdraw only, or also deposit/transfer?
2. How many PIN attempts before lockout?
3. Does the ATM track its own cash inventory, or is that external?
4. What happens if the network to the bank drops mid-transaction?
5. Can two people use the same account at two different ATMs simultaneously?
6. Is the PIN stored in the ATM or always validated against bank?
7. What denominations does the ATM dispense?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **ATM** — context; holds current state, cash inventory, card reader, screen
- **ATMState** — interface; Idle, CardInserted, PinVerified, TransactionInProgress
- **IdleState** — waits for card insertion
- **CardInsertedState** — prompts PIN; tracks failed attempts; locks on 3rd failure
- **PinVerifiedState** — shows menu; customer selects transaction type
- **TransactionState** — executes chosen transaction; returns to PinVerified on completion
- **Card** — card number, associated account id
- **Account** — balance, PIN (hashed), locked status; lives in BankSystem
- **BankSystem** — validates PIN, authorizes withdrawal, holds accounts
- **CashDispenser** — tracks available denominations; dispenses exact amount
- **Transaction** — interface; WithdrawTransaction, BalanceInquiry, DepositTransaction

</details>

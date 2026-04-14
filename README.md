# lean-refine

> [!IMPORTANT]
> This repository is superseded by [lean-agent](https://github.com/savarin/lean-agent). Please use that instead.

Autonomous code hardening via IES (Invariant Enforcement Score) optimization.

Wraps Claude to analyze a repository's invariants, build an evaluation harness,
and iteratively improve enforcement — from unguarded (0) through convention (1)
and validated (2) to structural (3).

Demo repo: [savarin/ledger](https://github.com/savarin/ledger) — see the [Result](#result) section below.

## Install

### As a Claude Code plugin (recommended)

```
/plugin marketplace add savarin/lean-refine
/plugin install lean-refine@lean-refine
```

Then invoke in any repo:

```
/lean-refine:refine                     # default: Lean formalization + optimization
/lean-refine:refine --direct            # skip formalization, code reading only
/lean-refine:refine --tag my-hardening  # custom branch name
```

### As a standalone CLI

```bash
uv tool install .
```

```bash
refine <repo>                          # default: Lean formalization + optimization (requires Lean 4)
refine <repo> --direct                 # skip formalization, code reading only
refine <repo> --model sonnet           # use a different model
refine <repo> --tag my-hardening       # branch: lean-refine/my-hardening
refine <repo> --skip-permissions       # autonomous operation (no permission prompts)
refine <repo> --max-turns 300          # increase turn limit (default: 200)
```

## What it does

1. **Analyzes** the repo — identifies 5-15 critical invariants
2. **Builds** `prepare.py` — an evaluation harness that scores each invariant
3. **Executes** known improvements — pre-planned targets, sequentially
4. **Explores** further improvements — AUTORESEARCH discovery loop
5. **Stops** when ceiling is reached or all invariants are covered

Output lives in `<repo>/.lean-refine/` on a `lean-refine/<tag>` branch.

## Example

[`ledger.py`](https://github.com/savarin/ledger) — a 13-line double-entry ledger. `refine ledger-repo` discovers five invariants the code assumes but doesn't enforce:

1. Balances must be numeric (passing a string silently corrupts arithmetic)
2. Transfer amounts must be positive (negative amounts reverse direction silently)
3. Accounts can't be duplicated (`create_account` silently overwrites existing balances)
4. Transfers require both accounts to exist (missing target debits source then crashes, losing money)
5. Self-transfers are meaningless (works by accident, breaks on any refactor)

Lean formalization surfaces these by forcing explicit type declarations (e.g., defining `Balance` with `val : Nat` makes negativity structurally impossible, revealing that the Python code allows it silently).

## Result

IES: 0.0476 → 1.0000 (perfect score) in 12 iterations, 0 discards.

| Iteration | IES | Change | Description | Code |
|-----------|-----|--------|-------------|------|
| 0 | 0.05 | — | Baseline: 6 unguarded, 1 convention | — |
| 1 | 0.24 | +0.19 | Account existence check in `transfer` (also caught atomicity) | [L52-53](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L52-L53) |
| 2 | 0.33 | +0.10 | Account existence check in `balance` | [L57-58](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L57-L58) |
| 3 | 0.43 | +0.10 | Transfer amount positivity check | [L16-17](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L16-L17) |
| 4 | 0.48 | +0.05 | Atomic dict rebuild for transfer | [L54](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L54) |
| 5 | 0.57 | +0.10 | Account overwrite protection | [L45-46](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L45-L46) |
| 6 | 0.67 | +0.10 | Self-transfer prevention | [L30-31](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L30-L31) |
| 7 | 0.71 | +0.05 | Runtime isinstance type checks | [L5-6](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L5-L6), [L14-15](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L14-L15) |
| 8 | 0.76 | +0.05 | `Balance` class (structural type safety) | [L1-7](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L1-L7) |
| 9 | 0.81 | +0.05 | `PositiveAmount` class (structural positivity) | [L10-18](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L10-L18) |
| 10 | 0.86 | +0.05 | `setdefault` (structural overwrite protection) | [L47](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L47) |
| 11 | 0.90 | +0.05 | `TransferRequest` class (structural self-transfer) | [L26-35](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L26-L35) |
| 12 | 1.00 | +0.10 | `AccountRef` type (structural account existence) | [L21-23](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L21-L23), [L50](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L50), [L56](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py#L56) |

All 7 invariants structural. 20 tests passing. Artifacts in [`.lean-refine/`](https://github.com/savarin/ledger/tree/lean-refine/lean/.lean-refine) with Lean formalization, frozen harness, results TSV, and iteration log.

## Before

[`src/ledger.py`](https://github.com/savarin/ledger/blob/67d6e236296e4787e8924eed860910475d37c138/src/ledger.py)

```python
class Ledger:
    def __init__(self):
        self.accounts = {}

    def create_account(self, account_id, opening_balance):
        self.accounts[account_id] = opening_balance

    def transfer(self, from_id, to_id, amount):
        self.accounts[from_id] -= amount
        self.accounts[to_id] += amount

    def balance(self, account_id):
        return self.accounts[account_id]
```

## After

[`src/ledger.py`](https://github.com/savarin/ledger/blob/506584542d1c1c2751e2d5819a5e7e29524de924/src/ledger.py)

```python
class Balance:
    __slots__ = ("value",)

    def __init__(self, value: int | float) -> None:
        if not isinstance(value, (int, float)):
            raise TypeError("value must be numeric")
        self.value = value


class PositiveAmount:
    __slots__ = ("value",)

    def __init__(self, value: int | float) -> None:
        if not isinstance(value, (int, float)):
            raise TypeError("amount must be numeric")
        if value <= 0:
            raise ValueError("amount must be positive")
        self.value = value


class AccountRef(str):
    """Verified account identifier obtained from create_account."""
    pass


class TransferRequest:
    __slots__ = ("from_id", "to_id", "amount")

    def __init__(self, from_id: str, to_id: str, amount: int | float) -> None:
        if from_id == to_id:
            raise ValueError("cannot transfer to same account")
        PositiveAmount(amount)
        self.from_id = from_id
        self.to_id = to_id
        self.amount = amount


class Ledger:
    def __init__(self):
        self.accounts = {}

    def create_account(self, account_id: str, opening_balance: int | float) -> AccountRef:
        Balance(opening_balance)
        ref = AccountRef(account_id)
        if ref in self.accounts:
            raise ValueError("account already exists")
        self.accounts.setdefault(ref, opening_balance)
        return ref

    def transfer(self, from_id: AccountRef, to_id: AccountRef, amount: int | float) -> None:
        request = TransferRequest(from_id, to_id, amount)
        if request.from_id not in self.accounts or request.to_id not in self.accounts:
            raise ValueError("account does not exist")
        self.accounts = {**self.accounts, request.from_id: self.accounts[request.from_id] - request.amount, request.to_id: self.accounts[request.to_id] + request.amount}

    def balance(self, account_id: AccountRef):
        if account_id not in self.accounts:
            raise ValueError("account does not exist")
        return self.accounts[account_id]
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI (`claude` on PATH)
- Python >= 3.10
- [Lean 4](https://leanprover.github.io/lean4/doc/setup.html) (`lean` and `lake` on PATH) — required for default mode, not needed with `--direct`

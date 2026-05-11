# DSCEngine Security Audit

**Protocol:** Multi-Collateral Stability Engine (DSCEngine)  
**Auditor:** CableGraph — White hats for Ethereum  
**Date:** May 10, 2026  
**Commit Audited:** `[e7231d9e86d18e4704f88b86cf84c01da9b2bbd6]`  
**Status:** Initial Report — Awaiting Client Response

---

## Summary

| Severity | Count | Status |
|---|---|---|
| 🔴 Critical | 2 | Open |
| 🟠 High | 4 | Open |
| 🟡 Medium | 3 | Open |
| 🟢 Low | 2 | Open |
| ℹ️ Informational | 1 | Open |
| **Total** | **12** | |

---

## Key Findings

| ID | Severity | Title |
|---|---|---|
| C-01 | Critical | Ownable constructor never initializes owner — protocol permanently DOA |
| C-02 | Critical | `emergencyWithdraw` drains all collateral with zero accounting update |
| H-01 | High | `_calculateHealthFactor` arguments swapped — all HF values inverted |
| H-02 | High | `depositCollateralAndMintDSC` permanently broken — double nonReentrant |
| H-03 | High | `redeemCollateral` and `burnDSC` missing `notPaused` modifier |
| H-04 | High | Liquidation collateral sent before DSC pulled — token flow ordering risk |
| M-01 | Medium | `LIQUIDATION_THRESHOLD = 150` — comment falsely claims 200% |
| M-02 | Medium | Oracle: missing `price > 0` and `answeredInRound` checks |
| M-03 | Medium | `ERC20YulLib` — no return value validation on low-level calls |

---

## Audit Report

📄 [DSCEngine-Audit-Report.md](./DSCEngine-Audit-Report.md)

To generate the PDF:

```bash
# Standard
pandoc DSCEngine-Audit-Report.md \
  -o DSCEngine-Audit-Report.pdf \
  --pdf-engine=xelatex \
  --toc \
  --highlight-style=tango \
  -V mainfont="DejaVu Sans" \
  -V monofont="DejaVu Sans Mono" \
  -V fontsize=11pt \
  -V geometry:margin=1in

# Blue theme (requires eisvogel template)
pandoc DSCEngine-Audit-Report.md \
  -o DSCEngine-Audit-Report.pdf \
  --pdf-engine=xelatex \
  --template eisvogel \
  --toc \
  -V titlepage=true \
  -V titlepage-color="1E40AF" \
  -V titlepage-text-color="FFFFFF" \
  -V titlepage-rule-color="FFFFFF" \
  --highlight-style=tango
```

> Install eisvogel: download from [github.com/Wandmalfarbe/pandoc-latex-template](https://github.com/Wandmalfarbe/pandoc-latex-template/releases) and place `eisvogel.latex` in `~/.local/share/pandoc/templates/`

---

## Test Suite

```
test/
├── unit/
│   ├── DSCEngineTest.t.sol                  ← existing baseline tests
│   ├── DecentralizedStableCoinTest.t.sol    ← C-01 PoC
│   ├── DSCEnginePoC.t.sol                   ← C-02, H-01..H-04, M-01..M-03
│   └── DSCEngineMath.t.sol                  ← Pass 2 math verification
├── fuzz/
│   ├── Invariants.t.sol                     ← existing invariant suite
│   ├── Handler.t.sol                        ← existing handler
│   ├── InvariantsExpanded.t.sol             ← solvency + HF invariants
│   └── HandlerExpanded.t.sol                ← price crash + liquidation + burn
└── Mocks/
    ├── ERC20Mock.sol
    ├── MockV3Aggregator.sol
    └── MockFailERC20.sol                    ← silent-fail token for M-03
```

### Run Commands

```bash
# 1. Baseline unit tests
forge test --match-path "test/unit/DSCEngineTest.t.sol" -vvvv

# 2. C-01 PoC — Ownable constructor
forge test --match-path "test/unit/DecentralizedStableCoinTest.t.sol" -vvvv

# 3. All Critical + High PoCs
forge test --match-path "test/unit/DSCEnginePoC.t.sol" -vvvv

# 4. Math pass tests
forge test --match-path "test/unit/DSCEngineMath.t.sol" -vvvv

# 5. Original invariant suite
forge test --match-path "test/fuzz/Invariants.t.sol" \
  --invariant-runs 1000 --invariant-depth 50 -vvvv

# 6. Expanded invariant suite (price crash + HF)
forge test --match-path "test/fuzz/InvariantsExpanded.t.sol" \
  --invariant-runs 1000 --invariant-depth 100 -vvvv

# 7. Full suite
forge test --match-path "test/**/*.t.sol" -vvvv

# 8. Coverage
forge coverage --report lcov
genhtml lcov.info --output-directory coverage/

# 9. Gas snapshot
forge snapshot --match-path "test/unit/DSCEnginePoC.t.sol"
```

### Finding → Test Mapping

| Finding | Test Function | Expected |
|---|---|---|
| C-01 | `test_C01_OwnableConstructorNeverSetsOwner` | PASS — bug confirmed |
| C-01 | `test_C01_MintRevertsForEveryCallerBecauseOwnerIsZero` | PASS — bug confirmed |
| C-02 | `test_C02_EmergencyWithdrawDrainsWithoutAccountingUpdate` | PASS — bug confirmed |
| H-01 | `test_H01_ArgumentSwapProducesInvertedHealthFactor` | **FAIL** — failure IS the PoC |
| H-01 | `test_H01_StateLevelHealthFactorIsInverted` | **FAIL** — failure IS the PoC |
| H-02 | `test_H02_DepositCollateralAndMintDSCAlwaysReverts` | PASS — bug confirmed |
| H-03 | `test_H03_RedeemCollateralSucceedsDuringPause` | PASS — bug confirmed |
| H-03 | `test_H03_BurnDSCSucceedsDuringPause` | PASS — bug confirmed |
| H-04 | `test_H04_LiquidationTokenFlowOrdering` | Documents behavior |
| M-01 | `test_M01_CollateralisationRatioIs150Not200` | PASS — bug confirmed |
| M-02 | `test_M02_ZeroOraclePricePassesStaleCheck` | PASS — bug confirmed |

> **H-01 note:** The test asserts `actualHF == 6e18`. When H-01 is present, Foundry prints the actual value (`0.375e18`). The printed trace is your machine-generated PoC — paste it directly into the report.

---

## Methodology

This audit applied a structured **5-pass manual review** before any automated tooling:

| Pass | Focus | Key Finding |
|---|---|---|
| Pass 1 — Trust model | State variables, privileges, ownership | C-02, L-01 |
| Pass 2 — Arithmetic | Health factor math, boundaries, units | H-01, M-01 |
| Pass 3 — External calls | Token flows, oracle, CEI order | H-04, M-02, M-03 |
| Pass 4 — Composability | Internal call chains, modifier stacking | H-02, H-03 |
| Pass 5 — Liquidation | End-to-end liquidation trace | H-01, H-04 |

**Tools used:**

| Tool | Purpose | Findings confirmed |
|---|---|---|
| Slither | Static analysis, call-graph | H-02, H-03, M-03 |
| Mythril | Symbolic execution | C-02, M-02 |
| Foundry fuzz | Property testing | H-01 (inverted HF) |
| Foundry invariant | Stateful sequence testing | Solvency invariant |

**Slither did not detect:** C-01, C-02, H-01, H-04, M-01, M-02 — all required manual review. This confirms: tools confirm patterns; humans discover logic errors.

---

## Scope

**Files audited:**

```
src/Core/DSCEngine.sol
src/Core/DecentralizedStableCoin.sol
src/Libraries/EngineMath.sol
src/Libraries/OracleLib.sol
src/Libraries/ERC20YulLib.sol
src/Libraries/AccountDataPackerLib.sol
```

**Out of scope:** Deployment scripts, HelperConfig, test mocks, frontend.

---

## About the Auditor

Cablegraph provides smart contract auditing and offensive security research for Ethereum ecosystems. We secure DeFi protocols, token contracts, vaults, governance systems, and oracle integrations through rigorous code review and vulnerability analysis.
White hats for Ethereum.

**Methodology stack:** Manual multi-pass review · Slither · Mythril · Foundry fuzz + invariant testing · Echidna

**Contact:** Open to audit engagements and research collaborations.  
**Portfolio:** github.com/denniskiptoo · [Your Twitter/X handle]

---

## Disclaimer

This audit makes every effort to identify as many vulnerabilities as possible within the given time and scope. It holds no responsibility for the findings herein. A security audit does not endorse the underlying business or product. This report does not constitute financial or legal advice.

---

*DSCEngine Security Audit — Dennis Kiptoo — May 2026*

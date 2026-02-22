# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Dart/Flutter monorepo for ERC-4337 (Account Abstraction) smart accounts, ported from [permissionless.js](https://github.com/pimlicolabs/permissionless.js). Two published packages:

- **`permissionless`** — Core ERC-4337 SDK: 9 smart account types, bundler/paymaster clients, ERC-7579 actions
- **`permissionless_passkeys`** — WebAuthn/Passkeys extension for biometric authentication with P256 signatures

Published to [pub.dev](https://pub.dev/packages/permissionless).

## Monorepo Setup

Uses Dart pub workspaces (defined in root `pubspec.yaml`) with Melos scripts for cross-package operations. Melos is a dev dependency — no global install required.

```bash
# Install all dependencies (resolves workspace)
dart pub get

# Bootstrap (links local packages via pubspec_overrides.yaml)
dart run melos bootstrap

# Cross-package commands
dart run melos test          # Run tests in all packages
dart run melos analyze       # Static analysis across all packages
dart run melos format        # Format all Dart files
dart run melos clean         # Clean build artifacts
```

## Package-Specific Commands

### permissionless (pure Dart)
```bash
cd packages/permissionless
dart test                                          # All tests
dart test -P quick                                 # Exclude integration/funded tests
dart test test/accounts/safe/safe_account_test.dart # Single test file
dart test -t integration                           # Integration tests only (needs network)
dart test -P ci                                    # CI preset (expanded reporter, 3x timeout)
dart analyze
```

### permissionless_passkeys (Flutter)
```bash
cd packages/permissionless_passkeys
flutter test                        # All tests
flutter test test/encoding/         # Test directory
flutter analyze
```

### Environment Variables (Integration Tests)
```bash
PIMLICO_API_KEY=your_key
TEST_PRIVATE_KEY=0x...
FUNDED_ACCOUNT_ADDRESS=0x...
```

## Changeset & Release Workflow

Custom changeset tooling in `tool/` (no external dependency):

```bash
make changeset        # Interactive: select packages, bump type, write .changesets/*.md
make version          # Apply changesets: bump pubspec.yaml versions, write _current_release.json
make changelog        # Generate CHANGELOG.md entries from _current_release.json
make prepare-release  # version + changelog in one step (does not publish)
```

Changesets live in `.changesets/*.md` with YAML frontmatter (title, date, packages with bump types) and a markdown body.

## Architecture

### Core Abstractions (permissionless package)

**`SmartAccount`** (`clients/smart_account/smart_account_interface.dart`) — Base interface all 9 account types implement:
- `getAddress()` — Deterministic CREATE2 address (works pre-deployment)
- `getFactoryData()` — Factory address + init data for deployment
- `encodeCall()` / `encodeCalls()` — ABI-encode execution calldata
- `getStubSignature()` — Dummy signature for gas estimation
- `signUserOperation()` — Sign with account owner(s)

**`AccountOwner`** (`accounts/account_owner.dart`) — Unified signing interface with three modes:
- `signPersonalMessage()` — EIP-191 prefixed (Simple, Nexus, Light, Trust, Thirdweb, Etherspot)
- `signRawHash()` — Direct hash signing (Kernel)
- `signTypedData()` — EIP-712 typed data (Safe)
- Has `type` field (`'local'` for ECDSA, `'webAuthn'` for passkeys) used by accounts to select signature encoding

**`SmartAccountClient`** — Orchestrates account + bundler + paymaster + public client for the full UserOperation lifecycle: prepare → sign → send → wait.

### Account Implementation Pattern

Each account in `lib/src/accounts/<name>/`:
- `<name>_account.dart` — Implements `SmartAccount`, handles version/EntryPoint dispatch
- `constants.dart` — Per-version, per-chain contract addresses (factory, singleton, modules)
- `<name>.dart` — Barrel file with public exports

### EntryPoint Versions

v0.6 and v0.7 have incompatible UserOperation formats. Each account declares which versions it supports. Key differences:
- v0.6: `initCode` field, separate `callGasLimit`/`verificationGasLimit`/`preVerificationGas`
- v0.7: `factory`+`factoryData` fields, packed gas fields, different signature encoding

### Client Layer

Six client types in `lib/src/clients/`, each wrapping JSON-RPC calls:
- `BundlerClient` — Standard ERC-4337 bundler RPCs
- `PaymasterClient` — `pm_getPaymasterStubData` / `pm_getPaymasterData`
- `PublicClient` — `eth_call`, `eth_getCode`, `eth_chainId`, EntryPoint nonce
- `PimlicoClient` — Pimlico-specific: gas prices, operation status, token quotes
- `EtherspotClient` — Etherspot/Skandha bundler extensions
- `SmartAccountClient` — High-level orchestration (not an RPC client)

### Passkeys Package Architecture

Extends the core package with WebAuthn support:
- `WebAuthnCredential` — Wraps P256 public key coordinates (x, y) + credential ID
- `WebAuthnAccount` — Implements `AccountOwner` with `type: 'webAuthn'`
- Kernel and Safe use different signature encoding formats (see `encoding/`)
- RIP-7212 P256 precompile detection: dynamic (`isRip7212Supported()` via `eth_call`) or static (curated chain ID list)

## Critical Implementation Details

- **BigInt everywhere** — All uint256 values use Dart `BigInt`, never `int`
- **Hex strings** — Always `0x`-prefixed
- **Signatures** — 65 bytes exactly: r(32) + s(32) + v(1)
- **Safe signatures** — V value adjusted +31 for `eth_sign` mode
- **ABI encoding** — Must match Solidity encoder exactly (use `web3dart`'s encoder)
- **CREATE2 addresses** — Must match on-chain proxy factory calculation exactly
- **Do NOT use `dart:mirrors`** — Breaks tree shaking

## Lint Configuration

Strict analysis enabled (`analysis_options.yaml`):
- `strict-casts: true`, `strict-inference: true`, `strict-raw-types: true`
- Notable enforced rules: `require_trailing_commas`, `prefer_single_quotes`, `prefer_const_constructors`, `unawaited_futures`, `always_declare_return_types`
- Uses `package:lints/recommended.yaml` as base

## Test Tags

Configured in `dart_test.yaml`:
- `@Tags(['integration'])` — Requires network access
- `@Tags(['funded'])` — Requires funded test accounts with real ETH
- Preset `-P quick` excludes both tags for fast local iteration

## TypeScript Reference

When implementing features, reference the TypeScript source at the sibling `../permissionless.js/` directory (if present in the parent workspace). The Dart API mirrors the TypeScript structure closely.

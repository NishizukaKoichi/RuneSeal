# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial RuneSeal implementation for automated PR evaluation
- GitHub API integration via octocrab
- Policy-based evaluation system with security gates
- Quality checks and metrics evaluation
- Auto-merge capability with configurable strategies
- CLI commands: evaluate-merge and audit
- YAML configuration for policies and preferences
- Comprehensive CI/CD pipeline with quality gates
- SBOM generation and signing with cosign
- Multi-platform build support (Linux, macOS, Windows, WASM)

### Security
- Dependency vulnerability scanning with cargo-audit
- License compliance checking with cargo-deny
- Keyless SBOM signing for supply chain security

## [0.1.0] - 2025-08-22

### Added
- Initial project setup
- Basic Rust/WASM project structure
- Core domain models and error handling
- Integration and E2E test frameworks
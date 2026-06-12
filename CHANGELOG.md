# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.1] - 2026-06-12

### Added

- GitHub Actions CI workflow for type checking, linting, and tests.
- Husky pre-commit hook with lint-staged.
- `CHANGELOG.md` and `.editorconfig`.
- Separate `tsconfig.build.json` so test files are excluded from published type declarations.

### Changed

- Cross-platform npm scripts for Windows, macOS, and Linux.
- Improved README documentation and npm publish metadata.
- Pinned `@types/node` and enabled `skipLibCheck` for stable TypeScript builds.

## [1.0.0] - 2020

### Added

- Initial release of `schema-checker` (formerly Funval).
- Runtime validation types for TypeScript with schema composition.
- Support for synchronous and asynchronous validators.
- Built-in types: `string`, `number`, `boolean`, `array`, `object`, `unknown`, and `DateType`.

[1.0.1]: https://github.com/deepthought26/schema-checker/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/deepthought26/schema-checker/releases/tag/v1.0.0

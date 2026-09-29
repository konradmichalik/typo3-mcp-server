# AGENTS.md

## Project overview

TYPO3 extension `mcp_server` (Composer package `hn/typo3-mcp-server`). It provides a Model Context Protocol (MCP) server so that people without technical knowledge can edit TYPO3 pages and content using LLMs.

- PHP: `>=8.2` (CI tests 8.2 to 8.5)
- TYPO3: `^13.4 || ^14.3` (`cms-core`, `cms-backend`, `cms-workspaces`)
- MCP SDK: `logiscape/mcp-sdk-php` `^2.0`

Read `TECHNICAL_OVERVIEW.md` and `Documentation/Architecture/` (`WorkspaceTransparency.md`, `LanguageOverlays.md`, `InlineRelations.md`) for design decisions and implementation details.

## Structure

- `Classes/MCP/` server factory and tool registry
  - `Tool/` page tools (`GetPageTool`, `GetPageTreeTool`, `SearchTool`, `AbstractTool`, `ToolInterface`)
  - `Tool/Record/` table tools (`ReadTableTool`, `WriteTableTool`, `ListTablesTool`, `GetTableSchemaTool`, `GetFlexFormSchemaTool`)
  - `Tool/File/` file tools (`UploadFileTool`)
- `Classes/Service/` workspace, language, table access, OAuth, file upload and site information services
- `Classes/Command/` console commands (`McpServerCommand`, `McpTestCommand`, `OAuthManageCommand`)
- `Classes/Controller/`, `Classes/Http/`, `Classes/Middleware/` HTTP endpoint and OAuth handling
- `Classes/Database/`, `Classes/Event/`, `Classes/EventListener/`, `Classes/Exception/`, `Classes/Traits/`, `Classes/Updates/`, `Classes/Utility/`
- `Configuration/`, `Resources/` extension configuration and assets
- `Tests/Functional/` functional tests (TYPO3 testing framework)
- `Tests/Unit/` unit tests
- `Tests/Llm/` tests that let real LLMs use the tools
- `Build/` test runner, setup scripts and Playwright E2E tests (`Build/tests/playwright/`)

## Development commands

- `composer install` also runs `Build/setup-typo3.sh` (`composer setup`)
- `composer test` runs the functional suite with paratest
- `composer test:llm` runs the LLM tests (needs `OPENROUTER_API_KEY`, see `Tests/Llm/README.md`)
- `composer build:ter` builds the TER package
- `Build/runTests.sh` runs suites in Docker with a configurable PHP version and database: `-p <8.2|8.3|8.4|8.5>`, `-d <sqlite|mysql|mariadb|postgres>`, `-s <functional|lint|e2e>`
- `Build/runTests.sh -s lint` runs `php -l` over `Classes` and `Tests`

## Testing

- Tests must cover the TYPO3 integration, so use functional tests
- E2E (Playwright): `Build/runTests.sh -s e2e`. It starts MySQL, TYPO3 and Playwright in Docker. Without Docker it falls back to local mode (host PHP, SQLite, local Playwright), which `-n` or `--no-docker` forces
- Check every MCP tool call for failure in tests: `$this->assertFalse($result->isError, json_encode($result->jsonSerialize()));`
- CI (`tests.yml`) runs the functional suite for TYPO3 13.4 and 14.3 on PHP 8.2 to 8.5 without coverage, plus the E2E suite on PHP 8.4 with `--no-docker`

## Code style and static analysis

No PHP CS Fixer, PHPStan, Rector or ECS configuration exists. The only automated check is `php -l` via `Build/runTests.sh -s lint`. Follow the style of the surrounding code.

## Conventions

- The MCP tools for LLMs need no backwards compatibility. Parameters and names may change freely if that describes them better
- Every tool that uses the database must use TYPO3 Workspaces explicitly. Live data must never be edited directly
- Workspaces must stay invisible to the MCP client, for example by exposing only the live id
- The extension creates a workspace if none exists. This is intentional and must work in tests too. Mixed live and workspace data is a real scenario, and tool results must behave as if there were no workspaces
- Use TYPO3 core APIs for TCA related data actions whenever one exists
- If the TYPO3 instance has no language support, hide all translation aspects such as parameters and database fields (check `LanguageService`)
- Language overlays use TYPO3's `PageRepository` API, workspace overlays use a custom implementation for transparency (see `Documentation/Architecture/LanguageOverlays.md`)

## Git workflow

- Commit format: `<type>: <description>` with type `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf` or `ci`
- One commit per logical change, single line message
- No co-author trailers
- Never skip hooks (`--no-verify`)

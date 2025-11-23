# CLAUDE.md - AI Assistant Guide for Harambe

## Project Overview

**Harambe** is a web extraction SDK developed by Reworkd AI. It provides a unified interface for building web scrapers using Playwright, with built-in data validation, pagination handling, and observer patterns for extensibility.

### Purpose
- Provide a standardized framework for web scraping/extraction
- Enable both manual and auto-generated web extractors
- Handle common scraping challenges: pagination, downloads, captchas, data validation

### Target Users
- Developers building web scrapers
- Reworkd's internal code generation system
- Teams needing structured data extraction with schema validation

### Problem Solved
Simplifies web scraping by providing:
- Automatic URL normalization and deduplication
- Schema-based data validation using Pydantic
- Observer pattern for flexible output handling
- Built-in pagination and download capture utilities

## Tech Stack

### Core Package (`harambe-core`)
| Dependency | Version | Purpose |
|------------|---------|---------|
| pydantic | 2.9.2 | Data validation and schema parsing |
| dateparser | 1.2.0 | Flexible date/time parsing |
| email-validator | 2.2.0 | Email format validation |
| phonenumbers | 8.13.47 | Phone number parsing/validation |
| python-slugify | 8.0.4 | String slug generation |
| price-parser | >=0.4.0 | Price extraction from strings |

### SDK Package (`harambe-sdk`)
| Dependency | Version | Purpose |
|------------|---------|---------|
| harambe-core | 0.77.5 | Core validation/types |
| playwright | 1.47.0 | Browser automation |
| beautifulsoup4 | 4.12.3 | HTML parsing |
| requests | 2.32.3 | HTTP requests |
| playwright-stealth | 1.0.6 | Anti-bot detection evasion |
| aiohttp | 3.10.10 | Async HTTP client |
| curl-cffi | 0.7.3 | HTTP client with TLS fingerprinting |
| ua-generator | 1.0.5 | User agent generation |
| markdownify | 0.14.1 | HTML to Markdown conversion |
| setuptools | >=73.0.0 | Package utilities |
| wrapt | >=1.17.2 | Function decorators |

### Dev Dependencies
- **ruff** 0.11.5 - Linting and formatting
- **mypy** 1.11.1 - Static type checking
- **pytest** 7.4.4/8.3.4 - Testing framework
- **pytest-cov** 4.1.0 - Coverage reporting
- **pytest-asyncio** 0.21.2 - Async test support

### Runtime Requirements
- Python 3.11+
- uv (package manager)
- Playwright browsers (chromium/firefox/webkit)

## Project Structure

```
harambe/
├── core/                          # harambe-core package
│   ├── harambe_core/
│   │   ├── __init__.py           # Exports Schema, SchemaParser
│   │   ├── errors.py             # Custom exception classes
│   │   ├── normalize_url.py      # URL normalization utilities
│   │   ├── types.py              # Type definitions (Schema, Cookie, etc.)
│   │   ├── observer/             # Observer pattern implementations
│   │   │   ├── base.py          # OutputObserver protocol
│   │   │   ├── json_observer.py # JSON output observer
│   │   │   ├── memory_observer.py
│   │   │   ├── logging_observer.py
│   │   │   ├── storage_observer.py
│   │   │   └── serialization_observer.py
│   │   └── parser/               # Schema parsing system
│   │       ├── parser.py        # Main SchemaParser class
│   │       ├── constants.py
│   │       ├── type_*.py        # Type-specific parsers
│   │       └── expression/      # Expression evaluation
│   └── test/                     # Core package tests
│
├── sdk/                          # harambe-sdk package
│   ├── harambe/
│   │   ├── __init__.py          # Main SDK exports
│   │   ├── core.py              # SDK class (main interface)
│   │   ├── types.py             # SDK-specific types
│   │   ├── cache.py             # Caching utilities
│   │   ├── handlers.py          # Request handlers
│   │   ├── pagination.py        # Deduplication handler
│   │   ├── proxy.py             # Proxy URL parsing
│   │   ├── tracker.py           # File data tracking
│   │   ├── user_agent.py        # UA generation
│   │   ├── utils.py             # Playwright utilities
│   │   ├── cookie_utils.py
│   │   ├── instrumentation.py
│   │   ├── meta.py
│   │   ├── contrib/             # Browser harness implementations
│   │   │   ├── playwright/      # Playwright harness
│   │   │   └── soup/            # BeautifulSoup harness
│   │   └── html_converter/      # HTML to text/markdown
│   └── test/                    # SDK tests
│
├── .github/
│   └── workflows/
│       └── python.yml           # CI/CD pipeline
├── schema.json                   # JSON schema definition
├── check.sh                      # Test runner script
└── README.md
```

## Architecture

### Data Flow
```
User Scraper → SDK.run() → Browser Harness → Page Navigation
                              ↓
                        Scrape Function
                              ↓
                    SDK.save_data() / SDK.enqueue()
                              ↓
                      SchemaParser.validate()
                              ↓
                    Observer.on_save_data()
```

### Key Abstractions

1. **SDK Class** (`sdk/harambe/core.py`)
   - Main interface for scrapers
   - Manages page, observers, validation
   - Provides: `save_data`, `enqueue`, `paginate`, `capture_*`

2. **SchemaParser** (`core/harambe_core/parser/parser.py`)
   - Converts JSON schema to Pydantic models
   - Validates and transforms scraped data
   - Handles type coercion (dates, phones, emails, URLs)

3. **OutputObserver Protocol** (`core/harambe_core/observer/base.py`)
   - Observer pattern for scraper outputs
   - Methods: `on_save_data`, `on_queue_url`, `on_download`, etc.
   - Implementations: LoggingObserver, JSONObserver, LocalStorageObserver

4. **WebHarness** (`sdk/harambe/contrib/`)
   - Factory pattern for browser instances
   - Implementations: playwright_harness, soup_harness
   - Handles stealth, proxy, cookies, headers

### Design Patterns
- **Observer Pattern**: Decouple data handling from scraping logic
- **Factory Pattern**: WebHarness for browser creation
- **Decorator Pattern**: `@SDK.scraper()` and `@SDK.with_headers()`
- **Protocol Pattern**: Type-safe interfaces via Python protocols

## Common Commands

```bash
# Install dependencies
cd sdk && uv sync
uv run playwright install chromium --with-deps

# Run all tests
./check.sh

# Run specific package tests
cd core && uv run pytest -vv
cd sdk && uv run pytest -vv

# Run with coverage
uv run pytest -vv --cov=harambe .

# Format code
uv run ruff format .

# Type checking
uv run mypy harambe
```

## Dependency Issues & Solutions

### Critical Issues

1. **curl-cffi Platform Limitations**
   - **Issue**: curl-cffi 0.7.3 has limited platform support and may fail on ARM/M1 Macs or certain Linux distros
   - **Solution**: Install with `pip install curl-cffi --no-binary curl-cffi` or use pre-built wheels from their releases page

2. **pytest-asyncio Version Mismatch**
   - **Issue**: Core uses pytest 7.4.4, SDK uses pytest 8.3.4 - can cause inconsistent behavior
   - **Solution**: Align both packages to pytest 8.x and pytest-asyncio 0.23+

3. **playwright-stealth Maintenance**
   - **Issue**: playwright-stealth 1.0.6 is not actively maintained; stealth techniques become outdated
   - **Solution**: Consider forking or using undetected-playwright as alternative

### Medium Issues

4. **setuptools Requirement**
   - **Issue**: `setuptools>=73.0.0` is unusual and may conflict with system packages
   - **Solution**: Remove if not strictly needed; use importlib.metadata instead of pkg_resources

5. **Pydantic Version Lock**
   - **Issue**: Pydantic 2.9.2 is pinned; may miss security patches
   - **Solution**: Use `pydantic>=2.9,<3.0` for flexibility

### Platform-Specific Issues

6. **Playwright Browser Installation**
   - **Issue**: Requires system dependencies on Linux
   - **Solution**: Run `playwright install-deps` before `playwright install chromium`

## Code Quality Issues

### Security Considerations

1. **Proxy Credential Handling** (`sdk/harambe/proxy.py`)
   - Proxy credentials parsed from URL but logged in some observers
   - Ensure credentials are not logged to console/files

2. **No Input Sanitization for console.log** (`sdk/harambe/core.py:585`)
   - `SDK.log()` method uses string formatting that could be exploited
   - Consider escaping special characters

3. **Temp File Handling** (`sdk/harambe/core.py:302`)
   - Download files saved to tempfile without explicit cleanup in all paths
   - Use context managers consistently

### Code Quality

1. **Type Ignore Comments**
   - Multiple `# type: ignore` comments indicate type system gaps
   - Consider proper generic typing for AbstractPage

2. **Recursive Pagination** (`core.py:244`)
   - `TODO` comment indicates pagination implementation needs refactoring
   - Could cause stack overflow with many pages

3. **Missing Error Handling in Observers**
   - Observer methods can fail silently
   - Add proper error aggregation

4. **Inconsistent Async Patterns**
   - Some functions accept both sync and async callables
   - Standardize interface

## Testing Analysis

### Coverage Overview
- **35 test files** across both packages
- Good coverage of parser types and expressions
- E2E tests present for SDK

### Well-Tested Areas
- Schema parser type conversions
- URL normalization
- Expression evaluation
- Observer serialization

### Under-Tested Areas
1. **Error conditions** - Few negative test cases
2. **Concurrency** - No tests for race conditions in observers
3. **Network failures** - Limited mocking of network errors
4. **Browser edge cases** - Download failures, timeouts
5. **Large data handling** - Memory pressure scenarios

### Test Quality Issues
- Test file naming inconsistency: `__Init__.py` vs `__init__.py`
- Some tests use hardcoded URLs that may become unavailable
- Missing integration tests between core and SDK

## Architecture Notes for AI Assistants

### When Modifying Code

1. **Schema Changes**
   - Update both `core/harambe_core/types.py` and add parser in `parser/type_*.py`
   - Add tests in `core/test/parser/`

2. **Adding SDK Methods**
   - Add to `SDK` class in `sdk/harambe/core.py`
   - Consider observer notification pattern
   - Update `__all__` exports

3. **Observer Implementations**
   - Implement full `OutputObserver` protocol
   - Handle all event types even if no-op

4. **Version Updates**
   - Update BOTH `core/pyproject.toml` and `sdk/pyproject.toml`
   - SDK depends on exact core version

### Key Patterns to Follow

```python
# Scraper function pattern
async def scrape(sdk: SDK, url: str, context: dict) -> None:
    page = sdk.page
    # ... scraping logic
    await sdk.save_data({"field": "value"})

# Decorated scraper pattern
@SDK.scraper(domain="example.com", stage="detail")
async def scrape(sdk: SDK, url: str, context: dict) -> None:
    ...

# Running a scraper
asyncio.run(SDK.run(scrape, "https://example.com", schema={...}))
```

### Important Invariants

1. **Version Sync**: Core and SDK versions must always match
2. **Schema Validation**: All `save_data` calls pass through SchemaParser when schema provided
3. **URL Normalization**: All URLs are normalized before saving/enqueueing
4. **Observer Notification**: All observers must be notified for all events

## Prioritized Improvements

### Critical Priority

1. **Fix pytest version mismatch**
   - Complexity: Low
   - Steps: Align both packages to pytest 8.x, pytest-asyncio 0.23+

2. **Add error handling for observer failures**
   - Complexity: Medium
   - Steps: Wrap observer calls in try/catch, aggregate errors

3. **Fix recursive pagination**
   - Complexity: Medium
   - Steps: Convert to iterative approach or use tail recursion

### High Priority

4. **Add comprehensive error tests**
   - Complexity: Medium
   - Steps: Add negative test cases for all parsers and SDK methods

5. **Improve proxy credential security**
   - Complexity: Low
   - Steps: Mask credentials in logs, add secure logging mode

6. **Update playwright-stealth or find alternative**
   - Complexity: Medium
   - Steps: Evaluate undetected-playwright or fork current package

### Medium Priority

7. **Add network failure resilience**
   - Complexity: High
   - Steps: Add retry logic, circuit breakers, timeout handling

8. **Improve type safety**
   - Complexity: Medium
   - Steps: Replace `# type: ignore` with proper generics

9. **Add integration test suite**
   - Complexity: High
   - Steps: Create tests that exercise full core+SDK flow

10. **Document schema field types**
    - Complexity: Low
    - Steps: Add docstrings and examples for each SchemaFieldType

### Low Priority

11. **Remove setuptools dependency**
    - Complexity: Low
    - Steps: Replace pkg_resources usage with importlib.metadata

12. **Add async context manager support to SDK**
    - Complexity: Medium
    - Steps: Implement `__aenter__` and `__aexit__`

13. **Standardize test file naming**
    - Complexity: Low
    - Steps: Rename `__Init__.py` to `__init__.py`

## API Quick Reference

### SDK Methods

| Method | Purpose |
|--------|---------|
| `SDK.run()` | Run a scraper function |
| `sdk.save_data()` | Save validated data |
| `sdk.enqueue()` | Queue URLs for scraping |
| `sdk.paginate()` | Handle pagination |
| `sdk.capture_url()` | Capture URL from click |
| `sdk.capture_download()` | Capture file download |
| `sdk.capture_html()` | Capture page HTML |
| `sdk.capture_pdf()` | Save page as PDF |
| `sdk.save_cookies()` | Save browser cookies |
| `sdk.save_local_storage()` | Save localStorage |
| `sdk.solve_captchas()` | Trigger captcha solving |
| `sdk.log()` | Log message |

### Schema Field Types

`string`, `boolean`, `integer`, `number`, `float`, `double`, `currency`, `price`, `email`, `enum`, `array`, `object`, `datetime`, `phone_number`, `url`

## License

MIT License - Copyright (c) 2025 Reworkd AI, INC

## External Resources

- GitHub: https://github.com/reworkd/harambe
- PyPI: https://pypi.org/project/harambe-sdk/
- Reworkd: https://reworkd.ai

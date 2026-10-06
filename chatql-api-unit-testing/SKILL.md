---
name: chatql-api-unit-testing
description: Write, update, and diagnose isolated pytest tests for dvh-chatql-api behavior changes and bug regressions. Use when adding test coverage or fixing a failing test in this repository.
---

# ChatQL API unit testing

Read the [development skill](../chatql-api-development/SKILL.md) for architecture and domain rules. Test implemented behavior; that document also describes target conventions and does not imply that every proposed module exists.

## Select the behavior and test layer

1. Read the request, changed code, nearby tests, and relevant fixtures before writing tests. Identify the observable contract and the failure the test should detect.
2. Extend the nearest test module under `tests/`; follow its naming and setup. Keep pure logic tests at the function or service boundary. Use route tests for HTTP contracts and isolated database tests for persistence behavior. Describe tests that exercise several layers as component tests, not pure unit tests.
3. Cover the changed behavior, relevant boundary cases, and meaningful failure paths. For chat changes, select applicable cases such as ownership isolation, missing or inactive records, validation errors, pagination ordering, feedback changes, transaction failures, or streaming cleanup. Derive expected outcomes from the actual contract rather than inventing status codes or requirements.
4. For a bug fix, add a regression test that exposes the original failure when feasible. Keep unrelated refactoring and new infrastructure outside the test task.

## Reuse the repository fixtures

- [pytest.ini](../../../pytest.ini) sets the repository Python path and excludes `tests/legacy_regression`. Preserve that exclusion unless the task specifically concerns those tests.
- [tests/conftest.py](../../../tests/conftest.py) loads [tests.chat_fixtures](../../../tests/chat_fixtures.py). Its chat fixtures are opt-in, not autouse; request only those the test needs.
- Reuse `chat_now` for a fixed clock, `chat_engine` for the isolated engine, `chat_session` for service/database work, and `chat_client` for HTTP tests when their setup matches the behavior. Read their current implementations before extending them.
- The chat fixtures avoid the production app and lifespan and use SQLite with a minimal schema. SQLite exercises SQLModel mappings but does not establish Oracle row-locking behavior. Keep Oracle-specific SQL checks and environment-backed integration validation distinct.
- [tests/api/test_chat.py](../../../tests/api/test_chat.py) is an example of HTTP/schema contract tests. Follow nearby service, database, datasource, engine, or shared tests for other areas rather than forcing every test through the chat client.

## Keep tests isolated and meaningful

- Keep the code under test real. Replace outbound database, HTTP, LLM, or datasource calls at the boundary the module actually uses; patch the lookup site and configure realistic results or failures. Do not mock the behavior being asserted.
- For pure unit tests, use focused fakes or mocks. For persistence behavior, reuse the isolated database fixtures where appropriate. Do not use production connection settings, credentials, application startup, or schema-changing operations as a shortcut to test setup.
- Control time, generated identifiers, and external responses only where they affect the assertion. Use the existing clock fixture, monkeypatch, or an established freezing pattern; avoid wall-clock sleeps and order-dependent state.
- Let fixtures and context managers own cleanup. Restore temporary dependency overrides, patched settings, sessions, and resources after the test; avoid leaking global state between cases.
- Assert observable results: returned values or schemas, HTTP responses, persisted state, emitted events, and necessary boundary interactions. Avoid tests that only repeat implementation steps or assert incidental call order.
- Use parametrization when cases share a contract and setup. Keep distinct failure modes readable. For async or streaming code, inspect the existing runner and consumption pattern and exercise completion, errors, or cancellation only as relevant to the change.

## Run and assess the tests

Run commands from the repository root using its configured Python environment. In Windows Git Bash, activate the existing environment with `source .venv/Scripts/activate` when appropriate, or use its Python executable directly. Do not assume the shell-selected Python is the repository interpreter.

Start with the changed test module or an exact test node. For example, for the existing chat API tests:

```bash
python -m pytest -q tests/api/test_chat.py
```

Choose the path for the actual change; this example is not a mandatory test for every task. After focused tests pass, run the affected neighboring tests or the wider suite when the change warrants it:

```bash
python -m pytest -q tests/
```

When coverage/CI parity is relevant, use the current [Jenkinsfile](../../../Jenkinsfile) as the source of truth. Its present pytest invocation collects application coverage and JUnit results while excluding legacy regression tests:

```bash
python -m pytest -q --junitxml=pytest-report.xml --cov=app --cov-report=xml:coverage.xml tests/ --ignore=tests/legacy_regression/
```

The CI setup includes pytest, pytest-cov, freezegun, and coverage. Confirm required tools are available in the selected environment. Do not invent a coverage threshold, change CI exclusions, or alter dependencies just to make an unrelated test task pass. Keep generated reports out of the proposed source changes.

If collection or execution fails, distinguish a missing dependency or environment problem from a failing behavior assertion. Diagnose the concrete failure; do not silence assertions, broaden skips, or remove coverage to obtain a green run. Run any environment-backed integration checks only within the requested scope, and state when they remain unverified.

## Report the result

Summarize the behavior covered, test files changed, and exact commands run. Report the observed pass/fail/skip results, any remaining failures, and checks not run with their reasons. Describe coverage only when measured. A collected test or a mocked success path alone does not demonstrate the real integration works.

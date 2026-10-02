# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Course Explainer App: a small Flask app with a home page listing courses and a per-course detail page. Python + Flask + Jinja templates, no database, no build step, no linter configured.

## Commands

Run everything from the repository root.

```bash
# Setup (no venv is checked in; README uses uv for the install step)
python3 -m venv venv
source venv/bin/activate
uv pip install -r requirements.txt      # or: pip install -r requirements.txt

# Run the dev server (debug mode) at http://127.0.0.1:5000
python3 src/app.py

# Run all tests
python3 -m unittest discover -s tests

# Run a single test
python3 -m unittest tests.test_app.AppTestCase.test_index
```

`gunicorn` and `python-dotenv` are in `requirements.txt` but nothing uses them yet: there is no `.env` file (despite the README's tree listing one), no `load_dotenv()` call, and no gunicorn config. A production invocation would need `--chdir src` (see the import note below).

## Architecture

- `src/app.py` creates the Flask `app` and registers routes with `app.add_url_rule(...)`, not decorators. The view functions live in `src/views.py` as plain functions. Adding a page means adding a function in `views.py` and a matching `add_url_rule` in `app.py`; templates reference routes by the `endpoint` name given there (`index`, `course`).
- Imports are flat, not package-relative (`from views import index, course`). `src/` has no `__init__.py` and must be on `sys.path`: this works for `python3 src/app.py` because the script's directory is added automatically, and `tests/test_app.py` inserts `src/` into `sys.path` by hand. Importing as `src.app` will fail.
- `src/models.py` defines a `Course` class (`title`, `description`, `instructor`, `duration`) and a hardcoded module-level `courses` list. This is the only data source, and **nothing imports it yet**.
- Templates are in `src/templates/`, static CSS in `src/static/css/styles.css`, both located by Flask's defaults relative to `src/app.py`.

## Known broken state

The app is partly wired up. Check these before assuming existing behavior is intended:

- **`/course/<course_id>` returns 500, and `test_course` fails.** `views.course` passes only `course_id` to the template, but `course.html` renders `course.title`, `course.description`, `course.instructor`, `course.duration` and loops over `course.topics`. The view never looks the course up in `models.courses`, and `Course` has no `topics` attribute. `test_index` passes.
- **`test_course` asserts the body contains `Course Details`.** Today that string only appears in the nav link inside `layout.html`, so the assertion depends on the layout being rendered.
- **`course_id` is an untyped string** in the route, and the index links use `1`, `2`, `3`, while `courses` is a zero-indexed list. Any lookup needs to decide on the mapping.
- **Template inheritance is used inconsistently.** `index.html` is a full HTML document that does `{% include 'layout.html' %}`, which nests a second complete document inside its body and leaves the layout's `content` block empty. `course.html` wraps `{% extends 'layout.html' %}` in its own `<html>`/`<head>`/`<body>`; with `extends`, everything outside `{% block %}` tags is discarded, so its own `<title>` never reaches the page. The intended pattern is a child template containing only `{% extends 'layout.html' %}` plus `title` and `content` blocks.
- **The index page hardcodes its course links** ("Course 1" to "Course 3") instead of iterating `models.courses`.

## Repo notes

- Only `requirements.txt` is committed; `src/`, `tests/`, `README.md` and `info.txt` are untracked. There is no `.gitignore`, so `venv/` and `__pycache__/` will show up as untracked.
- `info.txt` is a personal scratch file of setup commands, not project documentation.

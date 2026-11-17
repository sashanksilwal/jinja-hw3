# CS593 Homework 3 | Jinja issue 2069

Jinja 3.1.6 reports `output` as an external variable when an `if/elif/else`
assigns it on every path. This fork fixes that analysis while preserving
incoming values when a path skips the assignment or reads the variable first.
[Issue report](https://github.com/pallets/jinja/issues/2069) · [Investigation and test evidence](HW3-validation.md)

## Set up

Use Python 3.12 and Git. Run the commands below in the same shell.

```sh
git clone https://github.com/sashanksilwal/jinja-hw3.git
cd jinja-hw3
python3.12 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements/tests.txt MarkupSafe Babel
```

## Reproduce the original failure

The worktree at `da81cb6` contains the original Jinja 3.1.6 source and the
reproducing test. Its parent is the upstream release commit `1520688`.

```sh
git worktree add --detach ../jinja-hw3-before da81cb6
test_id=tests/test_api.py::TestMeta::test_conditional_assignment_all_branches
(cd ../jinja-hw3-before && PYTHONPATH=src python -m pytest -q "$test_id")
```

Expect one assertion failure. The reported set is `{'control', 'output'}` and
the test expects `{'control'}`.

## Verify the fix

From the main checkout, run the same test, the focused regression tests, and
the full suite. `PYTHONPATH=src` selects the code in the current checkout.

```sh
PYTHONPATH=src python -m pytest -q "$test_id"
PYTHONPATH=src python -m pytest -q tests/test_api.py::TestMeta tests/test_idtracking.py
PYTHONPATH=src python -m pytest -q
```

The reproducer passes on patch commit `e989db9`. The full validation run
passed all 927 tests on CPython 3.12.

# Jinja

Jinja is a fast, expressive, extensible templating engine. Special
placeholders in the template allow writing code similar to Python
syntax. Then the template is passed data to render the final document.

It includes:

-   Template inheritance and inclusion.
-   Define and import macros within templates.
-   HTML templates can use autoescaping to prevent XSS from untrusted
    user input.
-   A sandboxed environment can safely render untrusted templates.
-   AsyncIO support for generating templates and calling async
    functions.
-   I18N support with Babel.
-   Templates are compiled to optimized Python code just-in-time and
    cached, or can be compiled ahead-of-time.
-   Exceptions point to the correct line in templates to make debugging
    easier.
-   Extensible filters, tests, functions, and even syntax.

Jinja's philosophy is that while application logic belongs in Python if
possible, it shouldn't make the template designer's job difficult by
restricting functionality too much.


## In A Nutshell

```jinja
{% extends "base.html" %}
{% block title %}Members{% endblock %}
{% block content %}
  <ul>
  {% for user in users %}
    <li><a href="{{ user.url }}">{{ user.username }}</a></li>
  {% endfor %}
  </ul>
{% endblock %}
```

## Donate

The Pallets organization develops and supports Jinja and other popular
packages. In order to grow the community of contributors and users, and
allow the maintainers to devote more time to the projects, [please
donate today][].

[please donate today]: https://palletsprojects.com/donate

## Contributing

See our [detailed contributing documentation][contrib] for many ways to
contribute, including reporting issues, requesting features, asking or answering
questions, and making PRs.

[contrib]: https://palletsprojects.com/contributing/

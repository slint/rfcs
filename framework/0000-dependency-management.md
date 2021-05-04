- Start Date: 2020-03-29
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Alex Ioannidis

# Dependency management

## Summary

- Bundling of 3rd-party dependencies in "coordinator" packages
- For Travis CI builds pass only the module's "pure" extras to
  `requirements-builder` (i.e. not `all`/`tests`/`docs`)
- Introduce tilde constraints (`~=`) to all modules.

## Motivation

Dependency management is hard and unfortunately an unsolved problem for the
Python packaging ecosystem, and as an extension of that, also for the Invenio
ecosystem. Common issues that developers come across:

- 3rd-party dependencies release versions with breaking changes, that require
  us to have to quickly respond by either:
  - Writing compatibility fixes
  - Accepting the breaking changes and releasing a new backwards-incompatible
    version of the Invenio module
  - Upper-pin versions of the problematic 3rd-party modules
- 3rd-party dependency declarations are scattered around the different modules.
  This causes the following problems:
  - It's difficult to know where a library is being installed from
  - When deciding to bump the minimum version (or upper-pin) a 3rd-party
    library, many changes have to be made across all Invenio modules.
- When running a module's tests we should always be using the latest versions
  of test dependencies, i.e. there's no reason to run tests with minimum
  requirements in Travis CI.

## Detailed design

### Coordinator modules

To efficiently manage 3rd-party dependenceis, i.e. Python packages that are not
part of the `inveniosoftware` GitHub organization, the concept of "coordinator"
modules is introduced. These are core Invenio modules that are responsible for
properly specifying 3rd-party dependencies.

Here's a table of all the current coordinator modules:

| Module | Dependencies                                                                                                         |
| ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `invenio-base`     | `Flask`, `werkzeug`, `six`, `blinker`                                                                                |
| `invenio-accounts` | `Flask-Security`, `Flask-KVSession`, `Flask-Login`                                                                   |
| `invenio-admin`    | `Flask-Admin`                                                                                                        |
| `invenio-celery`   | `celery`, `kombu`, `msgpack`                                                                                         |
| `invenio-rest`     | `webargs`, `marshmallow`                                                                                             |
| `invenio-i18n`     | `Flask-BabelEx`                                                                                                      |
| `invenio-db`       | `Flask-SQLAlchemy`, `Flask-Alembic`, `SQLAlchemy`, `SQLAlchemy-Continuum`, `SQLAlchemy-Utils`, `psycopg2`, `pymysql` |
| `invenio-search`   | `elasticsearch`, `elasticsearch-dsl`                                                                                 |
| `pytest-invenio`   | `pytest`, `coverage`, `isort`, `check-manifest`                                                                      |

### `requirements-builder` options for Travis CI

`requirements-builder` is used in Travis CI builds, so that we can efficiently
test for all supported versions of a module's dependencies based on the ones
defined in `setup.py` and `requirements-devel.txt` files.

### Tilde requirement constraints

This protects us when breaking changes and major version updates come to
modules. The only downside is that since this is equivalent to upper pinning,
if someone wants to use a newer version of a 3rd-party library (e.g. for a new
feature) they have to track down where this dependency is in Invenio and bump
it there.

## Example

### Coordinator modules

> `Flask -> Invenio-Base` + `Flask-BabelEx -> Invenio-I18N` example

### `requirements-builder` options for Travis CI

> `.travis.yml` fix example

### Tilde requirement constraints

> `setup.py` fix example

## How we teach this

- Documentation section in main `invenio` docs and this RFC
- Maintainers of Invenio modules will need to be aware of best practices when
  reviewing PRs that modify dependencies in `setup.py`

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Invenio,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

- **Coordinator modules**
  -
- **Tilde requirement constraints**
  - As a replacement to using `>=` without upper pinning, this might be
    blocking users of Invenio from getting access to newer features.
- **`requirements-builder` options for Travis CI**
  - None obvious
- In general, all of these proposals require module-wide changes in order to be
  implemented. On the positive side, they can be done in a gradual fashion,
  since the intent is to upgrade to a more safe/stable dependency management
  style.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

### SemVer incompatibility

Although a lot of the proposed changes are based on the premise that SemVer is
followed by 3rd-party modules, inside the Invenio ecosystem, modules have
"dodged" SemVer compatibility throughout their releases, by shifting the
"semantic" meaning by one version slot to the right. What this means is that
from a patch version bump (e.g. `v1.0.1 -> v1.0.2`), one might not only get
backwards-compatible patches/fixes, but also feature additions.

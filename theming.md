- Start Date: 2020-04-20
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Lars Holm Nielsen
- State: DRAFT

# React-Semantic UI support for Invenio

## Summary

> One paragraph explanation of the feature.

## Motivation

In Invenio v3.0 we deprecated support for AngularJS as well as AMD/RequireJS builds, which was originally integrated with Invenio. Invenio v3.1 added support for the Webpack build system (i.e. replace for AMD/RequireJS builds). This RFC completes the last part of the transtion of moving away from AngularJS.

A [hackathon was held](https://github.com/inveniosoftware/invenio/wiki/JavaScript-framework-selection) in February 2018 to select a new JS framework. Angular, React, Vue and WebComponents were tested, and the outcome was to go with [**React**](https://reactjs.org/).

A key part of the JS framework was also to select a UI framework. Several UI frameworks were evaluated including Bootstrap, Material UI, SemanticUI and AntDesign. The outcome was to go with [**SemanticUI**](https://semantic-ui.com/).

The ILS team decided to make InvenioILS into a Single Page Application based on React and React-SemanticUI. As part of InvenioILS, the ILS team built React-Searchkit as a replacement for Invenio-Search-JS.

The current RFC includes:

- UI frameworks tested
- React-SemanticUI
- React Components
- The need to support Bootstrap style components

## Prerequisites

Following is a quick recap of how theming currently works in Invenio.

### Views and templating

Flask renders Jinja templates using the [``flask.render_template``](https://flask.palletsprojects.com/en/1.1.x/api/#flask.render_template) function. For instance:

```python
from flask import render_template

@app.route('/')
def myview():
    return render_template('mytemplate.html')
```

Jinja searchers for the template ``mytemplate.html`` in predefined **search paths**. The Flask application's Jinja environment defines these search paths. Jinja searchers the search paths in order. If the same template name exists in multiple search paths, the first hit wins.

Currently, Invenio has a template search path in ``<instance folder>/templates`` as well as most blueprints define a template folder.

#### Jinja templates

A Jinja template can extend on another. For instance the template ``mytempalte.html`` can extend a ``base.html`` template:

```jinja
{# mytemplate.html #}
{% extends "base.html" %}
{% block css %}
    # ...
{% endblock %}
```

The template ``base.html`` defines blocks which can be overwritten by the extending templates (eg. the ``css`` block)

```jinja
{# base.html #}
<html>
    <head>
        {% block css %}{% endblock %}
    </head>
</html>
```

#### Assets (CSS/JS)

Invenio uses **Webpack** for building and bundling web assets such as CSS and JavaScript.

#### Webpack

An Invenio module can define a Webpack bundle which provides one or more entries. An entry tells Webpack to start with e.g. ``theme.scss`` and build it. Webpack will automatically resolve all imports. For instance

```python
bundle = WebpackBundle(
    entry = {
        "theme": "./theme.scss"
    }
)
```

#### Manifests

Invenio-Assets is responsible for collecting all ``assets/`` folders from other Invenio modules and merge them into a full Webpack project. The Webpack project is then built, and produces a manifest, which is a JSON file that points to all the built assets. A manifest file could look like this:

```json
{
    "theme.css": "/static/dist/css/theme.<hash>.css"
}
```

#### Using manifests in templates

This manifest is then read by Flask, and allows a Jinja template to include the built file:

```jinja
{% block css %}
    {# This will be rendered as <link src="/static/dist/css/theme.<hash>.css" ... /> #}
    {{ webpack['theme.css'] }}
{% endblock %}
```

## Detailed design

### Theming support

In order to provide backward compatibility for current Invenio releases, Invenio must be able to support multiple UI frameworks concurrently. This also means that possible future transitions to other UI frameworks will be easier, and clients who must use (e.g. due to imposed requirements) another framework can do so. Supporting multiple UI frameworks can be achieved through theming support.

#### Selecting a theme

Universally, we use the `APP_THEME` configuration to provide a list of theme names/prefixes that are used in "fallback" fashion to search for templates and assets. In your config:

```python
# Use the "semantic-ui" theme templates and assets:
THEME = ['semantic-ui']

# Use the "my-site" theme, but if a template or asset is not found
# fallback to "semantic-ui":
THEME = ['my-site', 'semantic-ui']
```

### Basic principle: Template theme folders

Currently Invenio packages define templates inside an `<invenio_package>/templates` folder which gets "registered" via the package's blueprint(s). Flask uses a central "Jinja loader" (`app.jinja_env.loader`) which knows about all template folders registered via blueprints. Thus, any file that lives under a templates folder, can be used in a `render_template` call in a view or inside a Jinja template via `extends`/`include` statements.

:::warning
The `templates/<invenio_package>` namespacing convention is critical to make sure that there are no name clashes between templates. Since the loading order of blueprints is random during application initialization, if two packages both define a `templates/page.html`, you cannot know which one will be used when you call `render_tempalte('page.html')`. One can safely assume though that templates inside the `<instance_folder>/templates` folder will be looked-up first.
:::

To "inject" the theme prefixes we attack the problem to its core and directly implement the logic at Flask's global Jinja loader, `app.jinja_env.loader`. When you lookup the `invenio_theme/page.html` template now the following is done:

1. We take in order each prefix from `APP_THEME` (let's assume it's set to `['my-site', 'semantic-ui']`) and prepend it to the template name, but also keep the original template name, thus generating the following ordered list:
   - `my-site/invenio_theme/page.html`
   - `semantic-ui/invenio_theme/page.html`
   - `invenio_theme/page.html`
2. We use the above generated template names to sequentially lookup the template in the global Jinja loader
3. Once we get a match we return the template. Flask also caches this lookup for the future.
4. If we don't find the template a `TemplateNotFound` error is raised.

The lookup method described above is not only used for `render_template` calls but also inside Jinja templates themselves when `{% extends invenio_theme/page.html %}` or `{% include invenio_theme/page.html %}` is used.

This templating prefixing logic is implented inside `invenio-app`, since this is the point where the Invenio Flask application is originally created.

#### Addition of Semantic-UI templates

Based on the above method, to add Semantic-UI versions of the existing templates implemented throughout the different Invenio modules, a `templates/semantic-ui` folder will be created in each module (if it actually needs to provide Jinja templates), containing a converted version of every Bootstrap v3 template that is already defined.

#### What will happen to the existing Bootstrap v3 templates?

In order to keep backwards compatibility, and allow users of Invenio to selectively use the theme prefixing loading method, the existing old Bootstrap v3 versions of Jinja templates used so far, will remain in their exact locations. This assures that (given that the `APP_THEME` variale is set to `None`) past calls to `render_template('invenio_theme/page.html')` will still refer to the old Bootstrap v3 template.

#### Template APIs

Stable APIs for templates become critical now that we add new themes as multiple templates rely on the same template context. Stable APIs for templates means any of the following changes would be a breaking API change:

- Template names, since they're used in `render_template` and `extends`/`include` calls
- Modification and removals in the template context (additions are fine though)
- Additions, modifications, and removals of template blocks
- Parameters and behaviour of macros

#### Python code

We will have to make a review of Python code to see if there's any places that CSS classes are being specified. In these cases, there need to be an indirection between what is put in the template context.

#### Inline HTML

Also, we have to check inline HTML code in various modules. Must notably e.g. WTForms may have specific theming,

### Basic principle: Themed webpack bundles

The existing Webpack bundles define two things:

- **Entries**, allowing JS code to run or (S)CSS styles to be defined
- **Dependencies**, the NPM packages that the assets depend on. The NPM dependencies can be JS libraries like React/Angular, or CSS libraries like Semantic-UI/Bootstrap.

The issue for now is that a Webpack bundle can only define one set of entries and dependencies:

```python
theme = WebpackThemeBundle(
    __name__,
    'assets',
    entry={
        'base': './js/invenio_theme/base.js',
        'theme': './scss/invenio_theme/theme.scss',
    },
    dependencies={
        'bootstrap-sass': '~3.3.5',
        'font-awesome': '~4.4.0',
        'jquery': '~3.2.1',
    }
)
```

If we treat Webpack entries as the main component/entity of the **Assets API**, we can define a new type of "themed" Webpack bundle, which allows defining two sets of entries and dependencies for each theme:

```python
theme = WebpackThemeBundle(
    __name__,
    'assets',
    default='bootstrap3',
    themes={
        'bootstrap3': dict(
            entry={
                'base': './js/invenio_theme/base.js',
                'theme': './scss/invenio_theme/theme.scss',
            },
            dependencies={
                'bootstrap-sass': '~3.3.5',
                'font-awesome': '~4.4.0',
                'jquery': '~3.2.1',
            }
        ),
        'semantic-ui': dict(
            entry={
                'base': './js/invenio_theme/base.js',
                'theme': './scss/invenio_theme/theme.scss',
            },
            dependencies={
                'semantic-ui': '~2.4.0',
                'jquery': '~3.2.1',
            }
        ),
    },
)
```

This also allows keeping old calls to `{{ webpack['theme.css'] }}` backward compatible.

### Administration interface

The current Invenio-Admin interface is based on Bootstrap, and it's likely large task to upgrade it to support Semantic-UI. In addition the usefulness of the administration is somewhat limited. On the positive side, Flask-Admin has "vendored" its Bootstrap dependency. Thus, we suggest that we simply leave the admin as it is.

### Removal of Flask-Assets, Angular and AMD/RequireJS

Invenio v3.0 deprecated AngularJS, AMD/RequireJS and Flask-Assets in favour of Webpack builds. Invenio v3.2 still has Flask-Assets support for backward compatibility. Adding theming, support means that the old Flask-Assets integration should be completely removed in order to avoid confusion and problems with the build system. Also this would simplify some of the core templates.

## How we teach this

### Easy upgrade guide

### Deprecation notice

### Sound reasoning for choice

### Existing documentation

- Tutorial
- Cookiecutter

## Drawbacks

### Semantic UI maintainance status

- Karolina: How do we address the problem of Semantic-UI being a "dead" project?
  - Marco: More info https://github.com/Semantic-Org/Semantic-UI/issues/6465#issuecomment-590921008

## Alternatives

Thera are essentially two alternatives we can consider:

1. Stay with Boostrap 3 (existing solution)
2. Transition to another UI framework (e.g. Bootstrap 4 or 5)

### Staying with Bootstrap 3

Bootstrap 3 was released August 2013 and has been maintained for a long time. Instead of adding support for a new framework we could simply developed the new React based components against Bootstrap 3.

**Pros**

- Fast to ship: There would be no need for special handling of themes and templates. No templates to migrate.

**Cons**

- Outdated: Bootstrap 3 was superseded by Bootstrap 4 in August 2018, and reached end of life July 2019. We would be developing a lot of new code against an old framework.
- UI-wise we would not evolve from state of the art in 2013.
- ILS is on SemanticUI (chosen after a review of multiple frameworks), and thus we would have different styling on core Invenio products.
### Transition to Bootstrap 4 or 5

Instead of migrating to another framework, we could migrate to Bootstrap 4 or 5.

**Props**

- Bootstrap is rather stable compared to many other frameworks.

**Cons**

- Major version upgrades are as hard as upgrading to another UI framework.
- React support for Bootstrap is limited or not as good as other frameworks.
- A new Boostrap version is essentially a different framework, thus we don't have any experience with it so far.
- End of life is quite short (see [here](https://github.com/twbs/release))


#### Selecting a theme

Option 1: We imagine that a user could select a specific theme via either a config variable:

```python
THEME = 'semantic-ui'
```

Fallback option for template loading
```python
THEME = ['zenodo', 'semantic-ui']
```

or

Option 2: simply by only allowing one *theme package* to be installed.

### Basic principle: Template duplication

```
# semanticui/login_user.html
<div class="">...</div>

# bootstrap3/login_user.html
<div class="container">...</div>
```
TODO: Provide an example

### Template package: Discussion

Scenario: Invenio-Accounts `v1.0.0` has a template `login_user.html`. For some reasons, we need to change the template and the change will be backward incompatible. We will release Invenio-Accounts `v2.0.0`.

1. **Only central template package approach**

Versioning through the central package PyPI version:

```
invenio-templates-semanticui/templates/invenio_accounts/login_user.html
```

or (package versioning)

```
invenio-templates-semanticui/templates/invenio_accounts/v1.x/login_user.html
invenio-templates-semanticui/templates/invenio_accounts/v2.0/login_user.html
```

or (template versioning)

```
invenio-templates-semanticui/templates/invenio_accounts/login_user_v1.html
invenio-templates-semanticui/templates/invenio_accounts/login_user_v2.html
```

Notes:
- Module is installed by instance or main Invenio
- Pro: User advantage - easy for a user to understand all templates and what to override.
- Cons: The versioning of the templates
    - Alex hate it: mess of PRs and commits and different branches! Crazy to maintain versions

2. **Only module template approach**

Versioned through the module package PyPI version:

```
invenio-accounts/templates/<theme>/invenio_accounts/login_user.html
```

- Pro: Versioning is easy
- Cons: Templates are scattered around. Hard for a user to add a new UI framework.


- Customizing the loeader in order to prefix template name.
- We need good template documentation:
    - variable names for tempaltes
    - api

3. **Hybrid approach** (i.e use both)

Combine option 2 with one of the schemes from option 1. For instance:

```
invenio-theme-semanticui/templates/invenio_accounts/login_user_v1.html
invenio-theme-semanticui/templates/invenio_accounts/login_user_v2.html
invenio-accounts/templates/<theme>/invenio_accounts/login_user.html
```

- Pro: User is happy, developer still not happy.
- Con: Versioning still needed, A lot of duplication

- Provide bootstrap3 and semantic-ui templates in each Invenio module
- Provide a Invenio-Theme-SemanticUI module with templates from all modules, to make it easy for users to make a new theme


4. **Variable approach**

```
# In Invenio-Accounts
ACCOUNTS_LOGIN_TEMPLATE = "invenio_accounts/login_user.html"

render_template(ACCOUNTS_LOGIN_TEMPLATE)

# In Invenio-Templates-SemanticUI
ACCOUNTS_LOGIN_TEMPLATE = invenio_accounts/semanticui/login_user.html

# In My instance
/var/instance/templates/invenio_accounts/semanticui/login_user.html
```

- Pro: Both a central template package, and a modules, but only two templates. No need for template loading
- Con:
    - Not really needed if we go with option 2

### Importance of Template backward compatibility

-

### Theme package

In order to package and distribute a theme, we propose that all templates are gathered in a theme package per framework. This also allows easier forking and creation of new themes for Invenio by the community. Also it gives a better overview of everything involved in an theme.

A Semantic UI theme package structure could look like below (more explanation and details below):

    invenio-theme-semanticui/
        invenio_theme_semanticui/
            ext.py
            views.py
            webpack.py
            config.py
            templates/
                semantic-ui/<module>/<template>
            assets/
                semantic-ui/
            static/
        setup.py



##### Questions

- Nicola: can we provide a very short example here? From my understanding, we would like to have for example the following folders structure:
    ```
    assets/
        <theme>/
            homepage.css
    templates/
        <theme>/
            homepage.html
    ```
    - Lars: I've added an example

- Karolina:
    - by <module> in the package structure did you mean invenio module(that is what I assumed) or semantic element?
    - by template I understand you mean Jinja template?
    - we need to take into account the semantic-ui's own structure when designing this ( invenio modules x semantic-ui elements matrix, what will "overtake" the style in the cascade inheritance)


#### Releasing

A theme package gathers templates from different Invenio modules. For instance templates from both Invenio-Accounts and Invenio-OAuth2Server will live in the a Invenio-Theme-SemanticUI package. All three modules are versioned independently. At any given point, multiple versions of each package can be supported. For instance, Invenio versions can specify the following dependencies:

- Invenio v3.4:
    - Invenio-Accounts>=1.1.0,<1.2.0
    - Invenio-OAuth2Server>=1.3.0,<1.4.0
    - Invenio-Theme-SemanticUI>=1.0.0,<1.1.0
- Invenio v3.5:
    - Invenio-Accounts>=1.5.0,<1.6.0
    - Invenio-OAuth2Server>=1.4.0,<1.5.0
    - Invenio-Theme-SemanticUI>=1.2.0,<1.3.0

Now the question is, can Invenio-Theme-SemanticUI hold templates for:

1. Only one version of a module (e.g. Invenio-Accounts 1.1.x)?
2. All versions of a module (e.g. Invenio-Accounts 1.1.x and 1.5.x)?

If option 1 is chosen, then Invenio-Theme-SemanticUI must be fully aligned with an Invenio releases. I.e. Invenio-Theme-SemanticUI v1.0.x can only hold templates from Invenio-Accounts 1.1.x and Invenio-OAuth2Server 1.3.x released in Invenio v3.4. This makes it hard to use Invenio-Theme-SemanticUI if you don't depend on Invenio releases, but individual modules instead, as you may potentially have the following situation:

- My Repository:
    - Invenio-Accounts>=1.1.0,<1.2.0 (same as v3.4)
    - Invenio-OAuth2Server>=1.4.0,<1.5.0 (same as v3.5)
    - Invenio-Theme-SemanticUI>=1.0.0,<1.1.0 (??)

In above case Invenio-Theme-SemanticUI would have the templates for a previous version of Invenio-OAuth2Server.

If option 2 is chosen, then we must also include the package major/minor version into the template path loader So we potentially have multiple templates per version (e.g.``<theme>/<module>/<version>/<template path>``)


##### Questions

- Nicola: I am not sure I understand this point. I have Invenio 3.3 that has `bootstrap3`, so I have `templates/bootstrap3/homepage.html`, but Invenio 3.4 I have `templates/semantic-ui/homepage.html`. Can you show a very quick example of a conflict and both solutions?
- Nicola: I guess that for previous Invenio versions, e.g. Invenio 3.1/3.2, if the `<theme>` folder does not exist, it will default to `bootstrap`.
    - Lars: I've tried to expand the section to explain the problem.
- Karolina:
    - I had problems to understand the versioning problem, could we discuss in details? Maybe draw some dependencies to make it more visual.
    - Even with the 1st point, I have a feeling that the problem is very invenio-modules based and it makes me wonder why not to have one invenio theme which has all the needed "tools/components" rather than module based approach? I kinda have a feeling that it could be difficult to maintain with the per-module approach. I can elaborate in a discussion.
- Lars: Why not a hybrid approach?

#### Update policy

In both above solutions, we need a policy for how we keep a main theme package up to date.

#### Questions

- Lars: How do we develop new code in Invenio modules, if templates live in a theme package? (Nicola: can you elaborate? :))
    - Lars: I meant to say that what is it a developer needs to do when developing templates in Invenio-<module>? I.e. does a template existing in Invenio-<module> or only in Inveni-Theme-<theme>? How do I make PRs to both and make test pass? Just practical issues of having things separatae.

### Web assets

The Jinja templates are in full control of CSS and JavaScript loading, and thus could also simply use different CSS and JS bundles. E.g. currently today ``invenio_theme/page.html`` contains this:

```jinja
{%- block css %}
{{ webpack['theme.css']}}
{%- endblock css %}
```

Invenio-Theme:

```python
from flask_webpackext import WebpackBundle

theme_bootstrap = WebpackBundle(
    __name__,
    'assets',
    entry={
        'theme': './scss/invenio_theme/theme.scss',
    },
    dependencies={
        'bootstrap-sass': '~3.3.5',
        'font-awesome': '~4.4.0',..
    }
)

theme_semanticui = WebpackBundle(
```

The templates could either:

1. Reuse definitions as much as possible (i.e. theme templates use a common block with ``{{ webpack['theme-{}.css'.format(THEME)] }}``).
2. Be in full control of assets (i.e. theme templates hard code the theme ``{{ webpack['theme-semantic-ui.css'] }}``)

Likely option 2 will be the most flexible, and also the easiest to understand. Also, if theme packages are used, each theme package can provide their own Webpack entry points for building their assets.

##### Questions

- Nicola: I am probably missing something again here, but since at build time, we are building only 1 theme, `bootstrap3` or `semantic-ui`, why namespacing assets? I would say that the final generated assets should be simply `theme.css`...

### Webpack config

##### Questions

- Lars: Can Invenio-Assets webpack config be reused for each theme? If not, then how do we fix it? Nicola: me/Karolina might be able to help with this, the question is: do we use default Semantic UI theme or do we need to customize Semantic UI components? In any case the answer I "think" is yes, we just need a LESS loader in Webpack. We can chat IRL and summarize here.

### Theme package implementation

We have to investigate where and how the theme package support needs to be integrated.

- Invenio-App defines the Jinja template loader, likely a part needs to live here.
- Invenio-Theme has generic and theme specific parts that would need to be split.
    - Generic parts:
        - Initialisation of Flask-Menu/Breadcrumb
        - Registration of handlers for error pages (404, 500, etc)
        - Configuration variables for e.g THEME_LOGO
    - Theme specific:
        - Templates
        - Assets
        - Webpack module
    - Documentation has also both generic and theme specific parts.

#### Questions:

- Lars: A key decision to take is how the theme package is loaded and if there's anything special beyond a normal Invenio module.
- Lars: Developing invenio modules: We need to figure out what impact it has to create theme packages?


### Theme package: Boostrap 3

It's currently unclear if it's best to:

1. leave existing templates in place in the current modules
2. move the templates out into a new Boostrap 3 theme package.

Moving out the templates to a new Bootstrap 3 package could serve as a good test case that focuses on just getting the theming support working. A next step would then be to create a new package for Semantic UI. In addition, getting the templates out of the modules, means we later don't have to come back and remove the Bootstrap 3 templates.

Separating out the templates, would also ensure that we wouldn't build too many Webpack bundles (one for every theme installed). However, there should be a very easy upgrade path from Invenio v3.3.

For the package design itself, please see the Semantic-UI theme package.

#### Questions:

- Lars: If we don't create such a package, how do we then get rid of the boostrap 3 templates in the future and avoid to maintain them?
- Lars: If we do create a package, what impact does it have on the existing modules in terms of e.g. making test pass.

### Theme package: Semantic UI

The Semantic-UI theme package would be a normal Invenio module:

```
invenio-semantic-ui/
    invenio_semantic_ui/
        ext.py
        views.py (blueprint needed to register template)
        webpack.py
        config.py
        templates/
            semantic-ui/<module>/<template>
        assets/
            semantic-ui/
        static/
    setup.py
```

#### Flask/Invenio extension (ext.py)

The Flask extension is needed for registering e.g. configuration. Also, it would allow the package to provide e.g. template filters it needs by itself.

#### Configuration (config.py)

The configuration could allow to easily enable/disable certain features of the theme package

#### Templates folder

The template folder would be structured according to how the template loader requires it, and what the Invenio modules expect their templates to be named.

#### Webpack: CSS/JS (webpack.py)

The webpack bundles are specified by this module. Currently the theme would need the following special code:

- Lars: Language selector - this could likely be implemented in pure JavaScript to avoid framework dependency.


### Summary

Overall, no matter which UI framework we chose the general concern is that we should expect the framework to be out of support pretty quickly.


## Unresolved questions

### Semantic UI stability

- ....

## Resources/Timeline

### Release plan

1. Alpha release of Invenio v3.4 with Semantic UI support
1. Release Invenio v3.4 with SemanticUI support but with Bootstrap 3 as default framework (announce pending deprecation).
1. Release Invenio v3.5 with SemanticUI as default UI framework (deprecation)
2. Release Invenio v3.6 with Bootstrap 3 theme removed.

**Time frame**

Assuming a release every 3 months, first release happening in 2020 Q3, then the total transition will be completed by mid-2021, and Bootstrap 3 would be supported until minimum early 2022. All in all it will take almost 2 years to have Bootstrap 3 completely removed.

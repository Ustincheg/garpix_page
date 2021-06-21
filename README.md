# Garpix Page

Convenient page structure with any context and template.
It is suitable not only for a blog, but also for large sites with a complex presentation.
Supports SEO.

## Quickstart

Install with pip:

```bash
pip install garpix_page
```

Add the `garpix_page` and dependency packages to your `INSTALLED_APPS`:

```python
# settings.py

INSTALLED_APPS = [
    'modeltranslation',
    'polymorphic_tree',
    'polymorphic',
    'mptt',
    # ... django.contrib.*
    'django.contrib.sites',
    'garpix_page',
    # third-party and your apps
]

SITE_ID=1

LANGUAGE_CODE = 'en'
USE_DEFAULT_LANGUAGE_PREFIX = False

LANGUAGES = (
    ('en', 'English'),
)

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': ['templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

```

Package not included migrations, set path to migration directory. Don't forget create this directory (`app/migrations/garpix_page/`) and place empty `__init__.py`:

```
app/migrations/
app/migrations/__init__.py  # empty file
app/migrations/garpix_page/__init__.py  # empty file
```

Add path to settings:

```python
# settings.py

MIGRATION_MODULES = {
    'garpix_page': 'app.migrations.garpix_page',
}
```

Run make migrations:

```bash
python manage.py makemigrations
```

Migrate:

```bash
python manage.py migrate
```

Now, you can create your models from `BasePage` and set template and context. See example below.

### Important

**Page (Model Page)** - model, subclass from `BasePage`. You create it yourself. There must be at least 1 descendant from BasePage.

**Page Type** - hardcoded type of page. Required for the ability to set a **Page Model** has different behavior and representation on different URLs.

**Context** - includes `object` and `request`. It is a function that returns a dictionary. Values from the key dictionary can be used in the template.

**Template** - standard Django template.

### Example

Urls:

```python
# app/urls.py

from django.contrib import admin
from django.urls import path, re_path
from django.conf.urls.i18n import i18n_patterns
from garpix_page.views.page import PageView
from multiurl import ContinueResolving, multiurl
from django.http import Http404
from django.conf import settings

urlpatterns = [
    path('admin/', admin.site.urls),
]

urlpatterns += i18n_patterns(
    multiurl(
        path('', PageView.as_view()),
        re_path(r'^(?P<url>.*?)$', PageView.as_view(), name='page'),
        re_path(r'^(?P<url>.*?)/$', PageView.as_view(), name='page'),
        catch=(Http404, ContinueResolving),
    ),
    prefix_default_language=settings.USE_DEFAULT_LANGUAGE_PREFIX,
)

```

Models:

```python
# app/models/page.py

from django.db import models
from garpix_page.models import BasePage


class Page(BasePage):
    content = models.TextField(verbose_name='Content', blank=True, default='')
    
    template = 'pages/default.html'

    class Meta:
        verbose_name = "Page"
        verbose_name_plural = "Pages"
        ordering = ('-created_at',)


# app/models/category.py

from garpix_page.models import BasePage


class Category(BasePage):
    template = 'pages/category.html'

    def get_context(self, request=None, *args, **kwargs):
        context = super().get_context(request, *args, **kwargs)
        posts = Post.on_site.filter(is_active=True, parent=kwargs['object'])
        context.update({
            'posts': posts
        })
        return context

    class Meta:
        verbose_name = "Category"
        verbose_name_plural = "Categories"
        ordering = ('-created_at',)

        
# app/models/post.py

from django.db import models
from garpix_page.models import BasePage


class Post(BasePage):
    content = models.TextField(verbose_name='Content', blank=True, default='')
    
    template = 'pages/post.html'

    class Meta:
        verbose_name = "Post"
        verbose_name_plural = "Posts"
        ordering = ('-created_at',)
```

Admins:

```python
# app/admin/__init__.py

from .page import PageAdmin
from .category import CategoryAdmin
from .post import PostAdmin


# app/admin/page.py

from ..models.page import Page
from django.contrib import admin
from garpix_page.admin import BasePageAdmin


@admin.register(Page)
class PageAdmin(BasePageAdmin):
    pass

# app/admin/category.py

from ..models.category import Category
from django.contrib import admin
from garpix_page.admin import BasePageAdmin


@admin.register(Category)
class CategoryAdmin(BasePageAdmin):
    pass

# app/admin/post.py

from ..models.post import Post
from django.contrib import admin
from garpix_page.admin import BasePageAdmin


@admin.register(Post)
class PostAdmin(BasePageAdmin):
    pass

```

Translations:

```python
# app/translation/__init__.py

from .page import PageTranslationOptions
from .category import CategoryTranslationOptions
from .post import PostTranslationOptions

# app/translation/page.py

from modeltranslation.translator import TranslationOptions, register
from ..models import Page


@register(Page)
class PageTranslationOptions(TranslationOptions):
    fields = ('content',)


# app/translation/category.py

from modeltranslation.translator import TranslationOptions, register
from ..models import Category


@register(Category)
class CategoryTranslationOptions(TranslationOptions):
    fields = []

# app/translation/post.py

from modeltranslation.translator import TranslationOptions, register
from ..models import Post


@register(Post)
class PostTranslationOptions(TranslationOptions):
    fields = ('content',)

```

Templates:

```html
# templates/base.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    {% include 'garpix_page/seo.html' %}
</head>
<body>
<main>
    {% block content %}404{% endblock %}
</main>
</body>
</html>



# templates/pages/home.html

{% extends 'base.html' %}

{% block content %}
<h1>{{object.title}}</h1>
<div>
    {{object.content|safe}}
</div>
{% endblock %}



# templates/pages/default.html

{% extends 'base.html' %}

{% block content %}
<h1>{{object.title}}</h1>
<div>
    {{object.content|safe}}
</div>
{% endblock %}



# templates/pages/category.html

{% extends 'base.html' %}

{% block content %}
<h1>{{object.title}}</h1>
{% for post in posts %}
    <div>
        <h3><a href="{{post.get_absolute_url}}">{{post.title}}</a></h3>
    </div>
{% endfor %}

{% endblock %}



# templates/pages/post.html

{% extends 'base.html' %}

{% block content %}
<h1>{{object.title}}</h1>
<div>
    {{object.content|safe}}
</div>
{% endblock %}

```

Now you can auth in admin panel and starting add pages.

# Changelog

See [CHANGELOG.md](CHANGELOG.md).

# Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

# License

[MIT](LICENSE)
# Django Views, URL Routing & Templates

> This guide explains how Django routes incoming web requests, processes them through views, and renders presentations, with an emphasis on architectural trade-offs and interview questions.

---

# 1. URL Routing & The URL Resolver

## Interview Question

**How does Django's URL resolver work, and why should you use `reverse()` instead of hardcoded paths?**

---

### Answer

The URL resolver maps incoming request paths (like `/orders/123/`) to the exact Python view responsible for handling them.

When a request arrives, Django reads the URL path and checks it against the patterns listed in `urlpatterns` from top to bottom.

```python
# urls.py
from django.urls import path, register_converter
from . import views

urlpatterns = [
    path('articles/', views.article_list, name='article-list'),
    path('articles/<int:article_id>/', views.article_detail, name='article-detail'),
    path('articles/<slug:slug>/', views.article_by_slug, name='article-slug'),
]
```

### Path Converters

Notice the `<int:article_id>` syntax.

Path converters perform two jobs at once:

1. **Pattern Matching**: Ensuring only digits match `<int:id>`. A request to `/articles/abc/` will fail the `<int>` match and fall through to the next pattern.
2. **Type Casting**: Django automatically converts the matched string into a Python `int` before passing it as a keyword argument to your view function.

Django ships with default converters:
- `int`: Matches one or more digits and converts to `int`.
- `str`: Matches any non-empty string excluding path separators (`/`).
- `slug`: Matches ASCII letters, digits, hyphens, and underscores.
- `uuid`: Matches a standardized UUID string.
- `path`: Matches any non-empty string, including path separators.

---

### Why you should never hardcode URLs: The `reverse()` Function

Imagine writing `/articles/42/` directly inside twenty different templates and redirect functions.

Six months later, marketing decides the URL structure must change to `/posts/42/`.

If URLs are hardcoded, you have to find and modify twenty files. If you miss one, users encounter broken links.

Django solves this by decoupling the **name** of the route from its **pattern**:

```python
from django.urls import reverse
from django.shortcuts import redirect

def update_article(request, article_id):
    # ... save article ...
    
    # BAD: Hardcoded redirect
    # return redirect(f"/articles/{article_id}/")

    # GOOD: Dynamic reversal by route name
    return redirect(reverse('article-detail', kwargs={'article_id': article_id}))
```

In templates, you use the `{% url %}` tag:

```html
<!-- Inside templates/articles/list.html -->
<a href="{% url 'article-detail' article_id=article.id %}">{{ article.title }}</a>
```

If the URL pattern changes in `urls.py`, every view and template updates automatically without touching any other file.

---

### App Namespacing

In larger projects with multiple apps, different apps often have views with the same name (for example, both `users` and `products` might have a `detail` view).

Namespaces prevent collision:

```python
# users/urls.py
app_name = 'users'
urlpatterns = [
    path('<int:user_id>/', views.detail, name='detail'),
]

# products/urls.py
app_name = 'products'
urlpatterns = [
    path('<int:product_id>/', views.detail, name='detail'),
]
```

Now you reverse unambiguously using `namespace:name`:

```python
reverse('users:detail', kwargs={'user_id': 1})
reverse('products:detail', kwargs={'product_id': 99})
```

---

# 2. Function-Based Views (FBVs) vs Class-Based Views (CBVs)

## Interview Question

**When would you choose a Function-Based View (FBV) over a Class-Based View (CBV), and how does a CBV actually work under the hood?**

---

### Answer

I don't believe one is universally superior to the other. They represent different engineering trade-offs.

```
FBV (Function-Based View)        CBV (Class-Based View)
─────────────────────────        ──────────────────────
Explicit, top-to-bottom logic    High code reuse via inheritance
No hidden inheritance magic      Eliminates repetitive CRUD boilerplate
Easy to read and debug           Steeper learning curve (MRO complexity)
Great for custom workflows       Great for standard resource CRUD
```

---

### Function-Based Views: Maximum Clarity

A function-based view is straightforward. A request comes in, you inspect the method, perform the logic, and return a response:

```python
from django.shortcuts import render, get_object_or_404, redirect
from .models import Order
from .forms import OrderForm

def order_view(request, order_id):
    order = get_object_or_404(Order, id=order_id)
    
    if request.method == 'POST':
        form = OrderForm(request.POST, instance=order)
        if form.is_valid():
            form.save()
            return redirect('orders:list')
    else:
        form = OrderForm(instance=order)
        
    return render(request, 'orders/order_form.html', {'form': form, 'order': order})
```

**Why engineers love FBVs:**
- There is zero magic. Anyone reading the code understands the entire execution flow in 30 seconds.
- Decorators like `@login_required` or `@require_http_methods(['GET', 'POST'])` make behavior crystal clear.

---

### Class-Based Views: Code Reusability

When building standard database-driven applications, 80% of views follow the exact same pattern:
- Fetch a list of objects.
- Display a detail page.
- Show a form, validate, and save.
- Delete an object.

Writing the same form handling and queries across fifty models becomes tedious. Django provides generic CBVs to eliminate this boilerplate:

```python
from django.views.generic import ListView, DetailView, CreateView
from .models import Product

class ProductListView(ListView):
    model = Product
    template_name = 'products/list.html'
    context_object_name = 'products'
    paginate_by = 20
```

With 5 lines of code, Django handles querying, pagination, template rendering, and 404 errors.

---

### How does a CBV work under the hood?

In Django's routing, `urlpatterns` expects a callable function, not a class:

```python
urlpatterns = [
    path('products/', ProductListView.as_view(), name='product-list'),
]
```

What does `as_view()` actually do?

`as_view()` is a class method that returns an internal closure function:

```python
# Conceptual implementation of View.as_view()
@classonlymethod
def as_view(cls, **initkwargs):
    def view(request, *args, **kwargs):
        # 1. Instantiate the class for this specific request
        self = cls(**initkwargs)
        self.setup(request, *args, **kwargs)
        
        # 2. Call dispatch() to route to get(), post(), put(), etc.
        return self.dispatch(request, *args, **kwargs)
        
    return view
```

When a request arrives:
1. Gunicorn/Django invokes the `view` closure.
2. The closure creates a **fresh instance** of `ProductListView`. (This is crucial: request state is never shared between concurrent requests!).
3. The instance calls `self.dispatch(request)`.
4. `dispatch()` inspects `request.method.lower()`. If the request is a `GET`, it calls `self.get()`. If it's a `POST`, it calls `self.post()`. If the method isn't implemented, it returns `HttpResponseNotAllowed` (405).

---

### The Danger with CBVs: Method Resolution Order (MRO) Pitfalls

Where do developers get into trouble with CBVs? **Multiple Inheritance.**

When combining Mixins with CBVs, the order of inheritance matters:

```python
# CORRECT
class ProtectedOrderView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    ...

# BUGGY / DANGEROUS
class BrokenOrderView(UpdateView, LoginRequiredMixin):
    ...
```

In Python, inheritance resolution flows from left to right according to the C3 Linearization algorithm (MRO).

Mixins must **always be placed to the left** of the base generic view. If `UpdateView` comes first, its `dispatch()` method executes before `LoginRequiredMixin`'s `dispatch()` gets a chance to verify authentication, leaving the view unprotected!

---

### Practical Rule of Thumb for Interviews

- If an endpoint represents a standard database resource (listing products, viewing an order, editing a profile), I use **CBVs** to avoid reinventing the wheel.
- If an endpoint has complex custom business logic (e.g. Stripe webhook handlers, multi-step checkout wizard, bespoke financial reporting), I use **FBVs** because readability and simplicity beat clever inheritance.

---

# 3. Django Templates & Template Inheritance

## Interview Question

**How does Django's template inheritance work, and how does Django protect against Cross-Site Scripting (XSS)?**

---

### Answer

Django's template engine is designed around the concept of **template inheritance**.

Instead of copying navigation bars, headers, footers, and scripts across dozens of HTML files, you define a single "skeleton" template called `base.html`:

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <title>{% block title %}My Shop{% endblock %}</title>
</head>
<body>
    <header>
        <nav>...</nav>
    </header>

    <main>
        {% block content %}
        <!-- Child templates inject their content here -->
        {% endblock %}
    </main>

    <footer>
        <p>&copy; 2026 My Company</p>
    </footer>
</body>
</html>
```

A child template inherits the skeleton using the `{% extends %}` tag and overrides specific blocks:

```html
<!-- templates/products/detail.html -->
{% extends "base.html" %}

{% block title %}{{ product.name }} - Details{% endblock %}

{% block content %}
    <h1>{{ product.name }}</h1>
    <p>{{ product.description }}</p>
    <span>\${{ product.price }}</span>
{% endblock %}
```

---

### Template Security: Automatic HTML Escaping

One of Django's most important security features is that all variable expansions in templates are **automatically HTML-escaped**.

Suppose a malicious user enters this as their username:

```html
<script>alert('Stealing session cookie: ' + document.cookie);</script>
```

When you render that username in a template:

```html
Hello, {{ user.username }}
```

Django does not output raw HTML tags. It translates dangerous characters into safe HTML entities:

```html
Hello, &lt;script&gt;alert(&#x27;Stealing session cookie: &#x27; + document.cookie);&lt;/script&gt;
```

The browser displays the string as plain text rather than executing it as JavaScript. This automatically neutralizes reflected and stored Cross-Site Scripting (XSS) attacks.

---

### The Danger: `|safe` and `mark_safe()`

Sometimes developers want to render raw HTML (for example, formatted blog posts created using a rich-text editor).

Django provides the `|safe` template filter and the `mark_safe()` Python utility:

```html
<!-- DANGEROUS IF UNVALIDATED -->
<div>{{ blog_post.content|safe }}</div>
```

If you apply `|safe` to raw, untrusted user input without sanitizing it first with a library like `nh3` or `bleach`, you create an immediate, critical XSS vulnerability.

In an interview, always emphasize: *never use `mark_safe()` on unsanitized user-generated content.*

---

# 4. Context Processors & Production Performance

## Interview Question

**What is a Django Context Processor, and how can a poorly written one silently kill your application's performance?**

---

### Answer

A context processor is a Python function that takes the current `request` object and returns a dictionary of data that will be **automatically injected into every template's rendering context**.

Django provides several default context processors:

```python
# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',  # provides {{ user }} and {{ perms }}
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

Because of `auth` context processor, you can write `{% if user.is_authenticated %}` in any template without manually passing `{'user': request.user}` from every single view.

---

### Writing a Custom Context Processor

Suppose your navigation bar needs to display the user's unread notification count on every page:

```python
# core/context_processors.py
def notification_context(request):
    if request.user.is_authenticated:
        count = request.user.notifications.filter(is_read=False).count()
        return {'unread_notifications_count': count}
    return {'unread_notifications_count': 0}
```

Add it to `settings.py`:

```python
'OPTIONS': {
    'context_processors': [
        ...
        'core.context_processors.notification_context',
    ],
}
```

Now `{{ unread_notifications_count }}` is available in every HTML template across the entire site.

---

### The Production Performance Trap

Here is the danger that senior engineers watch out for:

**Context processors execute on EVERY template render.**

Look at the code above:

```python
count = request.user.notifications.filter(is_read=False).count()
```

If an HTML page includes three sub-templates using `{% include %}`, or if an endpoint renders multiple template fragments, this context processor executes a `SELECT COUNT(*)` database query on **every single render**.

If you have 500 requests per second, your database is suddenly hit with 500 unnecessary queries per second just to count notifications on pages that might not even show them!

### How to fix it in production:

1. **Cache the result**: Cache the notification count in Redis with a short TTL (e.g. 60 seconds) or invalidate the cache key only when a new notification is created.
2. **Use Lazy Evaluation**: Return a callable or lazy object that only queries the database if the template variable is actually referenced on the rendered page.
3. **Use API Fetch**: On modern frontends, fetch notifications asynchronously using a lightweight REST endpoint via JavaScript after the main HTML renders.

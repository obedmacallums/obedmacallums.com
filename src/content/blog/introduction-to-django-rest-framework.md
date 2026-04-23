---
author: "Obed Macallums"
pubDatetime: 2026-04-22T10:00:00Z
title: "Introduction to Django REST Framework: Building Modern APIs in Python"
slug: "introduction-to-django-rest-framework"
featured: false
draft: false
tags:
  - python
  - django
  - django-rest-framework
  - api
  - rest
  - backend
  - web-development
description: "Learn how to build powerful, production-ready REST APIs with Django REST Framework. Covers serializers, views, authentication, permissions, pagination, and best practices."
---

**Django REST Framework (DRF)** is the de facto standard for building REST APIs in Django. It takes everything you already love about Django —the ORM, the admin, the security defaults— and adds a powerful, batteries-included toolkit for designing APIs. In this post we will go from zero to a working API, covering serializers, views, authentication, permissions, pagination, and the best practices that separate a toy project from a production-ready service.

## Table of Contents

## What is Django REST Framework?

**Django REST Framework** is a third-party package built on top of Django that provides a flexible and expressive way to build Web APIs. While Django itself can serve JSON responses, DRF abstracts the repetitive parts —parsing, validation, serialization, content negotiation, authentication— so you can focus on your domain logic.

Some of the reasons DRF has become the default choice in the Django ecosystem:

- **Browsable API**: an auto-generated HTML interface that lets you explore endpoints from the browser.
- **Serializers**: a declarative way to convert complex data types (like querysets) into JSON and back, with built-in validation.
- **Generic views and ViewSets**: CRUD endpoints in a handful of lines.
- **Authentication and permissions** out of the box, with pluggable backends.
- **First-class support for pagination, filtering, throttling, and versioning.**
- A huge ecosystem of extensions (JWT, OpenAPI, nested routers, etc.).

## Prerequisites

Before starting, you should be comfortable with:

- Python 3.10+
- Django fundamentals (models, URLs, views). If this is new, read [Introduction to Django](/posts/introduction-to-django-python-web-framework/) first.
- A basic understanding of HTTP and REST principles.

## Installation and Project Setup

Start by creating a virtual environment and installing Django and DRF:

```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

pip install django djangorestframework
```

Now bootstrap a new Django project and an app to hold our API:

```bash
django-admin startproject config .
python manage.py startapp books
```

Register both `rest_framework` and the new app in `config/settings.py`:

```python
# config/settings.py
INSTALLED_APPS = [
    # ... default apps
    "rest_framework",
    "books",
]
```

Run the initial migrations to make sure everything is wired up:

```bash
python manage.py migrate
```

## Defining a Model

Let's model a simple `Book` resource. Add the following to `books/models.py`:

```python
# books/models.py
from django.db import models


class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    published_year = models.PositiveIntegerField()
    isbn = models.CharField(max_length=13, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self) -> str:
        return f"{self.title} ({self.author})"
```

Create and apply the migration:

```bash
python manage.py makemigrations
python manage.py migrate
```

## Serializers: The Heart of DRF

A **serializer** translates between complex Python objects (like Django model instances) and primitive data types that can be rendered as JSON. It also handles validation of incoming data.

The most common base class is `ModelSerializer`, which inspects the model to generate fields automatically:

```python
# books/serializers.py
from rest_framework import serializers
from .models import Book


class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ["id", "title", "author", "published_year", "isbn", "created_at"]
        read_only_fields = ["id", "created_at"]
```

### Custom Validation

You often need domain-specific rules. DRF offers field-level and object-level validation:

```python
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ["id", "title", "author", "published_year", "isbn", "created_at"]

    def validate_isbn(self, value: str) -> str:
        if len(value) not in (10, 13):
            raise serializers.ValidationError("ISBN must be 10 or 13 characters.")
        return value

    def validate(self, data: dict) -> dict:
        if data["published_year"] > 2026:
            raise serializers.ValidationError("published_year cannot be in the future.")
        return data
```

## Views and ViewSets

DRF offers several layers of abstraction for writing views. From lowest to highest:

1. **`APIView`**: request/response style, similar to Django's class-based views.
2. **Generic views** (`ListCreateAPIView`, `RetrieveUpdateDestroyAPIView`, etc.): CRUD on top of a queryset and serializer.
3. **ViewSets**: group related actions (list, create, retrieve, update, destroy) into a single class that plugs into a router.

For a typical CRUD resource, a `ModelViewSet` is the shortest path:

```python
# books/views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer


class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

That single class exposes `GET /books/`, `POST /books/`, `GET /books/{id}/`, `PUT /books/{id}/`, `PATCH /books/{id}/`, and `DELETE /books/{id}/`.

## Routers and URL Configuration

Routers wire ViewSets to URLs automatically:

```python
# books/urls.py
from rest_framework.routers import DefaultRouter
from .views import BookViewSet

router = DefaultRouter()
router.register(r"books", BookViewSet, basename="book")

urlpatterns = router.urls
```

Then include the app URLs in the project:

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", include("books.urls")),
]
```

Start the development server and you will have a fully browsable API at `http://127.0.0.1:8000/api/books/`:

```bash
python manage.py runserver
```

## Authentication

DRF supports multiple authentication schemes. The most common in production are **Token Authentication** and **JWT**.

### Token Authentication

Enable token auth globally in settings:

```python
# config/settings.py
INSTALLED_APPS += ["rest_framework.authtoken"]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ],
}
```

Run `python manage.py migrate` to create the token table. You can then expose an endpoint to obtain tokens:

```python
# config/urls.py
from rest_framework.authtoken.views import obtain_auth_token

urlpatterns += [
    path("api/auth/token/", obtain_auth_token),
]
```

Clients send the token in the `Authorization` header:

```http
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

### JWT Authentication

For stateless, scalable APIs, JWT is usually preferred. The `djangorestframework-simplejwt` package is the community standard:

```bash
pip install djangorestframework-simplejwt
```

```python
# config/settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}
```

```python
# config/urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns += [
    path("api/auth/jwt/", TokenObtainPairView.as_view()),
    path("api/auth/jwt/refresh/", TokenRefreshView.as_view()),
]
```

## Permissions

Authentication answers _who are you?_; permissions answer _what are you allowed to do?_. DRF ships with common classes:

- `AllowAny`
- `IsAuthenticated`
- `IsAdminUser`
- `IsAuthenticatedOrReadOnly`
- `DjangoModelPermissions` (ties into Django's built-in permission system)

Set a default globally and override per view when needed:

```python
# config/settings.py
REST_FRAMEWORK = {
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

```python
# books/views.py
from rest_framework.permissions import IsAuthenticatedOrReadOnly


class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
```

For fine-grained rules (for example, only the owner can edit), write a custom permission:

```python
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj) -> bool:
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.owner == request.user
```

## Pagination, Filtering, and Search

Production APIs almost always need pagination. Enable it globally:

```python
# config/settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
```

For filtering and search, install and configure `django-filter`:

```bash
pip install django-filter
```

```python
# config/settings.py
INSTALLED_APPS += ["django_filters"]

REST_FRAMEWORK = {
    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ],
}
```

```python
# books/views.py
class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filterset_fields = ["author", "published_year"]
    search_fields = ["title", "author"]
    ordering_fields = ["published_year", "created_at"]
```

Clients can now hit endpoints like `/api/books/?author=Tolkien&search=ring&ordering=-published_year`.

## Testing Your API

DRF provides an `APIClient` that makes writing tests straightforward:

```python
# books/tests.py
from django.contrib.auth import get_user_model
from rest_framework.test import APITestCase
from rest_framework import status
from .models import Book


class BookAPITests(APITestCase):
    def setUp(self) -> None:
        self.user = get_user_model().objects.create_user(
            username="reader", password="secret123"
        )
        self.client.force_authenticate(user=self.user)

    def test_create_book(self) -> None:
        payload = {
            "title": "The Hobbit",
            "author": "J.R.R. Tolkien",
            "published_year": 1937,
            "isbn": "9780547928227",
        }
        response = self.client.post("/api/books/", payload, format="json")
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Book.objects.count(), 1)
```

Run the test suite with:

```bash
python manage.py test
```

## Best Practices

A few habits that pay dividends as your API grows:

- **Version your API from day one** (`/api/v1/books/`). Breaking changes are inevitable.
- **Keep serializers thin**: move business logic into services or model methods.
- **Use `select_related` and `prefetch_related`** in your querysets to avoid N+1 queries.
- **Document with OpenAPI**: tools like [`drf-spectacular`](https://drf-spectacular.readthedocs.io/) generate interactive docs automatically.
- **Throttle public endpoints** to prevent abuse (`DEFAULT_THROTTLE_CLASSES`).
- **Never return Django model instances directly**; always go through a serializer.
- **Separate read and write serializers** when the shapes diverge.
- **Write integration tests**, not just unit tests —the wiring between URL, view, serializer, and permission is where most bugs hide.

## When to Use DRF (and When Not To)

DRF is an excellent choice when:

- You already have a Django codebase and want to expose parts of it as an API.
- You need an admin, an ORM, and an API in the same project.
- You value convention-over-configuration and a rich ecosystem.

It may be overkill if:

- You only need a tiny service with a couple of endpoints —a microframework like FastAPI or Flask may be lighter. See [FastAPI vs Django](/posts/building-apis-fastapi-vs-django-comprehensive-comparison/) for a deeper comparison.
- Your workload is heavily async and streaming-oriented; DRF's async support has improved but is still less mature than FastAPI's.

## Conclusion

**Django REST Framework** remains the most productive way to build REST APIs in the Python ecosystem. It layers serialization, validation, authentication, permissions, and pagination on top of Django's solid foundations, and it scales from weekend projects to services handling millions of requests. Start with `ModelViewSet` and routers to get moving quickly, then reach for custom permissions, filters, and throttles as your API matures. Pair it with `drf-spectacular` for documentation and `djangorestframework-simplejwt` for stateless authentication, and you have a stack that will serve you well for years.

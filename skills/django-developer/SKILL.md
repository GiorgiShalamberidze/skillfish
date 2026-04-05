---
name: django-developer
description: Django development: class-based views, ORM optimization, Django REST Framework, Celery async tasks, deployment patterns, and async Django.
---

# Django Developer

Expert-level Django development guidance covering the full stack: model design and ORM optimization, class-based views, Django REST Framework APIs, forms and validation, Celery async task processing, async Django (ASGI/Channels), testing with pytest, and production deployment patterns.

## When to Use

- Building or extending a Django application
- Designing models, managers, or complex querysets
- Creating REST APIs with Django REST Framework
- Integrating Celery for background and periodic tasks
- Migrating to async Django views or ASGI
- Writing tests for Django views, models, or APIs
- Preparing a Django project for production deployment

## Table of Contents

1. [Models and ORM](#1-models-and-orm)
2. [Views and URLs](#2-views-and-urls)
3. [Django REST Framework](#3-django-rest-framework)
4. [Forms and Validation](#4-forms-and-validation)
5. [Celery and Async Tasks](#5-celery-and-async-tasks)
6. [Async Django](#6-async-django)
7. [Testing](#7-testing)
8. [Deployment](#8-deployment)

---

## 1. Models and ORM

### Model Design

```python
from django.db import models
from django.utils import timezone


class TimestampMixin(models.Model):
    created_at = models.DateTimeField(default=timezone.now, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Article(TimestampMixin):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PUBLISHED = "published", "Published"
        ARCHIVED = "archived", "Archived"

    title = models.CharField(max_length=255)
    slug = models.SlugField(max_length=255, unique=True)
    author = models.ForeignKey(
        "auth.User", on_delete=models.CASCADE, related_name="articles",
    )
    body = models.TextField()
    status = models.CharField(
        max_length=20, choices=Status.choices, default=Status.DRAFT, db_index=True,
    )
    published_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        ordering = ["-published_at"]
        indexes = [models.Index(fields=["status", "-published_at"])]

    def __str__(self):
        return self.title
```

### Custom Managers and QuerySets

Keep query logic out of views.

```python
class ArticleQuerySet(models.QuerySet):
    def published(self):
        return self.filter(status=Article.Status.PUBLISHED)

    def by_author(self, user):
        return self.filter(author=user)

    def with_comment_count(self):
        return self.annotate(comment_count=models.Count("comments"))


class ArticleManager(models.Manager):
    def get_queryset(self):
        return ArticleQuerySet(self.model, using=self._db)

    def published(self):
        return self.get_queryset().published()
```

### select_related and prefetch_related

```python
# BAD — N+1: one query per article to fetch author
for a in Article.objects.all():
    print(a.author.username)

# GOOD — single JOIN
articles = Article.objects.select_related("author").all()

# prefetch_related for reverse/M2M relations
articles = Article.objects.prefetch_related("comments", "tags").select_related("author")
```

### F Objects, Q Objects, and Annotations

```python
from django.db.models import F, Q, Count, Avg

# F expressions — avoid race conditions
Article.objects.filter(pk=1).update(view_count=F("view_count") + 1)

# Q objects — complex OR / NOT filters
Article.objects.filter(
    Q(status="published") | Q(author=request.user),
    ~Q(title__startswith="[DRAFT]"),
)

# Aggregation
stats = Article.objects.aggregate(total=Count("id"), avg_comments=Avg("comment_count"))

# Annotation
articles = (
    Article.objects
    .annotate(num_comments=Count("comments"))
    .filter(num_comments__gte=5)
)
```

### Anti-Patterns

```python
# ANTI-PATTERN: filtering in Python instead of the database
bad = [a for a in Article.objects.all() if a.status == "published"]
good = Article.objects.filter(status="published")

# ANTI-PATTERN: len() instead of .count()
bad_count = len(Article.objects.all())
good_count = Article.objects.count()
```

---

## 2. Views and URLs

### Class-Based Views vs Function-Based Views

```python
from django.http import JsonResponse
from django.views.decorators.http import require_GET

@require_GET
def health_check(request):
    return JsonResponse({"status": "ok"})
```

```python
from django.views.generic import ListView, DetailView
from django.contrib.auth.mixins import LoginRequiredMixin


class ArticleListView(LoginRequiredMixin, ListView):
    model = Article
    template_name = "articles/list.html"
    context_object_name = "articles"
    paginate_by = 25

    def get_queryset(self):
        qs = super().get_queryset().published().select_related("author")
        tag = self.request.GET.get("tag")
        return qs.filter(tags__slug=tag) if tag else qs


class ArticleDetailView(DetailView):
    model = Article
    template_name = "articles/detail.html"

    def get_queryset(self):
        return super().get_queryset().select_related("author").prefetch_related("comments")
```

### Custom Mixins

```python
class OwnerRequiredMixin:
    def dispatch(self, request, *args, **kwargs):
        obj = self.get_object()
        if obj.author != request.user:
            raise PermissionDenied("You do not own this object.")
        return super().dispatch(request, *args, **kwargs)
```

### URL Configuration

```python
from django.urls import path, include

app_name = "articles"
urlpatterns = [
    path("", ArticleListView.as_view(), name="list"),
    path("<slug:slug>/", ArticleDetailView.as_view(), name="detail"),
    path("api/v1/", include("articles.api.urls")),
]
```

### Middleware

```python
import time, logging

logger = logging.getLogger("django.request")

class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        duration_ms = (time.monotonic() - start) * 1000
        logger.info("method=%s path=%s status=%s duration=%.1fms",
                     request.method, request.path, response.status_code, duration_ms)
        return response
```

### Anti-Patterns

```python
# ANTI-PATTERN: business logic in the view
class BadView(CreateView):
    def form_valid(self, form):
        obj = form.save()
        send_mail(...)           # email logic in view
        update_analytics(obj)    # analytics in view
        return redirect(obj)

# FIX: delegate side effects to a service layer
class GoodView(CreateView):
    def form_valid(self, form):
        obj = form.save()
        ArticleService.on_publish(obj)
        return redirect(obj)
```

---

## 3. Django REST Framework

### Serializers

```python
from rest_framework import serializers


class ArticleSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source="author.get_full_name", read_only=True)
    comment_count = serializers.IntegerField(read_only=True)

    class Meta:
        model = Article
        fields = ["id", "title", "slug", "body", "status",
                  "author", "author_name", "comment_count",
                  "published_at", "created_at"]
        read_only_fields = ["slug", "created_at"]

    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters.")
        return value


class ArticleCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ["title", "body", "status"]

    def create(self, validated_data):
        validated_data["author"] = self.context["request"].user
        validated_data["slug"] = slugify(validated_data["title"])
        return super().create(validated_data)
```

### ViewSets and Routers

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response


class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.select_related("author").annotate(
        comment_count=Count("comments"),
    )
    permission_classes = [IsAuthenticatedOrReadOnly]
    lookup_field = "slug"
    filterset_fields = ["status", "author"]
    search_fields = ["title", "body"]
    ordering_fields = ["published_at", "created_at"]
    ordering = ["-published_at"]

    def get_serializer_class(self):
        if self.action in ("create", "update", "partial_update"):
            return ArticleCreateSerializer
        return ArticleSerializer

    @action(detail=True, methods=["post"])
    def publish(self, request, slug=None):
        article = self.get_object()
        article.status = Article.Status.PUBLISHED
        article.published_at = timezone.now()
        article.save(update_fields=["status", "published_at"])
        return Response(ArticleSerializer(article).data)
```

```python
# api/urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register("articles", ArticleViewSet, basename="article")
urlpatterns = router.urls
```

### Permissions

```python
from rest_framework.permissions import BasePermission, SAFE_METHODS

class IsAuthorOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS:
            return True
        return obj.author == request.user
```

### Pagination and Throttling

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.CursorPagination",
    "PAGE_SIZE": 25,
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {"anon": "100/hour", "user": "1000/hour"},
    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ],
}
```

### Anti-Patterns

```python
# ANTI-PATTERN: one serializer for everything, fields="__all__"
class BadSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = "__all__"  # exposes every field including internal ones

# FIX: separate read/write serializers with explicit field lists
```

---

## 4. Forms and Validation

### ModelForm

```python
from django import forms

class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["title", "body", "status"]
        widgets = {"body": forms.Textarea(attrs={"rows": 12})}

    def clean_title(self):
        title = self.cleaned_data["title"]
        if Article.objects.filter(title__iexact=title).exclude(pk=self.instance.pk).exists():
            raise forms.ValidationError("An article with this title already exists.")
        return title
```

### Formsets

```python
from django.forms import inlineformset_factory

ImageFormSet = inlineformset_factory(
    Article, ArticleImage,
    fields=["image", "caption"], extra=3, max_num=10, can_delete=True,
)

def article_edit(request, pk):
    article = get_object_or_404(Article, pk=pk)
    if request.method == "POST":
        form = ArticleForm(request.POST, instance=article)
        formset = ImageFormSet(request.POST, request.FILES, instance=article)
        if form.is_valid() and formset.is_valid():
            form.save()
            formset.save()
            return redirect(article)
    else:
        form = ArticleForm(instance=article)
        formset = ImageFormSet(instance=article)
    return render(request, "articles/edit.html", {"form": form, "formset": formset})
```

### Custom Validators

```python
from django.core.exceptions import ValidationError

def validate_file_size(value):
    max_size = 5 * 1024 * 1024
    if value.size > max_size:
        raise ValidationError(f"File size must be under {max_size // (1024*1024)} MB.")

class ArticleImage(models.Model):
    article = models.ForeignKey(Article, on_delete=models.CASCADE, related_name="images")
    image = models.ImageField(upload_to="articles/", validators=[validate_file_size])
    caption = models.CharField(max_length=200)
```

### HTMX Integration

```python
class ArticleListView(ListView):
    model = Article
    paginate_by = 20

    def get_template_names(self):
        if self.request.headers.get("HX-Request"):
            return ["articles/_list_partial.html"]
        return ["articles/list.html"]
```

```html
<div id="article-list" hx-get="?page={{ page_obj.next_page_number }}"
     hx-trigger="revealed" hx-swap="afterend">
  {% include "articles/_list_partial.html" %}
</div>
```

### Anti-Patterns

```python
# ANTI-PATTERN: validation in the view
def bad_create(request):
    title = request.POST["title"]
    if len(title) < 5:
        return HttpResponse("Too short", status=400)

# FIX: use form.is_valid()
def good_create(request):
    form = ArticleForm(request.POST)
    if form.is_valid():
        form.save()
        return redirect("articles:list")
    return render(request, "articles/create.html", {"form": form})
```

---

## 5. Celery and Async Tasks

### Setup

```python
# proj/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "proj.settings")
app = Celery("proj")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

```python
# proj/__init__.py
from .celery import app as celery_app
__all__ = ["celery_app"]
```

```python
# settings.py
CELERY_BROKER_URL = os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/0")
CELERY_RESULT_BACKEND = "django-db"
CELERY_ACCEPT_CONTENT = ["json"]
CELERY_TASK_SERIALIZER = "json"
CELERY_TASK_TIME_LIMIT = 300
CELERY_TASK_SOFT_TIME_LIMIT = 240
```

### Task Patterns

```python
from celery import shared_task
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def send_notification_email(self, user_id, article_id):
    try:
        user = User.objects.get(pk=user_id)
        article = Article.objects.get(pk=article_id)
        send_mail(
            subject=f"New article: {article.title}",
            message=article.body[:200],
            from_email="noreply@example.com",
            recipient_list=[user.email],
        )
    except (User.DoesNotExist, Article.DoesNotExist):
        raise  # don't retry — object won't appear later
    except Exception as exc:
        raise self.retry(exc=exc)
```

### Task Chaining and Groups

```python
from celery import chain, group, chord

# Sequential pipeline
chain(extract_data.s(url), transform_data.s(), load_data.s("warehouse")).apply_async()

# Parallel fan-out
group(generate_thumbnail.s(img_id) for img_id in image_ids).apply_async()

# Fan-out then callback
chord(
    group(process_chunk.s(chunk) for chunk in chunks),
    merge_results.s(),
).apply_async()
```

### Periodic Tasks (Celery Beat)

```python
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "archive-stale-drafts": {
        "task": "articles.tasks.archive_stale_drafts",
        "schedule": crontab(hour=3, minute=0),
    },
    "refresh-sitemap-cache": {
        "task": "seo.tasks.refresh_sitemap",
        "schedule": crontab(minute="*/30"),
    },
}
```

### Error Handling with Backoff

```python
@shared_task(
    bind=True,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
    retry_backoff_max=600,
    retry_jitter=True,
    max_retries=5,
)
def call_external_api(self, payload):
    response = requests.post("https://api.example.com/ingest", json=payload, timeout=10)
    response.raise_for_status()
    return response.json()
```

### Anti-Patterns

```python
# ANTI-PATTERN: passing model instances to tasks (can't serialize)
send_notification_email.delay(user, article)
# FIX: pass IDs, re-fetch inside the task
send_notification_email.delay(user.id, article.id)

# ANTI-PATTERN: no time limit — stuck task blocks the worker forever
@shared_task
def unguarded():
    while True:
        process()

# FIX: always set time limits
@shared_task(time_limit=120, soft_time_limit=100)
def safe_task():
    process()
```

---

## 6. Async Django

### Async Views (Django 4.1+)

```python
import httpx
from django.http import JsonResponse

async def fetch_weather(request, city):
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"https://api.weather.example.com/{city}", timeout=5.0)
        resp.raise_for_status()
    return JsonResponse(resp.json())
```

### Async ORM (Django 5+)

```python
async def article_list(request):
    articles = [
        article async for article in
        Article.objects.filter(status="published").select_related("author")[:25]
    ]
    return JsonResponse({"articles": [{"title": a.title, "author": a.author.username} for a in articles]})

async def article_detail(request, slug):
    try:
        article = await Article.objects.select_related("author").aget(slug=slug)
    except Article.DoesNotExist:
        return JsonResponse({"error": "Not found"}, status=404)
    return JsonResponse({"title": article.title, "body": article.body})
```

### ASGI Configuration

```python
# proj/asgi.py
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "proj.settings")
application = get_asgi_application()
```

### Async Middleware

```python
class AsyncTimingMiddleware:
    async_capable = True
    sync_capable = False

    def __init__(self, get_response):
        self.get_response = get_response

    async def __call__(self, request):
        start = time.monotonic()
        response = await self.get_response(request)
        logger.info("path=%s duration=%.1fms", request.path, (time.monotonic() - start) * 1000)
        return response
```

### Django Channels (WebSockets)

```python
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class NotificationConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.user = self.scope["user"]
        if self.user.is_anonymous:
            await self.close()
            return
        self.group_name = f"user_{self.user.id}"
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        if hasattr(self, "group_name"):
            await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def notification_message(self, event):
        await self.send_json(event["data"])
```

### Anti-Patterns

```python
# ANTI-PATTERN: calling sync ORM from async view (raises SynchronousOnlyOperation)
async def bad_view(request):
    articles = Article.objects.all()

# FIX (Django 4.x): wrap with sync_to_async
from asgiref.sync import sync_to_async

async def ok_view(request):
    articles = await sync_to_async(list)(Article.objects.filter(status="published"))
    return JsonResponse({"count": len(articles)})

# FIX (Django 5+): use native async ORM
async def best_view(request):
    count = await Article.objects.filter(status="published").acount()
    return JsonResponse({"count": count})
```

---

## 7. Testing

### pytest-django Setup

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = proj.settings
python_files = tests.py test_*.py *_tests.py
addopts = --reuse-db --tb=short -q
```

### Factory Boy Factories

```python
import factory
from factory.django import DjangoModelFactory

class UserFactory(DjangoModelFactory):
    class Meta:
        model = "auth.User"
        skip_postgeneration_save = True

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.LazyAttribute(lambda o: f"{o.username}@example.com")
    password = factory.PostGenerationMethodCall("set_password", "testpass123")


class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Faker("sentence", nb_words=5)
    slug = factory.LazyAttribute(lambda o: slugify(o.title))
    author = factory.SubFactory(UserFactory)
    body = factory.Faker("paragraph", nb_sentences=10)
    status = Article.Status.DRAFT
```

### Model and View Tests

```python
import pytest
from django.test import RequestFactory

pytestmark = pytest.mark.django_db

class TestArticleModel:
    def test_str_returns_title(self):
        article = ArticleFactory(title="My Article")
        assert str(article) == "My Article"

    def test_published_manager(self):
        ArticleFactory(status=Article.Status.PUBLISHED, published_at=timezone.now())
        ArticleFactory(status=Article.Status.DRAFT)
        assert Article.objects.published().count() == 1

class TestArticleListView:
    def setup_method(self):
        self.factory = RequestFactory()
        self.user = UserFactory()

    def test_list_returns_200(self):
        ArticleFactory.create_batch(3, status=Article.Status.PUBLISHED, published_at=timezone.now())
        request = self.factory.get("/articles/")
        request.user = self.user
        response = ArticleListView.as_view()(request)
        assert response.status_code == 200
```

### DRF API Tests

```python
from rest_framework.test import APIClient

pytestmark = pytest.mark.django_db

class TestArticleAPI:
    def setup_method(self):
        self.client = APIClient()
        self.user = UserFactory()

    def test_list_articles(self):
        ArticleFactory.create_batch(3, status=Article.Status.PUBLISHED, published_at=timezone.now())
        response = self.client.get("/api/v1/articles/")
        assert response.status_code == 200
        assert len(response.data["results"]) == 3

    def test_create_requires_auth(self):
        response = self.client.post("/api/v1/articles/", {"title": "New", "body": "Content"})
        assert response.status_code == 401

    def test_create_article(self):
        self.client.force_authenticate(user=self.user)
        payload = {"title": "New Article", "body": "Content here", "status": "draft"}
        response = self.client.post("/api/v1/articles/", payload, format="json")
        assert response.status_code == 201
        assert Article.objects.filter(title="New Article").exists()

    def test_publish_action(self):
        article = ArticleFactory(author=self.user, status=Article.Status.DRAFT)
        self.client.force_authenticate(user=self.user)
        response = self.client.post(f"/api/v1/articles/{article.slug}/publish/")
        assert response.status_code == 200
        article.refresh_from_db()
        assert article.status == Article.Status.PUBLISHED
```

### Coverage

```bash
pytest --cov=articles --cov-report=html --cov-report=term-missing
pytest --cov=articles --cov-fail-under=85  # enforce in CI
```

### Anti-Patterns

```python
# ANTI-PATTERN: depending on fixture files or live data
def test_bad():
    article = Article.objects.get(pk=1)  # fragile

# FIX: use factory_boy for isolated test data
def test_good():
    article = ArticleFactory(title="Expected Title")
    assert str(article) == "Expected Title"
```

---

## 8. Deployment

### Gunicorn / Uvicorn

```python
# gunicorn.conf.py
import multiprocessing

bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "gthread"
threads = 4
timeout = 30
max_requests = 1000
max_requests_jitter = 50
accesslog = "-"
errorlog = "-"
```

```bash
gunicorn proj.wsgi:application -c gunicorn.conf.py            # WSGI
uvicorn proj.asgi:application --host 0.0.0.0 --port 8000 --workers 4  # ASGI
```

### Docker

```dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends libpq-dev gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements/production.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN python manage.py collectstatic --noinput
EXPOSE 8000
CMD ["gunicorn", "proj.wsgi:application", "-c", "gunicorn.conf.py"]
```

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }

  celery-worker:
    build: .
    command: celery -A proj worker -l info --concurrency=4
    env_file: .env
    depends_on: [db, redis]

  celery-beat:
    build: .
    command: celery -A proj beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    env_file: .env
    depends_on: [db, redis]

  db:
    image: postgres:16
    volumes: [pgdata:/var/lib/postgresql/data]
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

### Static Files with WhiteNoise

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",  # right after SecurityMiddleware
    # ...
]
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
STORAGES = {
    "staticfiles": {"BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"},
}
```

### Environment Configuration

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.environ.get("DJANGO_DEBUG", "false").lower() == "true"
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS", "localhost").split(",")

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ.get("DB_NAME", "app"),
        "USER": os.environ.get("DB_USER", "app"),
        "PASSWORD": os.environ.get("DB_PASSWORD", ""),
        "HOST": os.environ.get("DB_HOST", "localhost"),
        "PORT": os.environ.get("DB_PORT", "5432"),
        "CONN_MAX_AGE": 600,
        "CONN_HEALTH_CHECKS": True,
    },
}
```

### Migrations in CI

```bash
#!/usr/bin/env bash
set -euo pipefail
python manage.py migrate --noinput
python manage.py collectstatic --noinput
python manage.py makemigrations --check --dry-run
python manage.py check --deploy
```

### Health Checks

```python
from django.db import connection
from django.http import JsonResponse
from django.core.cache import cache

def health(request):
    return JsonResponse({"status": "ok"})

def ready(request):
    errors = []
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
    except Exception as exc:
        errors.append(f"db: {exc}")
    try:
        cache.set("_health", "1", timeout=5)
        if cache.get("_health") != "1":
            errors.append("cache: read-back failed")
    except Exception as exc:
        errors.append(f"cache: {exc}")
    if errors:
        return JsonResponse({"status": "unhealthy", "errors": errors}, status=503)
    return JsonResponse({"status": "ready"})
```

```python
# urls.py — mount outside authentication
urlpatterns = [
    path("healthz/", health, name="health"),
    path("readyz/", ready, name="ready"),
]
```

### Production Security Settings

```python
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

### Anti-Patterns

```python
# ANTI-PATTERN: DEBUG=True or hardcoded secrets in production
DEBUG = True
SECRET_KEY = "django-insecure-abc123"

# FIX: always read from environment
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.environ.get("DJANGO_DEBUG", "false").lower() == "true"

# ANTI-PATTERN: running migrate during application startup (race condition with replicas)
# FIX: run migrations as a separate one-shot job in CI/CD
```

---

## Output Format

When generating Django code, follow this structure:

1. **Models first** — data structures, managers, signals
2. **Serializers / Forms** — validation and data transformation
3. **Views / ViewSets** — thin controllers that delegate to services
4. **URLs** — route registration
5. **Tasks** — async work outside request/response cycle
6. **Tests** — critical paths, edge cases, permissions
7. **Configuration** — settings, Docker, CI/CD

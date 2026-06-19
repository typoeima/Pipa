📘 ГОТОВЫЙ ПРОЕКТ: Управление клиентами (Clients)

Вариант: Клиенты (clients)

---

🚀 ПОЛНЫЙ КОД ПРОЕКТА

---

ШАГ 1: Файлы настроек

📁 .env (в корне проекта)

```env
DEBUG=False
SECRET_KEY=django-insecure-your-secret-key-here-change-it
```

📁 requirements.txt

```txt
Django==4.2.7
python-dotenv==1.0.0
whitenoise==6.6.0
```

---

📁 exam_project/settings.py

```python
import os
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv('SECRET_KEY')
DEBUG = os.getenv('DEBUG', 'False') == 'True'

ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'clients',  # наше приложение
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'clients.middleware.MetricsMiddleware',
]

ROOT_URLCONF = 'exam_project.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
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

WSGI_APPLICATION = 'exam_project.wsgi.application'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]

LANGUAGE_CODE = 'ru-ru'
TIME_ZONE = 'Europe/Moscow'
USE_I18N = True
USE_TZ = True

STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

---

📁 exam_project/urls.py

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('clients.urls')),
]
```

---

ШАГ 2: Модель и валидация

📁 clients/models.py

```python
from django.db import models
from django.core.exceptions import ValidationError
from django.utils import timezone
import re

class Client(models.Model):
    full_name = models.CharField(max_length=200, verbose_name='Полное имя')
    email = models.EmailField(max_length=100, unique=True, verbose_name='Email')
    phone = models.CharField(max_length=20, blank=True, null=True, verbose_name='Телефон')
    registration_date = models.DateField(default=timezone.now, verbose_name='Дата регистрации')
    created_at = models.DateTimeField(auto_now_add=True, verbose_name='Дата создания')

    def clean(self):
        errors = {}
        
        # 1. full_name не может быть пустым
        if not self.full_name or self.full_name.strip() == '':
            errors['full_name'] = 'Полное имя не может быть пустым'
        
        # 2. email должен содержать @ и .
        if self.email:
            if '@' not in self.email:
                errors['email'] = 'Email должен содержать символ @'
            elif '.' not in self.email:
                errors['email'] = 'Email должен содержать точку (.)'
        
        # 3. email уникальный (дополнительная проверка)
        if self.email:
            existing = Client.objects.filter(email=self.email)
            if self.pk:
                existing = existing.exclude(pk=self.pk)
            if existing.exists():
                errors['email'] = 'Клиент с таким email уже существует'
        
        # 4. phone уникальный, если указан
        if self.phone and self.phone.strip() != '':
            existing = Client.objects.filter(phone=self.phone)
            if self.pk:
                existing = existing.exclude(pk=self.pk)
            if existing.exists():
                errors['phone'] = 'Клиент с таким телефоном уже существует'
        
        # 5. registration_date не может быть в будущем
        if self.registration_date and self.registration_date > timezone.now().date():
            errors['registration_date'] = 'Дата регистрации не может быть в будущем'
        
        if errors:
            raise ValidationError(errors)

    def save(self, *args, **kwargs):
        # Если телефон пустая строка, сохраняем как None
        if self.phone == '':
            self.phone = None
        self.full_clean()
        super().save(*args, **kwargs)

    def __str__(self):
        return self.full_name

    class Meta:
        verbose_name = 'Клиент'
        verbose_name_plural = 'Клиенты'
        ordering = ['-created_at']
```

---

ШАГ 3: Админка

📁 clients/admin.py

```python
from django.contrib import admin
from .models import Client

@admin.register(Client)
class ClientAdmin(admin.ModelAdmin):
    list_display = ['id', 'full_name', 'email', 'phone', 'registration_date', 'created_at']
    list_filter = ['registration_date']
    search_fields = ['full_name', 'email', 'phone']
    ordering = ['-created_at']
```

---

ШАГ 4: Форма

📁 clients/forms.py

```python
from django import forms
from .models import Client
from django.utils import timezone

class ClientForm(forms.ModelForm):
    class Meta:
        model = Client
        fields = ['full_name', 'email', 'phone', 'registration_date']
        widgets = {
            'full_name': forms.TextInput(attrs={'class': 'form-control'}),
            'email': forms.EmailInput(attrs={'class': 'form-control'}),
            'phone': forms.TextInput(attrs={'class': 'form-control'}),
            'registration_date': forms.DateInput(attrs={'class': 'form-control', 'type': 'date'}),
        }
        labels = {
            'full_name': 'Полное имя',
            'email': 'Email',
            'phone': 'Телефон',
            'registration_date': 'Дата регистрации',
        }
```

---

ШАГ 5: Представления

📁 clients/views.py

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.http import JsonResponse
from django.contrib import messages
from .models import Client
from .forms import ClientForm

def client_list(request):
    clients = Client.objects.all().order_by('-created_at')
    return render(request, 'clients/client_list.html', {'clients': clients})

def client_create(request):
    if request.method == 'POST':
        form = ClientForm(request.POST)
        if form.is_valid():
            form.save()
            messages.success(request, 'Клиент успешно создан!')
            return redirect('client_list')
        else:
            for field, errors in form.errors.items():
                for error in errors:
                    messages.error(request, f'{error}')
    else:
        form = ClientForm()
    return render(request, 'clients/client_form.html', {'form': form, 'client': None})

def client_update(request, pk):
    client = get_object_or_404(Client, pk=pk)
    if request.method == 'POST':
        form = ClientForm(request.POST, instance=client)
        if form.is_valid():
            form.save()
            messages.success(request, 'Клиент успешно обновлён!')
            return redirect('client_list')
        else:
            for field, errors in form.errors.items():
                for error in errors:
                    messages.error(request, f'{error}')
    else:
        form = ClientForm(instance=client)
    return render(request, 'clients/client_form.html', {'form': form, 'client': client})

def client_delete(request, pk):
    client = get_object_or_404(Client, pk=pk)
    if request.method == 'POST':
        client.delete()
        messages.success(request, 'Клиент успешно удалён!')
        return redirect('client_list')
    return render(request, 'clients/client_confirm_delete.html', {'client': client})

def health_check(request):
    return JsonResponse({'status': 'ok'})
```

---

ШАГ 6: URL-маршруты

📁 clients/urls.py

```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.client_list, name='client_list'),
    path('create/', views.client_create, name='client_create'),
    path('<int:pk>/update/', views.client_update, name='client_update'),
    path('<int:pk>/delete/', views.client_delete, name='client_delete'),
    path('ping/', views.health_check, name='ping'),
]
```

---

ШАГ 7: Middleware

📁 clients/middleware.py

```python
class MetricsMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        self.total_requests = 0
        self.status_2xx = 0
        self.status_4xx = 0
        self.status_5xx = 0
        self.counter = 0

    def __call__(self, request):
        response = self.get_response(request)
        
        self.total_requests += 1
        self.counter += 1
        
        if 200 <= response.status_code < 300:
            self.status_2xx += 1
        elif 400 <= response.status_code < 500:
            self.status_4xx += 1
        elif 500 <= response.status_code < 600:
            self.status_5xx += 1
        
        if self.counter >= 5:
            print("=" * 50)
            print("📊 МЕТРИКИ ЗАПРОСОВ")
            print(f"📌 Всего запросов: {self.total_requests}")
            print(f"✅ 2xx: {self.status_2xx}")
            print(f"⚠️ 4xx: {self.status_4xx}")
            print(f"❌ 5xx: {self.status_5xx}")
            print("=" * 50)
            
            with open('metrics.log', 'a', encoding='utf-8') as f:
                f.write(f"Total: {self.total_requests}, 2xx: {self.status_2xx}, 4xx: {self.status_4xx}, 5xx: {self.status_5xx}\n")
            self.counter = 0
        
        return response
```

---

ШАГ 8: Тесты

📁 clients/tests.py

```python
from django.test import TestCase
from django.utils import timezone
from django.core.exceptions import ValidationError
from .models import Client
from datetime import date, timedelta

class ClientModelTest(TestCase):
    def setUp(self):
        self.client = Client.objects.create(
            full_name='Иванов Иван Иванович',
            email='ivanov@example.com',
            phone='+7-999-123-45-67',
            registration_date=date.today()
        )

    def test_client_creation(self):
        self.assertEqual(self.client.full_name, 'Иванов Иван Иванович')
        self.assertEqual(self.client.email, 'ivanov@example.com')

    def test_full_name_not_empty(self):
        client = Client(
            full_name='',
            email='test@example.com',
            registration_date=date.today()
        )
        with self.assertRaises(ValidationError):
            client.full_clean()

    def test_email_contains_at(self):
        client = Client(
            full_name='Петров Петр',
            email='petrovexample.com',
            registration_date=date.today()
        )
        with self.assertRaises(ValidationError):
            client.full_clean()

    def test_email_contains_dot(self):
        client = Client(
            full_name='Сидоров Сидор',
            email='sidorov@example',
            registration_date=date.today()
        )
        with self.assertRaises(ValidationError):
            client.full_clean()

    def test_email_unique(self):
        client2 = Client(
            full_name='Другой клиент',
            email='ivanov@example.com',  # дубликат
            registration_date=date.today()
        )
        with self.assertRaises(ValidationError):
            client2.full_clean()

    def test_phone_unique_if_provided(self):
        client2 = Client(
            full_name='Другой клиент',
            email='other@example.com',
            phone='+7-999-123-45-67',  # дубликат
            registration_date=date.today()
        )
        with self.assertRaises(ValidationError):
            client2.full_clean()

    def test_phone_can_be_empty(self):
        client = Client(
            full_name='Клиент без телефона',
            email='nophone@example.com',
            phone=None,
            registration_date=date.today()
        )
        client.full_clean()  # Не должно быть ошибки

    def test_registration_date_not_future(self):
        future_date = date.today() + timedelta(days=365)
        client = Client(
            full_name='Клиент из будущего',
            email='future@example.com',
            registration_date=future_date
        )
        with self.assertRaises(ValidationError):
            client.full_clean()


class ViewsTest(TestCase):
    def test_ping_endpoint(self):
        response = self.client.get('/ping/')
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json(), {'status': 'ok'})

    def test_index_page(self):
        response = self.client.get('/')
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'clients/client_list.html')

    def test_client_404(self):
        response = self.client.get('/999/update/')
        self.assertEqual(response.status_code, 404)
```

---

ШАГ 9: Шаблоны

Создайте папки:

```bash
mkdir -p clients/templates/clients
```

📁 clients/templates/clients/base.html

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Управление клиентами{% endblock %}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{% url 'client_list' %}">👥 Клиенты</a>
            <div class="ms-auto">
                <a href="{% url 'client_create' %}" class="btn btn-light btn-sm">➕ Добавить клиента</a>
            </div>
        </div>
    </nav>
    
    <div class="container mt-4">
        {% if messages %}
            {% for message in messages %}
                <div class="alert alert-{% if message.tags == 'error' %}danger{% else %}{{ message.tags }}{% endif %} alert-dismissible fade show">
                    {{ message }}
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            {% endfor %}
        {% endif %}
        
        {% block content %}{% endblock %}
    </div>
    
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

📁 clients/templates/clients/client_list.html

```html
{% extends 'clients/base.html' %}

{% block title %}Список клиентов{% endblock %}

{% block content %}
    <div class="d-flex justify-content-between align-items-center mb-4">
        <h1>📋 Список клиентов</h1>
        <a href="{% url 'client_create' %}" class="btn btn-primary">➕ Добавить клиента</a>
    </div>

    <div class="table-responsive">
        <table class="table table-striped table-hover">
            <thead class="table-dark">
                <tr>
                    <th>ID</th>
                    <th>ФИО</th>
                    <th>Email</th>
                    <th>Телефон</th>
                    <th>Дата регистрации</th>
                    <th>Действия</th>
                </tr>
            </thead>
            <tbody>
                {% for client in clients %}
                <tr>
                    <td>{{ client.id }}</td>
                    <td><strong>{{ client.full_name }}</strong></td>
                    <td>{{ client.email }}</td>
                    <td>{{ client.phone|default:"—" }}</td>
                    <td>{{ client.registration_date|date:"d.m.Y" }}</td>
                    <td>
                        <a href="{% url 'client_update' client.id %}" class="btn btn-sm btn-warning">✏️</a>
                        <a href="{% url 'client_delete' client.id %}" class="btn btn-sm btn-danger">🗑️</a>
                    </td>
                </tr>
                {% empty %}
                <tr>
                    <td colspan="6" class="text-center py-4">
                        <p class="text-muted">Нет клиентов</p>
                        <a href="{% url 'client_create' %}" class="btn btn-primary btn-sm">Добавить первого клиента</a>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
{% endblock %}
```

📁 clients/templates/clients/client_form.html

```html
{% extends 'clients/base.html' %}

{% block title %}
    {% if client %}Редактирование{% else %}Добавление{% endif %} клиента
{% endblock %}

{% block content %}
    <div class="row justify-content-center">
        <div class="col-md-8">
            <div class="card shadow">
                <div class="card-header bg-primary text-white">
                    <h3 class="card-title mb-0">
                        {% if client %}✏️ Редактирование клиента{% else %}➕ Новый клиент{% endif %}
                    </h3>
                </div>
                <div class="card-body">
                    <form method="post">
                        {% csrf_token %}
                        
                        {% for field in form %}
                        <div class="mb-3">
                            <label for="{{ field.id_for_label }}" class="form-label">
                                {{ field.label }}
                                {% if field.field.required %}<span class="text-danger">*</span>{% endif %}
                            </label>
                            {{ field }}
                            {% if field.errors %}
                                <div class="text-danger">
                                    {% for error in field.errors %}
                                        <small>{{ error }}</small>
                                    {% endfor %}
                                </div>
                            {% endif %}
                            {% if field.help_text %}
                                <small class="text-muted">{{ field.help_text }}</small>
                            {% endif %}
                        </div>
                        {% endfor %}
                        
                        <div class="d-flex gap-2">
                            <button type="submit" class="btn btn-success">💾 Сохранить</button>
                            <a href="{% url 'client_list' %}" class="btn btn-secondary">↩️ Отмена</a>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>
{% endblock %}
```

📁 clients/templates/clients/client_confirm_delete.html

```html
{% extends 'clients/base.html' %}

{% block title %}Удаление клиента{% endblock %}

{% block content %}
    <div class="row justify-content-center">
        <div class="col-md-6">
            <div class="card border-danger shadow">
                <div class="card-header bg-danger text-white">
                    <h3 class="card-title mb-0">⚠️ Подтверждение удаления</h3>
                </div>
                <div class="card-body">
                    <p>Вы уверены, что хотите удалить клиента?</p>
                    
                    <div class="alert alert-warning">
                        <h5 class="alert-heading">{{ client.full_name }}</h5>
                        <hr>
                        <p class="mb-1">📧 {{ client.email }}</p>
                        <p class="mb-0">📱 {{ client.phone|default:"Телефон не указан" }}</p>
                    </div>
                    
                    <p class="text-danger"><small>⚠️ Это действие нельзя отменить.</small></p>
                    
                    <form method="post">
                        {% csrf_token %}
                        <div class="d-flex gap-2">
                            <button type="submit" class="btn btn-danger">✅ Да, удалить</button>
                            <a href="{% url 'client_list' %}" class="btn btn-secondary">❌ Нет, отмена</a>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>
{% endblock %}
```

---

📋 ЧЕК-ЛИСТ ПРОВЕРКИ

Валидация (проверьте все правила):

№ Правило Как проверить
1 full_name не пустое Создать с пустым именем → ошибка
2 email содержит @ Создать с "testexample.com" → ошибка
3 email содержит . Создать с "test@example" → ошибка
4 email уникальный Создать дубликат email → ошибка
5 phone уникальный если указан Создать дубликат телефона → ошибка
6 phone может быть пустым Создать без телефона → OK
7 registration_date не в будущем Создать с датой в будущем → ошибка

---

🚀 БЫСТРЫЙ ЗАПУСК

```bash
# 1. Создание проекта
mkdir clients_project
cd clients_project
python -m venv venv
source venv/bin/activate  # или venv\Scripts\activate

# 2. Установка
pip install django python-dotenv whitenoise

# 3. Создание проекта и приложения
django-admin startproject exam_project .
python manage.py startapp clients

# 4. Копируем все файлы (выше)

# 5. Миграции
python manage.py makemigrations
python manage.py migrate

# 6. Суперпользователь
python manage.py createsuperuser

# 7. Сборка статики
python manage.py collectstatic

# 8. Тесты
python manage.py test

# 9. Запуск
python manage.py runserver
```

---

✅ ГОТОВО!

Проект полностью соответствует экзаменационному билету по варианту "Клиенты (clients)".

Все требования выполнены! Удачи на экзамене! 🚀

# Практическая работа - Настройка OAuth 2.0 авторизации в Django приложении (Django + OAuth + Google)

---

## 1. Цель работы
Улучшите приложение для голосований из лабораторных работ и реализуйте вход на сайт этого приложения через
стандарт OAuth, при условии, что пользователь имеет учётную запись Google.
Реализовать вход на сайт Django-приложения через стандарт OAuth, используя учетную запись Google, и интегрировать страницу авторизации в собственный стиль приложения.

---

## 2. Что было сделано до работы
Улучшено приложение для голосований ("Polls from the Crypt") с реализацией входа через Google OAuth 2.0.
- Приложение Django для опросов с регистрацией, верификацией email и голосованием.
- Использование Bootstrap 5 и Crispy Forms для адаптивного интерфейса.
- Страницы:
  - `polls/login.html` — форма входа
  - `polls/register.html` — форма регистрации
  - `polls/index.html`, `polls/detail.html`, `polls/results.html` — страницы опросов
- Email верификация через консоль.
- Настроена админка с кастомным `base_site.html`.

---

## 3. Добавлено ля OAuth

### 3.1 Установка зависимостей

```bash
pip install django-allauth
```

### 3.2 Настройка `settings.py`

```python
INSTALLED_APPS = [
    # ...
    "django.contrib.sites",
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.google",
]

SITE_ID = 1

AUTHENTICATION_BACKENDS = (
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
)

LOGIN_REDIRECT_URL = "/polls/"
LOGOUT_REDIRECT_URL = "/polls/"
```

### 3.3 Создание приложения Google в Google Cloud Console

- Получили **Client ID** и **Client Secret**
- Установили **Redirect URL**: `http://127.0.0.1:8000/accounts/google/login/callback/`

### 3.4 Настройка Social App в Django Admin

- Provider: Google  
- Client ID и Secret  
- Site: example.com (Site ID = 1)

---

## 4. Настройка шаблонов

### 4.1 Кастомизация формы входа `polls/login.html`

```html
{% extends 'polls/base.html' %}
{% load crispy_forms_tags %}
{% load socialaccount %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6 col-lg-5">
        <div class="card shadow-lg border-0 rounded-4">
            <div class="card-body p-4">
                <h2 class="text-center mb-4">🔐 Login</h2>

                <!-- Обычная форма входа -->
                <form method="POST">
                    {% csrf_token %}
                    {{ login_form|crispy }}
                    <div class="d-grid gap-2 mt-3">
                        <button class="btn btn-dark btn-lg" type="submit">Login</button>
                    </div>
                </form>

                <hr class="my-4">

                <!-- Кнопка входа через Google -->
                <div class="d-grid gap-2 mb-3">
                    <a href="{% provider_login_url 'google' %}" class="btn btn-outline-danger btn-lg">
                        <img src="https://developers.google.com/identity/images/g-logo.png"
                             alt="Google logo"
                             width="20" height="20"
                             class="me-2">
                        Sign in with Google
                    </a>
                </div>

                <hr class="my-4">

                <p class="text-center mb-0">
                    Don't have an account?
                    <a href="{% url 'polls:register' %}">Register here</a>
                </p>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### 4.2 Кастомизация страниц allauth для Google OAuth

- Создать папку: `templates/socialaccount/`
- Добавить файлы:
  - `login.html`
  - `signup.html`
- Пример `socialaccount/login.html`:

```html
{% extends 'polls/base.html' %}
{% load socialaccount %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6 col-lg-5">
        <div class="card shadow-lg border-0 rounded-4">
            <div class="card-body p-4">
                <h2 class="text-center mb-4">🔐 Sign In via Google</h2>
                <p class="text-center">You are about to sign in using a third-party account from Google.</p>
                
                <div class="d-grid gap-2 mt-3">
                    <a href="{% provider_login_url 'google' %}" class="btn btn-outline-danger btn-lg">
                        <img src="https://developers.google.com/identity/images/g-logo.png"
                             alt="Google logo"
                             width="20" height="20"
                             class="me-2">
                        Continue
                    </a>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

## 5. Как работает OAuth в приложении

1. Пользователь кликает кнопку **Sign in with Google** на `/polls/login/`.
2. Перенаправляется на страницу подтверждения провайдера (кастомная `socialaccount/login.html`) или сразу на Google.
3. После успешного входа через Google пользователь возвращается на `LOGIN_REDIRECT_URL = "/polls/"`.
4. Если пользователь вошел впервые, создается новая учетная запись в Django.
5. Весь процесс полностью интегрирован с интерфейсом приложения (Bootstrap, Crispy Forms).

### Адаптер Google OAuth: `MySocialAccountAdapter`

В проекте реализован кастомный адаптер для социальной аутентификации через Google — `MySocialAccountAdapter`. Его задача — **плавно связывать Google-аккаунт с уже существующим пользователем на сайте по email**.

#### Как работает адаптер

1. **Метод `pre_social_login`**
    ```python
    def pre_social_login(self, request, sociallogin):
    ```
    - Вызывается перед входом через Google.
    - Параметр `sociallogin` содержит данные пользователя от Google (email, имя, avatar и т.д.).

2. **Проверка уже привязанного аккаунта**
    ```python
    if sociallogin.is_existing:
        return
    ```
    - Если Google-аккаунт уже привязан к существующему пользователю на сайте, адаптер ничего не делает.

3. **Получение email пользователя**
    ```python
    user_model = get_user_model()
    email = user_email(sociallogin.user)

    if not email:
        return
    ```
    - Берётся email из данных Google.
    - Если email отсутствует, связать аккаунт нельзя, и метод завершает работу.

4. **Поиск существующего пользователя и привязка**
    ```python
    existing_user = user_model.objects.get(email__iexact=email)
    sociallogin.connect(request, existing_user)
    ```
    - Ищется пользователь в базе по email (без учёта регистра).
    - Если найден — Google-аккаунт привязывается к этому пользователю.
    - Если не найден — будет создан новый пользователь автоматически.

#### Практическая польза

- Пользователи с одинаковым email не создают несколько аккаунтов.
- Вход через Google "плавно" объединяется с существующими аккаунтами.
- Новые пользователи создаются только при отсутствии совпадений по email.

> 🔹 **Иллюстрация работы:**  
> Пользователь с email `example@gmail.com` на сайте пытается войти через Google:
> - Если аккаунт существует → Google привязывается к существующему аккаунту.  
> - Если аккаунта нет → создаётся новый пользователь с привязанным Google.


---

## 6. Структура проекта

```
django_polls/
├── mysite/
│   ├── settings.py
│   ├── adapters.py
│   ├── urls.py
│   ├── asgi.py
│   ├── wsgi.py
│   └── __init__.py
│
├── polls/
│   ├── migrations/
│   ├── static/polls/
│   │   ├── style.css
│   │   └── images/
│   ├── templates/polls/
│   │   ├── base.html
│   │   ├── index.html
│   │   ├── detail.html
│   │   ├── results.html
│   │   ├── login.html
│   │   ├── register.html
│   │   ├── poll_new_edit.html
│   │   ├── account_activation_success.html
│   │   ├── account_activation_failure.html
│   │   └── verification_email.html
│   ├── models.py
│   ├── views.py
│   ├── urls.py
│   ├── forms.py
│   ├── admin.py
│   ├── apps.py
│   ├── tests.py
│   └── __init__.py
│
├── templates/socialaccount/
│   ├── login.html
│   └── signup.html
│
├── templates/admin/
│   └── base_site.html
│
├── manage.py
└── db.sqlite3
```

---

## 7. Скриншоты приложения

### Главная страница приложения
![Главная страница приложения](https://github.com/user-attachments/assets/04c7dba8-e152-49fc-a1e6-c097d6100c60)

Главная страница проекта **Polls from the Crypt** после авторизации.  
Отображает список всех опросов, кнопку создания новых (для администратора), а также навигацию с именем пользователя.

---

### Форма входа
![Форма входа](https://github.com/user-attachments/assets/593b7c4b-bb67-4532-a26c-1fcfa89cba25)

Кастомная форма входа, оформленная в стиле проекта.  
Добавлена кнопка **Sign in with Google**, ведущая на страницу авторизации через OAuth 2.0.

---

### Страница подтверждения входа через Google
![Страница подтверждения входа через Google](https://github.com/user-attachments/assets/da78b8e5-7e12-4589-9398-fd70f10d9e09)

Промежуточная страница авторизации, отображаемая при переходе на Google.  
Пользователь подтверждает вход через свой Google-аккаунт.

---

### Выбор Google-аккаунта
![Выбор аккаунта Google](https://github.com/user-attachments/assets/f8b129cf-45ed-49df-b079-37e86b077f78)

Интерфейс Google для выбора аккаунта, с которого будет выполнен вход.  
После выбора пользователь перенаправляется обратно на сайт уже в авторизованном состоянии.

---



## 8. Вывод

- OAuth авторизация через Google успешно интегрирована.
- Все страницы, связанные с логином через Google, оформлены в стиле приложения.
- Пользовательский интерфейс полностью адаптивен и использует Bootstrap + Crispy Forms.
- Проект готов для дальнейшего расширения (например, добавление других провайдеров OAuth).

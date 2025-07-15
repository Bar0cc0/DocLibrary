# DJANGO

## Table of Contents
0. Useful Resources
1. Basic Structure of a Django Project
2. Shell Commands
3. Project Settings
4. Define Data Models
5. Customize Admin Console
6. Define Views and Templates
7. Render Data from Database
8. HTTP Methods & Form Validation
9. Tests
10. How To guides
  	- Route URLs Using Primary Keys
  	- Bind Functions with Signals
  	- Use Foreign Keys
11. Migrating from SQLite to PostgreSQL
12. Security Configuration for production


## 0. Useful Resources

- [Advanced Form Rendering with Django Crispy Forms](https://simpleisbetterthancomplex.com/tutorial/2018/11/28/advanced-form-rendering-with-django-crispy-forms.html)
- [Building a Social Network with Django](https://realpython.com/django-social-network-1/)

## 1. Basic Structure of a Django Project

```
ROOT_DIR
├── COMPONENT
│   ├── models.py    --> Database schema & data models
│   ├── admin.py     --> Customize admin console
│   ├── views.py     --> Render webpages
│   └── urls.py      --> Route webpages (folder needs to be manually added)
│
├── MEDIA
│
├── PROJECT
│   ├── settings.py  --> Register apps
│   ├── urls.py      --> URL routing for the project
│   └── wsgi.py      --> Entry point for WSGI-compatible web servers
│
├── TEMPLATES
│   ├── base.html
│   └── *.html
│
├── database.connector
└── manage.py
```

## 2. Shell Commands

### Project Setup
```bash
# Create a project 
django-admin startproject <PROJECT_NAME>

# Create a new component
python manage.py startapp <COMPONENT>

# Migrate
python manage.py makemigrations    # Whenever a new app is created or modified
python manage.py migrate           # Hooks up django apps and db == sync db and project 

# Prints the SQL for the named migration
python manage.py sqlmigrate <APP_NAME> <MIGRATION_NAME>

# Create admin profile
python manage.py createsuperuser  

# Start server
python manage.py runserver localhost:8080  # Start server on localhost:8080
python -m gunicorn mysite.asgi:application # Start server on localhost:8000
python -m gunicorn Portefolio.asgi:application -k uvicorn.workers.UvicornWorker -b 127.0.0.1:8080  # Start server on localhost:8080

# Start Python interpreter
python manage.py shell
```

## 3. Project Settings (`<PROJECT>/settings.py`)

### Register Components
```python
INSTALLED_APPS = [
    # Django defaults
    'django.contrib.admin',
    'django.contrib.auth',
    # ...
    
    # Third-party apps
    'crispy_forms',
    'crispy_bootstrap4',
    
    # Your apps
    'my_app',
    'my_other_app.apps.My_other_appConfig'
]
```

### Database Configuration

#### Native DB Support
```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.<engine>",
        "NAME": "mydatabase",
        "USER": "mydatabaseuser",
        "PASSWORD": "mypassword",
        "HOST": "127.0.0.1",
        "PORT": "5432",
    }
}
```

Available engines:
- `django.db.backends.postgresql`
- `django.db.backends.mysql`
- `django.db.backends.sqlite3` (default)
- `django.db.backends.oracle`

#### Non-Native DB: SQL Server
```python
DATABASES = {
    "default": {
        "ENGINE": "mssql",
        "NAME": "DATABASE_NAME",
        "USER": "USER_NAME",
        "PASSWORD": "PASSWORD",
        "HOST": "HOST_ADDRESS",
        "PORT": "1433",
        "OPTIONS": {
            "driver": "ODBC Driver 17 for SQL Server", 
        },
    },
}
```

### Other Global Variables
```python
STATIC_URL = 'static/'
MEDIA_ROOT = 'media/'
CRISPY_FORMS_PACK = 'bootstrap4'
CRISPY_ALLOWED_TEMPLATE_PACKS = "bootstrap4"
CRISPY_TEMPLATE_PACK = 'bootstrap4'
```

## 4. Define Data Models (`<COMPONENT>/models.py`)

First, create a new component. See [Django model field reference](https://docs.djangoproject.com/en/4.2/ref/models/fields/) for data types.

```python
from django.db import models
from django.dispatch import receiver

class <CLASSNAME>(models.Model):
    vartxt = models.TextField()

@receiver(models.signals.post_save, sender=User) 
def create_entry(sender, instance, created, **kwargs):
    if created:
        entry = <CLASSNAME>(user=instance)
        entry.save()
```

Add component to `<PROJECT>/settings.py`:
```python
INSTALLED_APPS = ['COMPONENT.apps.COMPONENTConfig']  # name is defined in component/apps.py
```

Register to admin console in `<COMPONENT>/admin.py`:
```python
from .models import <CLASSNAME>
admin.site.register(<CLASSNAME>)
```

## 5. Customize Admin Console (`<COMPONENT>/admin.py`)

```python
from django.contrib import admin
from django.contrib.auth.models import User, Group
from . import models

class ProfileInline(admin.StackedInline):
    model = models.Profile
    
class UserAdmin(admin.ModelAdmin):
    model = User
    # Only display the "username" field
    fields = ["username"]
    # Display the profile information within the Users tab
    inlines = [ProfileInline]

# Admin customization
admin.site.unregister(User)
admin.site.register(User, UserAdmin)
admin.site.unregister(Group)

# Model registration for the admin site
# Display the profile information as a separate tab 
# but can be removed if the profile information is displayed inlined within the Users tab
admin.site.register(models.Profile) 
```

## 6. Define Views and Templates

### A. Basic View Definition

Edit `<COMPONENT>/views.py`:
```python
from django.shortcuts import render
def BasicView(request):
    some_text_to_display = "hello world"
    return render(request, 'base.html', some_text_to_display)
```

Create `<COMPONENT>/urls.py` & add URL path:
```python
from django.urls import path
from .views import BasicView
app_name = '<COMPONENT>'
urlpatterns = [
    path('', BasicView, name="basicview"),
]
```

Add URL routing to `<PROJECT>/urls.py`:
```python
from django.urls import path, include
from <COMPONENT>.views import BasicView
urlpatterns = [
    path('', include('component.urls')),
]
```

### B. Class Based Generic Views

Edit `<COMPONENT>/views.py`:
```python
from django.views.generic import (
    CreateView,
    DetailView,
    ListView,
    UpdateView,
    DeleteView
)

from .models import <CLASSNAME>
def class_list_view(ListView) -> None:                  # will look for <COMPONENT>/<template>_list.html
    model = <CLASSNAME>
    template_name = '<template_path>.html'              # overrides the generic view 
```

Add to `<PROJECT>/urls.py`:
```python
urlpatterns = [
    path('', class_list_view.as_view(), name='class_list_view')
]
```

### C. Advanced View Definition: URL Routing

Edit `<COMPONENT>/views.py`:
```python
from django.shortcuts import render, get_object_or_404, redirect
from .models import <CLASSNAME>                                  # imports components' models
def _view(request, my_id):
    try:    
        obj = get_object_or_404(<CLASSNAME>, id=my_id)          # get_list_or_404(<CLASSNAME>, "lookup_param"=value)
        return redirect('../')
    except: raise DoesNotExistsError('Not possible')
    context = {'object': obj}
    return render(request, "<TEMPLATE_NAME>.html", context)     # instead of HttpResponse()
```

Add view to `<COMPONENT>/urls.py`:
```python
from .views import <VIEW_NAME>                                  
urlpatterns = [                                                 
    path('view_name/<int:my_id>/', <VIEW_NAME>, name='view_name')          
]                                                               
```

Add instance method for dynamic URL routing in `<COMPONENT>/models.py`:
```python
from django.urls import reverse 
class CLASSNAME(models.Model):
    def get_absolute_url(self):
        return reverse('urlpatterns_name_in-urls.py', kwargs={'id':self.id})
```

### D. Templates Inheritance

Create a 'TEMPLATES' folder in ROOT_DIR and set path in `<PROJECT>/settings.py`:
```python
TEMPLATES = [
    {
        # ...
        "DIRS": [os.path.join(BASE_DIR, "TEMPLATES")],
        # ...
    }
]
```

Create a base class called `TEMPLATES/default.html`:
```html
<!doctype html>
<html>
<head> 
    <title>...</title>
</head>
<body>
    <div>...</div>                                   <!-- static code --> 
    <div>{% block <VAR> %}...{% endblock %}</div>    <!-- dynamic code -->
</body>
</html>
```

Create templates through inheritance:
```html
{% extends "default.html" %}
{% include "<OTHER_TEMPLATE_NAME>.html" %}
{% block <VAR> %}  
    ...
{% endblock %}
```

### E. Render with Hardcoded Context (Using Tags and Filters)

See [Django template built-ins](https://docs.djangoproject.com/en/4.2/ref/templates/builtins/)

Edit `<COMPONENT>/views.py`:
```python
from django.shortcuts import render 
def _view(request, *args, **kwargs):
    my_context = {
        'key_1': value,
        'key_2': [list],
        'key_3': bool(1)
    }
    return render(request, "<TEMPLATE_NAME>.html", my_context)
```

Add the view to `<PROJECT>/urls.py`

Edit `TEMPLATES/<TEMPLATE_NAME>.html`:
```html
{% extends "default.html" %}
{% block <VAR> %}  
    <p> 
        {{ key_1 }},
        <ul> 
            {% for elmt in key_2 %} 
                <li>{{ elmt }}</li> 
            {% endfor %} 
        </ul>,
        {% if key_3 == True %} 
            {{ key_3 }} 
        {% else %} 
            {{ key_3|add:1 }}
        {% endif %}
    </p>
{% endblock %}
```

### F. Removing Hardcoded URLs in Templates (Decoupling)

Template:
```html
<!-- BAD: tightly coupled templates & urls -> Can't easily manage path modifications -->
<a href="/polls/{{ question.id }}/"></a>

<!-- GOOD: Decouples templates & url definitions -->
<a href="{% url 'detail' question.id %}"></a>
```

`component.urls.py`:
```python
path("<int:question_id>/", views.detail, name="detail"),         # the 'name' value as called by the {% url %} template tag
path("specifics/<int:question_id>/", views.detail, name="detail"),  # so the URL can be easily modified without having to track all instances
```

### G. Removing Hardcoded URLs in Templates (Namespacing)

`component.urls.py`:
```python
app_name = "polls"
urlpatterns = [
    path("", views.index, name="index"),
    path("<int:question_id>/", views.detail, name="detail"),
    path("<int:question_id>/results/", views.results, name="results"),
]
```

Template:
```html
<a href="{% url 'polls:detail' question.id %}"></a>    <!-- <=> href="{% url 'detail' question.id %}" -->
<a href="{% url 'polls:results' question.id %}"></a>
```

## 7. Render Data from Database

### A. Display One Element from Database

Edit `<COMPONENT>/views.py`:
```python
from .models import <CLASSNAME>
def class_item_view(request):
    item = <CLASSNAME>.objects.get(id=<int>)
    context = {
        'item': item
    }
    return render(request, <TEMPLATE_NAME>.html, context) 
```

Add the view to `<PROJECT>/urls.py`

Use context variables in `<TEMPLATES>/<TEMPLATE_NAME>.html`:
```html
{{ item.field_1 }}, {{ item.field_2 }} 
```

### B. List of Database Objects

Edit `<COMPONENT>/views.py`:
```python
from .models import <CLASSNAME>
def class_list_view(request):
    queryset = <CLASSNAME>.objects.all()
    context = {
        'object_list': queryset
    }
    return render(request, <TEMPLATE_NAME>.html, context) 
```

Add the view to `<PROJECT>/urls.py`

Edit `TEMPLATES/<TEMPLATE_NAME>.html`:
```html
{% extends "default.html" %}
{% block <VAR> %} 
    {{ object_list }}
    {% for user in object_list %}
        <p>
            <a href='{{ entity.get_absolute_url }}'>
                {{ user.id }}, 
                {{ user.alias }} 
            </a>
        </p>
        <p>
            <input type="radio" name="user" id="user{{ forloop.counter }}" value="{{ user.id }}">
            <label for="user{{ forloop.counter }}">{{ user.alias }}</label>
        </p>
    {% endfor %}
{% endblock %}
```

### C. Accessing Attributes & Methods in `views.py`

```python
<model_instance_name>.id
<model_instance_name>.<attribute/field_name>
<model_instance_name>.<function_name>()

<model_instance_name>.objects.all()                              # .all()[<OFFSET:int>:<LIMIT:int>:<STEP:int>]
<model_instance_name>.objects.get(<attribute>__<lte|gt|exact>=<some_value>)      # .order_by(<attribute>).get()[<OFFSET:int>:<LIMIT:int>]
<model_instance_name>.objects.filter(<attribute>__<lte|gt|exact>=<some_value>)   # lookup types{iexact, contains, startswith, endswith, isnull, in}
<model_instance_name>.objects.exclude(<attribute>__<lte|gt|exact>=<some_value>)

<model_instance_name>.count()
<model_instance_name>.save()
<model_instance_name>.delete()

# F() references a model field within a query to compare values of two different fields on the same model instance
<model_instance_name>.objects.filter(<field_1>=F(<field_2>))

# .create(val_1=,val_2=,...) function == INSERT(val_1=,val_2=,...)
<model_instance_name>.objects.create(<attribute>=<some_value>)

# "_set" refers to the "other side" of a ForeignKey relation 
<model_instance_name_ONE>.<model_instance_name_MANY>_set.<any_method_or_attribute>
```

## 8. HTTP Methods & Form Validation

Create `<COMPONENT>/forms.py`:
```python
from django import forms 
from .models import MailClass

class Mail_Form(forms.ModelForm):
    email = forms.EmailField()
    # Model metadata. List of options: https://docs.djangoproject.com/en/3.2/ref/models/options/
    class Meta:
        model   = MailClass()
        fields  = ['Alias', 'Age']
        exclude = ('user',)  
    
    def email_check(self):
        email = self.cleaned_data.get('email')
        if not email.endswith('edu'): 
            raise forms.ValidationError('Not a valid email')
        return email
```

Edit `<COMPONENT>/views.py`:
```python
from django.shortcuts import render, redirect
from .forms import Mail_Form, delete_form

def Some_View(request):
    # 'or None' if function gets called with GET := Django will instantiate Mail_Form as unbound form (i.e. no data)
    form = Mail_Form(request.POST or None)     
    # If function gets called with POST := bound form (i.e. pass data for validation)
    if request.method == "POST":
        if form.is_valid():
            form.save(commit=False)
            form.user = request.user
            form.save()
            return redirect("component:some_view")
    context = {'form': form}
    return render(request, 'FORM_TEMPLATE.html', context) 
```

Create `<TEMPLATES>/<FORM_TEMPLATE>.html`:
```html
{% extends 'default.html' %}
{% block content %}
    <!-- Insert an entity -->
    <form action='.' method='POST'> <!-- <form action="{% url 'polls:vote' question.id %}" method="post"> -->
        {% csrf_token %}
        {{ form.as_p }}
        <input type='submit' value='Save'> 
    </form>
    
    <!-- Delete an entity -->
    <form action='.' method='POST'> 
        {% csrf_token %}
        <p>Do you want to delete the entry?</p>
        <input type='submit' value='Yes'> 
        <a href='../'>No</a>  
    </form>
{% endblock %}
```

Add to `<PROJECT>/admin.py`:
```python
app_name = "component"
urlpatterns = [path("", Some_View, "some_view")]
```

Add to `<PROJECT>/urls.py`

## 9. Tests

`component/tests.py`:
```python
from django.test import TestCase
from .models import <CLASSNAME>
from django.urls import reverse
    
def create_mock_data(*args, **kwargs):
    ...
    return <CLASSNAME>.objects.create(...)

class TestClassName(TestCase):
    def _init_(self):
        self.view_url = reverse("namespace:view", args=(...))
        
    def test_name(self):
        ret_val_to_test = create_mock_data(*args, **kwargs)
        http_response = self.client.get(self.view_url)
        self.assertIs(ret_val_to_test, val_expected)
        self.assertEqual(http_response.status_code, val_expected)
        self.assertContains(http_response, val_expected)
        self.assertQuerySetEqual(response.context["val"], [queryset_objects,],)
```

Run test:
```bash
python manage.py test <COMPONENT>
```

## 10. How To

### A. Route URLs Using Primary Keys

`<COMPONENT>/models.py`:
```python
from django.db import models
from django.contrib.auth.models import User

# Define a toy-model
class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    follows = models.ManyToManyField('self', related_name='followed_by', symmetrical=False, blank=True)
    
    def __str__(self):
        return self.user.username
```

`<COMPONENT>/views.py`:
```python
from django.shortcuts import render
from . import models

# This view holds all the profile_list items -> href in profile_list.html
def profile_list(request):
    profiles = models.Profile.objects
    context = {'profiles': profiles.all()}
    return render(request, 'profile_list.html', context)

# This view references profile on pk -> urlpattern := component/<int:pk>
def profile(request, pk):
    profile = models.Profile.objects.get(pk=pk)
    return render(request, "profile.html", {"profile": profile})
```

`<COMPONENT>/templates/profile_list.html`:
```html
{% extends 'base.html' %}
{% block content %}
    {% for profile in profiles %}
    <a href="{% url 'component:profile' profile.id %}">{{ profile.user.username }}</a>
    {% endfor %}
{% endblock content %}
```

`<COMPONENT>/templates/profile.html`:
```html
{% extends 'base.html' %}
{% block content %}
    <div class="container">
        {% for following in profile.follows.all %}
        {{ following }}
        {% endfor %}
    </div>
    <div class="container">
        {% for follower in profile.followed_by.all %}
        {{ follower }}
        {% endfor %}
    </div>
{% endblock content %}
```

`<COMPONENT>/urls.py`:
```python
from django.urls import path
from . import views
app_name = 'component'
urlpatterns = [
    path('profile_list/', views.profile_list, name='profile_list'),
    path("profile/<int:pk>", views.profile, name="profile"),
]
```

`<PROJECT>/urls.py`:
```python
from django.contrib import admin
from django.urls import path, include
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('component.urls')),
]
```

`<PROJECT>/settings.py`:
```python
INSTALLED_APPS = [
    'component',
    # ...
]
```

### B. Bind Functions with Signals

`<COMPONENT>/models.py`:
```python
from django.db import models
from django.contrib.auth.models import User
from django.dispatch import receiver

# Contains information that users create only after they have created a user account
class Profile(models.Model):
    # ...

# Sets the post-save signal to execute create_profile() every time the User model executes .save()
@receiver(models.signals.post_save, sender=User) 
def create_profile(sender, instance, created, **kwargs):
    # ...
```

### C. Use Foreign Keys

`<COMPONENT>/models.py`:
```python
from django.db import models
from django.contrib.auth.models import User
# Define a toy-model ::= user <-one-to-many-> messages
class Message(models.Model):
    user = models.ForeignKey(
                User, 
                related_name="messages",    # allows to access the messages objects from the user side of the relationship through "profile.user.messages.all"
                on_delete=models.DO_NOTHING)
    body = models.CharField(max_length=140)
    
    def __str__(self): 
        return f"{self.user} - {self.body[:10]}..."
```

## 11. Migrating from SQLite to PostgreSQL

### 1. Install Required Packages

```bash
pip install psycopg2-binary dj-database-url
```

### 2. Create the PostgreSQL Database

```sql
CREATE DATABASE "PortefolioDB";
CREATE USER postgres WITH PASSWORD 'postgres';
GRANT ALL PRIVILEGES ON DATABASE "PortefolioDB" TO postgres;
```

### 3. Export Data from SQLite

First, temporarily switch back to SQLite:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
"""
DATABASES = {
    'default': dj_database_url.config(
        default='postgresql://postgres:postgres@localhost:5432/PortefolioDB',
        conn_max_age=600
    )
}
"""
```

Then, dump your data to a JSON file:

```bash
python manage.py dumpdata --exclude auth.permission --exclude contenttypes > data.json
```

### 4. Switch to PostgreSQL

```python
"""
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
"""
DATABASES = {
    'default': dj_database_url.config(
        default='postgresql://postgres:postgres@localhost:5432/PortefolioDB',
        conn_max_age=600
    )
}
```

### 5. Create the Schema in PostgreSQL

```bash
python manage.py migrate
```

### 6. Load Data into PostgreSQL

```bash
python manage.py loaddata data.json
```

### 7. Verify the Migration

```bash
python manage.py shell
```

```python
>>> from app_projects.models import Project  # Use one of your models
>>> Project.objects.all()  # Check if your data is there
```

### Additional Tips

1. **Backup your SQLite database** before starting the migration process
2. **Test in development** before applying to production
3. **Handle model-specific issues**: Some models may need special attention when migrating

### Troubleshooting

- If you get encoding errors during loaddata, try:
  ```bash
  python -Xutf8 manage.py loaddata data.json
  ```

- If some model relationships cause errors, you may need to migrate models in order:
  ```bash
  python manage.py dumpdata app_projects.SimpleModel > simple_model.json
  python manage.py dumpdata app_projects.ComplexModel > complex_model.json
  
  # Then load them in the same order
  python manage.py loaddata simple_model.json
  python manage.py loaddata complex_model.json
  ```


## 12. Security Configuration


### 1. Secret Key Management

```python
# Replace this
SECRET_KEY = 'django-insecure-aa*@1u_jky7@b7m*tsvx@=g9+jnqc5@ih1i74fhch3&z553g@-'

# With this
SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY', 'django-insecure-aa*@1u_jky7@b7m*tsvx@=g9+jnqc5@ih1i74fhch3&z553g@-')
```

For production, set the environment variable and NEVER use the default value.

### 2. Debug Mode Configuration

```python
# Replace this
DEBUG = True # Should never be True in production

# With this
DEBUG = os.environ.get('DJANGO_DEBUG', '').lower() != 'false'
```

### 3. ALLOWED_HOSTS Configuration

```python
# Replace this
ALLOWED_HOSTS = [] # Should never be empty in production

# With this
ALLOWED_HOSTS = os.environ.get('DJANGO_ALLOWED_HOSTS', 'localhost,127.0.0.1').split(',')
if os.environ.get('RENDER_EXTERNAL_HOSTNAME'):
    ALLOWED_HOSTS.append(os.environ.get('RENDER_EXTERNAL_HOSTNAME')) 
    ALLOWED_HOSTS.append('yourdomain.com') # Add your production domain here
```

### 4. Security Middleware Settings

Add these settings after your MIDDLEWARE list:

```python
# Security settings
if not DEBUG:
    # HTTPS/SSL
    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
	CSRF_COOKIE_HTTPONLY = True
	CSRF_USE_SESSIONS = True
    
    # HSTS settings
    SECURE_HSTS_SECONDS = 31536000  # 1 year
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_HSTS_PRELOAD = True
    
    # Content security
    SECURE_CONTENT_TYPE_NOSNIFF = True
    SECURE_BROWSER_XSS_FILTER = True
    X_FRAME_OPTIONS = 'DENY'
```

### 5. Database Security

For your PostgreSQL connection:

```python
# Replace this
DATABASES = {
    'default': dj_database_url.config(
        default='postgresql://postgres:postgres@localhost:5432/portefoliodb',
        conn_max_age=600
    )
}

# With environment variables
DATABASES = {
    'default': dj_database_url.config(
        default=os.environ.get('DATABASE_URL', 'postgresql://postgres:postgres@localhost:5432/portefoliodb'),
        conn_max_age=600,
        conn_health_checks=True,  # Django 4.1+
        ssl_require=not DEBUG,    # Require SSL in production
    )
}
```

### 6. Rate Limiting (Protection Against Brute Force Attacks)

Install Django Axes:

```bash
pip install django-axes
```

Add it to `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...existing apps...
    'axes',
]

MIDDLEWARE = [
    # Add to the end of your middleware list
    'axes.middleware.AxesMiddleware',
]

# Axes configuration
AXES_FAILURE_LIMIT = 5  # Number of login attempts before blocking
AXES_COOLOFF_TIME = 1  # Hours before resetting the counter
AXES_LOCKOUT_TEMPLATE = None  # Custom template for locked-out users
```

### 7. Content Security Policy

Install Django CSP:

```bash
pip install django-csp
```

Add to `INSTALLED_APPS` and middleware:

```python
INSTALLED_APPS = [
    # ...existing apps...
    'csp',
]

MIDDLEWARE = [
    # Add after SecurityMiddleware
    'csp.middleware.CSPMiddleware',
    # ...other middleware...
]

# CSP settings
CSP_DEFAULT_SRC = ("'self'",)
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'", "https://cdnjs.cloudflare.com")
CSP_SCRIPT_SRC = ("'self'", "'unsafe-inline'", "https://cdnjs.cloudflare.com")
CSP_FONT_SRC = ("'self'", "https://cdnjs.cloudflare.com")
CSP_IMG_SRC = ("'self'", "data:")
```

### 8. Admin URL Protection

For your `urls.py` file, change the admin URL to something non-standard:

```python
urlpatterns = [
    path('custom-admin-url/', admin.site.urls),
    # other URL patterns...
]
```

### 9. Logging for Security Events

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'WARNING',
            'class': 'logging.FileHandler',
            'filename': os.path.join(BASE_DIR, 'security.log'),
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django.security': {
            'handlers': ['file'],
            'level': 'WARNING',
            'propagate': True,
        },
    },
}
```

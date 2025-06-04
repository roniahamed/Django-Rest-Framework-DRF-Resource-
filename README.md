This tutorial will guide you step-by-step, explain the "why" behind each action, and point out common pitfalls.

# Mastering Django: Email as Username with a CustomUser Model

Welcome! If you're looking to customize Django's authentication system, particularly by using email as the primary identifier instead of a username, you've come to the right place. This guide will walk you through the entire process, from understanding the basics to implementing a robust CustomUser model.

## Table of Contents

1.  [Introduction](#1-introduction)
    *   [What is a CustomUser model in Django?](#what-is-a-customuser-model-in-django)
    *   [Why and when do we need to use a CustomUser model?](#why-and-when-do-we-need-to-use-a-customuser-model)
2.  [How Django Handles Authentication by Default](#2-how-django-handles-authentication-by-default)
    *   [What changes when we override the User model?](#what-changes-when-we-override-the-user-model)
3.  [Step-by-Step Guide to Creating a CustomUser Model](#3-step-by-step-guide-to-creating-a-customuser-model)
    *   [Step 0: Create a Dedicated App for Users](#step-0-create-a-dedicated-app-for-users)
    *   [Step 1: Define Your CustomUser Model (Extending `AbstractUser`)](#step-1-define-your-customuser-model-extending-abstractuser)
    *   [Step 2: Create a Custom User Manager](#step-2-create-a-custom-user-manager)
    *   [Step 3: Update `settings.py` with `AUTH_USER_MODEL`](#step-3-update-settingspy-with-auth_user_model)
    *   [Step 4: Apply Migrations](#step-4-apply-migrations)
    *   [Step 5: Create a Superuser (with Email)](#step-5-create-a-superuser-with-email)
    *   [Step 6: Register CustomUser Model in Admin](#step-6-register-customuser-model-in-admin)
    *   [Step 7: Test Your CustomUser Model](#step-7-test-your-customuser-model)
4.  [Common Mistakes and How to Avoid Them](#4-common-mistakes-and-how-to-avoid-them)
5.  [Best Practices for Using a CustomUser in Real Projects](#5-best-practices-for-using-a-customuser-in-real-projects)
6.  [Bonus Section: Integrating with DRF Authentication Libraries](#6-bonus-section-integrating-with-drf-authentication-libraries)
    *   [Using with `dj-rest-auth`](#using-with-dj-rest-auth)
    *   [Using with `djoser`](#using-with-djoser)
7.  [Conclusion](#7-conclusion)

---

## 1. Introduction

### What is a CustomUser model in Django?

Django comes with a built-in `User` model (`django.contrib.auth.models.User`) that handles authentication and authorization. It includes fields like `username`, `password`, `email`, `first_name`, and `last_name`.

A **CustomUser model** is a model that you, the developer, create to replace Django's default `User` model. This allows you to tailor the user-related fields and authentication behavior precisely to your application's needs.

### Why and when do we need to use a CustomUser model?

While the default User model is convenient for many projects, there are several scenarios where a CustomUser model becomes necessary or highly beneficial:

1.  **Email as the Primary Identifier:** Many modern applications prefer using an email address for login instead of a separate username. This is the main focus of our tutorial.
2.  **Adding Extra Profile Information:** You might want to store additional information directly on the User model, such as `date_of_birth`, `phone_number`, `profile_picture_url`, `bio`, or `subscription_type`. While a separate Profile model (linked via OneToOneField) is an option, sometimes integrating these directly into the User model is simpler for core attributes.
3.  **Removing Default Fields:** Perhaps you don't need `first_name` or `last_name`.
4.  **Modifying Field Properties:** You might want to change the `max_length` of a field or make certain fields required that aren't by default.

**Crucial Advice:** If you decide you need a CustomUser model, **do it at the very beginning of your project**, before running your first `migrate` command. Changing it mid-project is complex and can lead to database headaches.

---

## 2. How Django Handles Authentication by Default

Django's default authentication system revolves around the `django.contrib.auth.models.User` model. This model uses `username` as its unique identifier for login (the `USERNAME_FIELD`). It also has an `email` field, but by default, it's not unique and not used for login.

When a user tries to log in, Django's authentication backend typically attempts to find a user with the provided `username` and then verifies the password.

### What changes when we override the User model?

When you implement a CustomUser model and tell Django to use it (via the `AUTH_USER_MODEL` setting):

1.  **Django uses your model:** All Django components that refer to the user (admin, authentication backends, forms, permissions) will now use your CustomUser model.
2.  **`USERNAME_FIELD` dictates login:** You'll define which field acts as the unique identifier for login. In our case, this will be `email`.
3.  **Manager methods become critical:** Methods for creating users (like `create_user` and `createsuperuser`) will need to be adapted to work with your new `USERNAME_FIELD`.

---

## 3. Step-by-Step Guide to Creating a CustomUser Model

Let's get our hands dirty! We'll create a CustomUser model that uses `email` as the login identifier, removes the `username` field, and adds a couple of custom fields.

### Step 0: Create a Dedicated App for Users

It's good practice to manage your user model in a dedicated app. Let's call it `accounts`. If you already have an app for user-related logic, you can use that.

```bash
python manage.py startapp accounts
```

Don't forget to add this new app to `INSTALLED_APPS` in your project's `settings.py`:

```python
# project/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Your other apps
    'accounts', # Add your new app here
]
```

### Step 1: Define Your CustomUser Model (Extending `AbstractUser`)

Django provides two base classes for creating custom user models: `AbstractBaseUser` and `AbstractUser`.

*   `AbstractBaseUser`: Gives you full control but requires you to implement all authentication-related fields and methods (password management, active status, staff status, etc.). More work, but maximum flexibility.
*   `AbstractUser`: Inherits from `AbstractBaseUser` and provides the default fields (like `first_name`, `last_name`, `is_staff`, `is_active`, `date_joined`) and methods of Django's `User` model, minus `username` (which you can add back if needed, or leave out). This is often the easier starting point.

We'll use `AbstractUser` because it provides a lot of boilerplate for free.

Open `accounts/models.py` and add the following:

```python
# accounts/models.py

from django.contrib.auth.models import AbstractUser
from django.db import models
from django.utils.translation import gettext_lazy as _

class CustomUser(AbstractUser):
    # Remove the username field
    username = None

    # Make email the unique identifier and a required field
    email = models.EmailField(_('email address'), unique=True)

    # Add your custom fields here
    # For example:
    date_of_birth = models.DateField(null=True, blank=True)
    phone_number = models.CharField(max_length=20, blank=True, null=True)
    profile_picture = models.ImageField(upload_to='profile_pics/', null=True, blank=True)
    bio = models.TextField(max_length=500, blank=True, null=True)

    # Set email as the USERNAME_FIELD
    USERNAME_FIELD = 'email'
    # Define required fields for createsuperuser command (email and password are required by default)
    REQUIRED_FIELDS = ['first_name', 'last_name'] # Add other fields you want to be prompted for

    def __str__(self):
        return self.email
```

**Explanation:**

*   `from django.contrib.auth.models import AbstractUser`: We import the base class.
*   `from django.utils.translation import gettext_lazy as _`: For internationalization of field labels, a good practice.
*   `class CustomUser(AbstractUser):`: Our model inherits from `AbstractUser`.
*   `username = None`: This is crucial. We explicitly remove the `username` field provided by `AbstractUser`.
*   `email = models.EmailField(_('email address'), unique=True)`: We redefine the `email` field. `AbstractUser` has an email field, but it's not unique by default. We make it `unique=True` because it will be our primary identifier.
*   **Custom Fields:**
    *   `date_of_birth = models.DateField(null=True, blank=True)`: A field to store the user's birth date. `null=True` allows the database to store NULL values, and `blank=True` allows the field to be empty in forms.
    *   `phone_number = models.CharField(max_length=20, blank=True, null=True)`: A character field for a phone number.
    *   `profile_picture = models.ImageField(upload_to='profile_pics/', null=True, blank=True)`: For user avatars. You'll need `Pillow` installed (`pip install Pillow`) and `MEDIA_ROOT` and `MEDIA_URL` configured in `settings.py` for image uploads.
    *   `bio = models.TextField(max_length=500, blank=True, null=True)`: A text field for a short user biography.
*   `USERNAME_FIELD = 'email'`: This tells Django to use the `email` field for authentication purposes (i.e., what the user will use to log in).
*   `REQUIRED_FIELDS = ['first_name', 'last_name']`: This list specifies which fields (besides the `USERNAME_FIELD` and `password`) will be prompted for when creating a superuser via the `createsuperuser` command. `email` should NOT be in this list as it's already designated by `USERNAME_FIELD`.
*   `def __str__(self): return self.email`: A string representation for the model, useful in the Django admin and debugging.

### Step 2: Create a Custom User Manager

Because we've removed the `username` field and `AbstractUser`'s default manager expects it, we need to create a custom manager that knows how to create users and superusers using `email`.

Create a new file `accounts/managers.py`:

```python
# accounts/managers.py

from django.contrib.auth.models import BaseUserManager
from django.utils.translation import gettext_lazy as _

class CustomUserManager(BaseUserManager):
    """
    Custom user model manager where email is the unique identifier
    for authentication instead of usernames.
    """
    def create_user(self, email, password, **extra_fields):
        """
        Create and save a User with the given email and password.
        """
        if not email:
            raise ValueError(_('The Email must be set'))
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password, **extra_fields):
        """
        Create and save a SuperUser with the given email and password.
        """
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        extra_fields.setdefault('is_active', True)

        if extra_fields.get('is_staff') is not True:
            raise ValueError(_('Superuser must have is_staff=True.'))
        if extra_fields.get('is_superuser') is not True:
            raise ValueError(_('Superuser must have is_superuser=True.'))
        return self.create_user(email, password, **extra_fields)
```

**Explanation:**

*   `from django.contrib.auth.models import BaseUserManager`: We import the base manager class.
*   `create_user(self, email, password, **extra_fields)`:
    *   This method is used to create regular users.
    *   It takes `email`, `password`, and any `extra_fields` (like `first_name`, `last_name`).
    *   It normalizes the email (converts domain part to lowercase).
    *   It sets the password using `user.set_password()` (which handles hashing).
    *   It saves the user to the database.
*   `create_superuser(self, email, password, **extra_fields)`:
    *   This method is used by the `createsuperuser` management command.
    *   It sets `is_staff`, `is_superuser`, and `is_active` to `True` by default.
    *   It includes basic validation for these fields.
    *   It then calls `create_user` to actually create the user.

Now, hook this manager into your `CustomUser` model. Modify `accounts/models.py`:

```python
# accounts/models.py

from django.contrib.auth.models import AbstractUser
from django.db import models
from django.utils.translation import gettext_lazy as _
from .managers import CustomUserManager # Import the custom manager

class CustomUser(AbstractUser):
    username = None
    email = models.EmailField(_('email address'), unique=True)
    date_of_birth = models.DateField(null=True, blank=True)
    phone_number = models.CharField(max_length=20, blank=True, null=True)
    profile_picture = models.ImageField(upload_to='profile_pics/', null=True, blank=True)
    bio = models.TextField(max_length=500, blank=True, null=True)

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['first_name', 'last_name']

    objects = CustomUserManager() # Tell Django to use this manager

    def __str__(self):
        return self.email
```

**Explanation:**

*   `from .managers import CustomUserManager`: We import our newly created manager.
*   `objects = CustomUserManager()`: This line is crucial. It tells Django to use `CustomUserManager` for operations on `CustomUser` objects (like `CustomUser.objects.create_user(...)`).

### Step 3: Update `settings.py` with `AUTH_USER_MODEL`

Now, you need to tell Django to use your `CustomUser` model instead of the default one. This is done by setting the `AUTH_USER_MODEL` variable in your project's `settings.py` file.

**IMPORTANT:** This step *must* be done before you run `makemigrations` or `migrate` for the first time, or for the app containing your custom user model.

```python
# project/settings.py

# ... other settings ...

AUTH_USER_MODEL = 'accounts.CustomUser' # 'app_label.ModelName'

# ... other settings ...
```

**Explanation:**

*   `AUTH_USER_MODEL = 'accounts.CustomUser'`: This setting tells Django to use the `CustomUser` model (defined in the `accounts` app) as the authentication user model for this project.

### Step 4: Apply Migrations

With the model and settings in place, it's time to create and apply database migrations.

1.  **Delete existing migrations (if any, and ONLY if you haven't run `migrate` yet for your user app or `django.contrib.auth`):**
    If this is a brand-new project, or you haven't run `migrate` yet, you can skip this. If you accidentally ran `makemigrations` before setting `AUTH_USER_MODEL`, you might need to delete the migration files in your `accounts/migrations/` directory (except `__init__.py`) and potentially clear the `django_migrations` table entries related to `auth` and `accounts` if you're in early development and can reset the DB. **Be very careful with this step on existing projects with data.**

2.  **Create migrations for your `accounts` app:**

    ```bash
    python manage.py makemigrations accounts
    ```
    You should see output indicating that a migration file was created for the `CustomUser` model.

3.  **Apply the migrations to the database:**

    ```bash
    python manage.py migrate
    ```
    This will create the necessary tables in your database, including the table for your `CustomUser` model and update other tables to reference it.

### Step 5: Create a Superuser (with Email)

Now, let's test if the `createsuperuser` command works with our email-centric setup.

```bash
python manage.py createsuperuser
```

You should be prompted for:

*   Email: (This is your `USERNAME_FIELD`)
*   First name: (Because it's in `REQUIRED_FIELDS`)
*   Last name: (Because it's in `REQUIRED_FIELDS`)
*   Password:

Enter the details, and your superuser should be created. Notice it doesn't ask for a `username`!

### Step 6: Register CustomUser Model in Admin

To manage your custom users through the Django admin interface, you need to register your `CustomUser` model. Django's default `UserAdmin` is quite good, and it generally adapts well to custom user models that inherit from `AbstractUser`.

Open `accounts/admin.py` and add:

```python
# accounts/admin.py

from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import CustomUser

class CustomUserAdmin(UserAdmin):
    model = CustomUser
    # Define which fields to display in the list view
    list_display = ['email', 'first_name', 'last_name', 'is_staff', 'is_active', 'date_of_birth', 'phone_number']
    # Define which fields to use for searching
    search_fields = ['email', 'first_name', 'last_name']
    # Define which fields to use for filtering
    list_filter = ['is_staff', 'is_active', 'groups']

    # Customize the fieldsets for the add/change forms
    # This is important to include your custom fields
    fieldsets = (
        (None, {'fields': ('email', 'password')}),
        ('Personal info', {'fields': ('first_name', 'last_name', 'date_of_birth', 'phone_number', 'profile_picture', 'bio')}),
        ('Permissions', {'fields': ('is_active', 'is_staff', 'is_superuser', 'groups', 'user_permissions')}),
        ('Important dates', {'fields': ('last_login', 'date_joined')}),
    )
    # For the 'add user' form, if you have fields not in AbstractUser that are required
    add_fieldsets = UserAdmin.add_fieldsets + (
        (None, {'fields': ('date_of_birth', 'phone_number', 'profile_picture', 'bio')}), # Add your custom fields here
    )

admin.site.register(CustomUser, CustomUserAdmin)
```

**Explanation:**

*   `from django.contrib.auth.admin import UserAdmin`: We import the standard `UserAdmin` class.
*   `class CustomUserAdmin(UserAdmin):`: We create our own admin class inheriting from `UserAdmin`. This allows us to customize how `CustomUser` is displayed and managed.
*   `list_display`: Controls which fields appear in the list view of users.
*   `search_fields`: Enables searching by these fields.
*   `list_filter`: Adds filters to the sidebar.
*   `fieldsets`: This is a key customization. `UserAdmin.fieldsets` defines the layout of the user edit page. We override it to include our custom fields (`date_of_birth`, `phone_number`, `profile_picture`, `bio`). If you don't do this, your custom fields won't appear on the admin change form.
*   `add_fieldsets`: Similarly, this customizes the "add user" form. We append our custom fields to the default `add_fieldsets`.
*   `admin.site.register(CustomUser, CustomUserAdmin)`: Registers our model with our custom admin options.

### Step 7: Test Your CustomUser Model

1.  **Run the development server:**
    ```bash
    python manage.py runserver
    ```
2.  **Access the Django Admin:**
    Navigate to `http://127.0.0.1:8000/admin/` in your browser.
3.  **Log in:**
    Use the email and password of the superuser you created in Step 5.
4.  **Manage Users:**
    You should see "Custom users" (or similar, depending on your model's `verbose_name_plural`) in the admin dashboard under your "ACCOUNTS" app. Click on it.
    *   Verify you can see the superuser you created.
    *   Try adding a new user. You should see your custom fields (`date_of_birth`, `phone_number`, etc.) in the form.
    *   Try editing an existing user.

5.  **Test in Django Shell (Optional but good):**
    ```bash
    python manage.py shell
    ```
    Then in the shell:
    ```python
    from accounts.models import CustomUser
    from django.contrib.auth import get_user_model

    User = get_user_model() # Good practice to get the active user model

    # Create a new user
    new_user = User.objects.create_user(
        email='testuser@example.com',
        password='somepassword123',
        first_name='Test',
        last_name='User',
        date_of_birth='1990-01-01'
    )
    print(new_user.email)
    print(new_user.date_of_birth)

    # Try to fetch the user
    fetched_user = User.objects.get(email='testuser@example.com')
    print(fetched_user.is_active)

    # Attempt to create a user without an email (should fail due to manager validation)
    try:
        User.objects.create_user(email=None, password='badpassword')
    except ValueError as e:
        print(f"Error: {e}")
    ```

If all these steps work without errors, congratulations! You've successfully implemented a CustomUser model with email as the username.

---

## 4. Common Mistakes and How to Avoid Them

1.  **Setting `AUTH_USER_MODEL` too late:**
    *   **Mistake:** Running `makemigrations` or `migrate` *before* setting `AUTH_USER_MODEL = 'app_name.ModelName'` in `settings.py`.
    *   **Consequence:** Django will create tables for the default `auth.User` model. Later, when you try to switch, you'll face migration conflicts and potential issues with foreign keys pointing to the wrong user model.
    *   **Avoidance:** Always set `AUTH_USER_MODEL` at the very beginning of your project, before the first migration concerning authentication or your user app. If you're past this point, it's complex to fix and might involve manual database alterations or tricky data migrations (not recommended for beginners). Starting fresh is often easier.

2.  **Forgetting to create a Custom User Manager (or not assigning it):**
    *   **Mistake:** Removing `username` from `AbstractUser` but not providing a `CustomUserManager` that handles `email` in `create_user` and `create_superuser`. Or creating it but forgetting `objects = CustomUserManager()` in the model.
    *   **Consequence:** The `createsuperuser` command will likely fail because the default manager it inherits will look for a `username` field.
    *   **Avoidance:** Always create and assign a custom manager when you significantly alter the identification fields or user creation logic, especially when removing `username`.

3.  **Incorrect `USERNAME_FIELD` or `REQUIRED_FIELDS`:**
    *   **Mistake:** Setting `USERNAME_FIELD` to a non-unique field. Or including the `USERNAME_FIELD` (e.g., `email`) in the `REQUIRED_FIELDS` list.
    *   **Consequence:** Authentication issues, or redundant prompts during `createsuperuser`.
    *   **Avoidance:** Ensure your `USERNAME_FIELD` (e.g., `email`) has `unique=True` in its model definition. The `USERNAME_FIELD` is implicitly required, so it should not be listed in `REQUIRED_FIELDS`.

4.  **Not making the `email` field unique:**
    *   **Mistake:** Defining `email = models.EmailField()` without `unique=True` when `USERNAME_FIELD = 'email'`.
    *   **Consequence:** You could have multiple users with the same email, which breaks the concept of email as a unique identifier for login. Django will likely throw an error during `createsuperuser` or login if duplicates exist.
    *   **Avoidance:** Always set `unique=True` on the field designated as `USERNAME_FIELD`.

5.  **Migration issues after initial setup:**
    *   **Mistake:** Trying to change `AUTH_USER_MODEL` after significant development and data creation.
    *   **Consequence:** This is a major change affecting many database relationships. It can lead to broken foreign keys and data integrity issues.
    *   **Avoidance:** Decide on your user model structure early. If a change is absolutely necessary later, it requires careful planning, data migration scripts, and thorough testing.

6.  **Custom fields not appearing in Django Admin:**
    *   **Mistake:** Adding custom fields to `CustomUser` but not updating `fieldsets` and `add_fieldsets` in `CustomUserAdmin`.
    *   **Consequence:** Your custom fields won't be visible or editable in the Django admin interface.
    *   **Avoidance:** Always customize `UserAdmin`'s `fieldsets` and `add_fieldsets` to include any new fields you've added to your user model.

---

## 5. Best Practices for Using a CustomUser in Real Projects

1.  **Implement Early:** As stressed before, set up your CustomUser model at the absolute start of your project.
2.  **Dedicated App:** Keep your user model and related authentication logic (managers, forms, etc.) in a dedicated app (e.g., `accounts`, `users`).
3.  **Use `get_user_model()`:** When you need to refer to your user model in other parts of your Django project (e.g., in other apps' `models.py` for ForeignKey relationships, or in views), don't import it directly. Instead, use `django.contrib.auth.get_user_model()`:
    ```python
    from django.conf import settings
    from django.db import models
    from django.contrib.auth import get_user_model

    User = get_user_model() # This will return your CustomUser model

    class Post(models.Model):
        author = models.ForeignKey(
            User, # Use the result of get_user_model()
            on_delete=models.CASCADE
        )
        title = models.CharField(max_length=200)
        # ...
    ```
    Or, if you need to specify it as a string (e.g., in `ForeignKey` in migrations or older Django versions):
    ```python
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    ```
    This makes your code more robust and less dependent on the specific app/model name.
4.  **Consider a Separate Profile Model:** If you find yourself adding many non-authentication-related fields to your `CustomUser` model (e.g., extensive profile details, user preferences, social media links), consider creating a separate `Profile` model linked to your `CustomUser` model with a `OneToOneField`. This keeps your User model focused on authentication and core identity.
    ```python
    # In accounts/models.py or a new profiles/models.py
    from django.db import models
    from django.conf import settings

    class Profile(models.Model):
        user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
        address = models.CharField(max_length=255, blank=True)
        # ... other profile fields
    ```
5.  **Thoroughly Test Authentication Flow:** After setting up your CustomUser model, test registration, login, logout, password reset, and superuser creation thoroughly.
6.  **Forms and Serializers:** Remember that default Django forms (like `AuthenticationForm`) or forms/serializers from third-party packages might need to be subclassed or configured to work correctly with `email` as the username field if they explicitly look for `username`. However, many modern packages adapt well if `USERNAME_FIELD` is set.

---

## 6. Bonus Section: Integrating with DRF Authentication Libraries

If you're building an API with Django Rest Framework (DRF), you'll likely use libraries like `dj-rest-auth` or `djoser` for authentication endpoints. Good news: they generally work well with custom user models!

### Using with `dj-rest-auth`

`dj-rest-auth` is designed to work seamlessly with Django's authentication system, including custom user models.

1.  **Installation & Setup:**
    Follow the `dj-rest-auth` installation guide.
    ```bash
    pip install dj-rest-auth
    ```
    Add `rest_framework`, `rest_framework.authtoken`, and `dj_rest_auth` to `INSTALLED_APPS`. If you want social auth or registration, add `dj_rest_auth.registration` as well.

2.  **Configuration:**
    Typically, no special configuration is needed for `dj-rest-auth` to recognize your `CustomUser` model as long as `AUTH_USER_MODEL` is correctly set in `settings.py`. It uses `get_user_model()`.

3.  **Serializers:**
    `dj-rest-auth` provides default serializers. If your `CustomUser` model has extra fields you want to include in registration or user detail responses, you might need to create custom serializers and configure `dj-rest-auth` to use them.

    For example, to include `date_of_birth` and `phone_number` in the user details endpoint:
    ```python
    # accounts/serializers.py
    from rest_framework import serializers
    from dj_rest_auth.serializers import UserDetailsSerializer
    from .models import CustomUser # Or get_user_model()

    class CustomUserDetailsSerializer(UserDetailsSerializer):
        class Meta(UserDetailsSerializer.Meta):
            model = CustomUser # Or get_user_model()
            fields = UserDetailsSerializer.Meta.fields + ('date_of_birth', 'phone_number', 'profile_picture', 'bio') # Add your custom fields
            read_only_fields = ('email',) # email might be part of UserDetailsSerializer.Meta.fields already

    # In project/settings.py
    REST_AUTH = {
        'USER_DETAILS_SERIALIZER': 'accounts.serializers.CustomUserDetailsSerializer',
        # Potentially custom registration serializer too
        # 'REGISTER_SERIALIZER': 'accounts.serializers.CustomRegisterSerializer',
    }
    ```
    For registration, you'd create a `CustomRegisterSerializer` inheriting from `dj_rest_auth.registration.serializers.RegisterSerializer` and add your fields.

### Using with `djoser`

`djoser` is another popular library for REST API authentication endpoints. It also adapts well.

1.  **Installation & Setup:**
    Follow the `djoser` installation guide.
    ```bash
    pip install djoser
    ```
    Add `djoser` to `INSTALLED_APPS`.

2.  **Configuration:**
    `djoser` will automatically use your `AUTH_USER_MODEL`.

3.  **Serializers:**
    `djoser` allows you to specify custom serializers for various actions (user creation, user details, etc.) through its settings.

    ```python
    # project/settings.py
    DJOSER = {
        'USER_ID_FIELD': 'id', # Or 'pk'
        'LOGIN_FIELD': 'email', # Crucial: tell Djoser to use email for login
        'USER_CREATE_PASSWORD_RETYPE': True,
        'PASSWORD_RESET_CONFIRM_RETYPE': True,
        'SERIALIZERS': {
            'user_create': 'accounts.serializers.CustomUserCreateSerializer', # Your custom serializer
            'user': 'accounts.serializers.CustomUserSerializer',             # For user details
            'current_user': 'accounts.serializers.CustomUserSerializer',
            # ... other serializers if needed
        },
        # ... other Djoser settings
    }
    ```
    You would then create `CustomUserCreateSerializer` and `CustomUserSerializer` in `accounts/serializers.py`, inheriting from `djoser.serializers.UserCreateSerializer` and `djoser.serializers.UserSerializer` respectively, and add your custom fields to their `Meta.fields`.

    Example `CustomUserSerializer`:
    ```python
    # accounts/serializers.py
    from djoser.serializers import UserSerializer as BaseUserSerializer
    from django.contrib.auth import get_user_model

    User = get_user_model()

    class CustomUserSerializer(BaseUserSerializer):
        class Meta(BaseUserSerializer.Meta):
            model = User
            fields = ('id', 'email', 'first_name', 'last_name', 'date_of_birth', 'phone_number', 'profile_picture', 'bio') # Add your custom fields
    ```
    And `CustomUserCreateSerializer`:
    ```python
    # accounts/serializers.py
    from djoser.serializers import UserCreateSerializer as BaseUserCreateSerializer
    from django.contrib.auth import get_user_model

    User = get_user_model()

    class CustomUserCreateSerializer(BaseUserCreateSerializer):
        class Meta(BaseUserCreateSerializer.Meta):
            model = User
            fields = ('id', 'email', 'password', 'first_name', 'last_name', 'date_of_birth', 'phone_number', 'profile_picture', 'bio') # Add your custom fields
    ```

Both libraries are powerful and flexible. Refer to their official documentation for more advanced customization.

---

## 7. Conclusion

Phew! That was a comprehensive journey. You've learned:

*   Why and when a CustomUser model is beneficial.
*   How to extend `AbstractUser` to remove `username` and use `email` as the primary identifier.
*   The importance of a `CustomUserManager` for handling user creation.
*   How to correctly configure `AUTH_USER_MODEL` and manage migrations.
*   How to integrate your `CustomUser` model with the Django admin.
*   Common pitfalls and best practices to ensure a smooth implementation.
*   A glimpse into using your `CustomUser` model with popular DRF authentication libraries.

**Suggestions for Beginners:**

*   **Start Fresh:** If you're new to this, try implementing a CustomUser model in a brand-new Django project first to get a feel for the process without worrying about existing data or migrations.
*   **Follow Steps Carefully:** The order of operations, especially setting `AUTH_USER_MODEL` before migrations, is critical.
*   **Test Incrementally:** Test after each major step (e.g., after creating the model, after creating the superuser, after configuring the admin).
*   **Read Django Docs:** The official Django documentation on [Customizing authentication](https://docs.djangoproject.com/en/stable/topics/auth/customizing/#substituting-a-custom-user-model) is an excellent resource.

Using a CustomUser model gives you significant power to shape your Django application's authentication system to your exact needs. While it requires careful setup, the flexibility it offers is often well worth the effort for real-world projects.

Happy coding!
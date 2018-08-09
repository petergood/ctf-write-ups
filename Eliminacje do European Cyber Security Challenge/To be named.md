# To be named
Problem: https://hack.cert.pl/challenge/web-to-be-named

In this challenge we are given a simple website where we can save notes. Users can create new accounts, login, browse their notes and create new ones. It turns out that the most important endpoint, which we will use to solve the problem, is the `note` endpoint, which returns a note given it's title.

When a user logs in the server sets two cookies, `csrftoken` and `sessionid`. If one sends a POST request to `note` with `csrftoken` set to an arbitrary value, an error will be displayed.

![error](https://i.imgur.com/FMIT9rR.png)

Based on the design (and the `Debug=True` message) of the error page it is safe to assume that the application is powered by Django.

Requests to `note` contain a single parameter, `name`. A simple directory traversal attack reveals that the notes are actually stored in files without extensions, and the `name` parameter is not sanitised.

![dir traversal](https://i.imgur.com/PBdGoes.png)

The next step will be to view the contents of the most important Django project files. A typical Django project has the following structure:
+ /Project_name
  + /Project_name
    + settings.py
    + urls.py
    + wsgi.py
  + manage.py
  + /app_name
    + admin.py
    + views.py
    + ...

The most interesting files for solving the challenge are `views.py`, which contains the the app's route handlers, and `settings.py`, which contains the various Django settings.

Unfortunately we know neither the `Project_name` nor the `app_name`, so we will have to guess them. After sending a few requests I discovered that the `Project_name` is `app`, and the `app_name` is `notes`.

Here are a few interesting snippets from `views.py`:

```python
def get_notes_directory(username):
    return 'notes_upload/{}/'.format(hashlib.sha256(username.encode('utf-8')).hexdigest())


def is_admin(request):
    return request.user.username == 'admin'

[...]

@login_required
@require_POST
def note(request):
    name = request.POST.get('name')
    try:
        with open(get_notes_directory(request.user.username) + name, 'rb') as f:
            return HttpResponse(f.read())
    except (FileNotFoundError, IOError):
        raise Http404()
```

It turns out that users's notes are stored in directories that correspond to the SHA256 hash of their username. For example, the path where the administrator's (`admin`) notes are stored is `notes_upload/8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918/`. The flag is most likely located in that directory, however, we do not know it's name.

```python
@login_required
def notes(request):
    notes = listdir(get_notes_directory(request.user.username))
    return render(request, 'notes/notes_list.html', {'notes': notes})
```
We see that a simple listdir (`ls`) recovers the names of the notes of a given user, however, the username is taken from the session, so we will have to figure out a way to trick the server into thinking that our username is `admin`.

Let's have a look at `settings.py`:
```python
SECRET_KEY = os.environ["I_LIKE_TRAINS"]

[...]

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}

SESSION_ENGINE = 'django.contrib.sessions.backends.signed_cookies'
```

We now know that this application is using signed cookies for session authentication, which means that a JSON object is serialised (pickled) and then cryptographically signed using the `SECRET_KEY`. If we can recover the `SECRET_KEY`, we will be able to forge session cookies and potentially gain access to the admin's account. We also see that the app is using SQLLite as it's database, this will be important later on.

The next step will be to recover the `SECRET_KEY`, which means we have to find the value of the `I_LIKE_TRAINS` environmental variable. After trying the `.bashrc`, `.bash_profile` and `.profile` files of the accounts in `/etc/passwd` and finding that they do not exist (most probably the environmental variables are set at runtime as the app is launched in a secure container), I turned to the magical resources of `/proc`. Accessing `/proc/1/environ`, which contains the environmental variables passed into the application with a `PID` of 1, revealed the value of `I_LIKE_TRAINS`.

![Ilovetrains](https://i.imgur.com/rNPzplF.png)

Now that we know the value of the SECRET_KEY, let's try deserialising a session cookie (using `django.core.signing`):

![fail](https://i.imgur.com/bBZbcV1.png)

As you can see, `signing.loads` throws a `BadSignature` exception, meaning that something is not right. To find out what is wrong we have to dive into Django's source code to see how session cookies are created. `django/contrib/sessions/backends/signed_cookies.py` reveals the problem:

```python
def _get_session_key(self):
        return signing.dumps(
            self._session, compress=True,
            salt='django.contrib.sessions.backends.signed_cookies',
            serializer=self.serializer,
        )

```

As we can see, the session cookies are additionally salted. If we include the same salt when deserialising the session cookie, everything works correctly:

![yay](https://i.imgur.com/TLPJqBo.png)

This reveals that the session cookie contains the users's unique ID (presumably changing it will allow us to access other user accounts), a `_auth_user_backend` value which does not really concern us, and a `_auth_user_hash`, which we will also have to modify to access a different account. Once more we resort to Django's source code to find out what `_auth_user_hash` is, or more importantly, what is it a hash of.

(fragments of [django/contrib/auth/\__init__.py](https://github.com/django/django/blob/master/django/contrib/auth/__init__.py))
```python
HASH_SESSION_KEY = '_auth_user_hash'
[...]

def login(request, user, backend=None):
  [...]
  session_auth_hash = ''
    if user is None:
        user = request.user
    if hasattr(user, 'get_session_auth_hash'):
        session_auth_hash = user.get_session_auth_hash()

  [...]

  request.session[HASH_SESSION_KEY] = session_auth_hash
```

`_auth_user_hash` is the value returned by `user.get_session_auth_hash()`. [Django/contrib/auth/base_user.py](https://github.com/django/django/blob/master/django/contrib/auth/base_user.py) finally reveals what we are dealing with:

```python
def get_session_auth_hash(self):
        """
        Return an HMAC of the password field.
        """
        key_salt = "django.contrib.auth.models.AbstractBaseUser.get_session_auth_hash"
        return salted_hmac(key_salt, self.password).hexdigest()
```

`_auth_user_hash` is actually the HMAC hash of the hash of the user's password. If we want to gain access to the administrator's account, we will have to find his password hash. This is where the SQLLite database steps in. After downloading it (using CUrl's very handy --output flag) and viewing it we not only find the administrator's password hash, but also his user ID.

![db](https://i.imgur.com/mETyf6o.png)

Now all that is left to do is to generate a session cookie that belongs to the administrator. I wrote a simple script to do that:

```python
import django.core.signing as signing
import django
from django.utils.crypto import salted_hmac
from django.conf import settings

settings.configure()

settings.SECRET_KEY = '@cHu_ChU-chU!!!'
passwordHash = 'pbkdf2_sha256$100000$PciiyrqGTHyQ$2cO66Q/7vQMRysIWOlzkjv2dhf8FLHT5XWR/LjMxD3U='
authUserHash = salted_hmac("django.contrib.auth.models.AbstractBaseUser.get_session_auth_hash", passwordHash).hexdigest()

sessionObject = {
	'_auth_user_id': '1',
	'_auth_user_backend': 'django.contrib.auth.backends.ModelBackend',
	'_auth_user_hash': authUserHash
}

sessionCookieValue = signing.dumps(
	sessionObject,
	compress=True,
	salt='django.contrib.sessions.backends.signed_cookies'
)

print(sessionCookieValue)
```

After setting this new cookie using Chrome's Dev Tools and refreshing the page I gained access to the administrator's account and to the flag.

I really enjoyed solving `To be named`, as I learned a lot about how Django works under the hood. I am also pleased to say that nobody was injured by trains coming out of nowhere.

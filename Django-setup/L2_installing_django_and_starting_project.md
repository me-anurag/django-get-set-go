# Make sure venv is active first — you should see (venv) in terminal

# ---- INSTALL DJANGO ----
pip install django

# ---- CHECK DJANGO VERSION ----
python -m django --version
# or
django-admin --version

# ---- CREATE DJANGO PROJECT ----
django-admin startproject myproject .
#                          ^name    ^ dot = create IN current folder (recommended)

# WITHOUT the dot:
django-admin startproject myproject
# This creates an extra nested folder: myproject/myproject/  ← confusing

# ---- WHAT GETS CREATED ----
# myproject/        ← root (your folder)
# ├── manage.py
# └── myproject/
#     ├── __init__.py
#     ├── settings.py
#     ├── urls.py
#     ├── wsgi.py
#     └── asgi.py

# ---- RUN THE SERVER ----
python manage.py runserver
# Visit: http://127.0.0.1:8000  → you see the Django welcome page

# ---- RUN ON DIFFERENT PORT ----
python manage.py runserver 8080
python manage.py runserver 0.0.0.0:8000   # accessible on local network

# ---- STOP THE SERVER ----
# Press Ctrl + C in terminal
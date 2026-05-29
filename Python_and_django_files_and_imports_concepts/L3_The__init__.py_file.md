# An empty __init__.py just marks the folder as a package.
# But you can also USE it to expose shortcuts.

# myapp/__init__.py  (empty = fine, this is just to explain options)

# ---- EXAMPLE: expose things at the package level ----
# Suppose myapp/utils.py has a function called clean_data()
# Inside myapp/__init__.py you can write:
from .utils import clean_data

# Now anywhere in the project you can do:
from myapp import clean_data   # instead of from myapp.utils import clean_data
myapp/
├── models.py
├── views.py
├── utils.py
└── forms.py

# ---- views.py ----

# Import a model from the same app
from myapp.models import Article          # absolute import
from .models import Article               # relative import (dot = current package)

# Import a utility function from same app
from myapp.utils import format_date       # absolute
from .utils import format_date            # relative

# Import a form from same app
from myapp.forms import ArticleForm       # absolute
from .forms import ArticleForm            # relative

# Import multiple things at once
from .models import Article, Category, Tag

# Import everything from models (not recommended but valid)
from .models import *


#Tip: Inside an app, use relative imports (.models, .utils).
Between apps, use absolute imports (from otherapp.models import ...).
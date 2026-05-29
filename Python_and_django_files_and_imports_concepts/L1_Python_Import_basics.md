# --- ABSOLUTE IMPORT (most common) ---
import os
import sys
from pathlib import Path

# --- IMPORT SPECIFIC THINGS ---
from os import getcwd, listdir
from os.path import join, exists, dirname

# --- IMPORT WITH ALIAS ---
import numpy as np
import pandas as pd

# --- IMPORT EVERYTHING (avoid in large projects) ---
from os.path import *

# --- IMPORT YOUR OWN FILE ---
# If you have a file called utils.py in the same folder:
import utils
from utils import my_function

# --- CONDITIONAL IMPORT ---
try:
    import ujson as json   # faster json
except ImportError:
    import json            # fallback to standard
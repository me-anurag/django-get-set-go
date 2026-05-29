<!-- . (dot) in imports = "current app folder"
appname.file = cross-app import, always use full path
__init__.py = the file that makes a folder importable
settings.BASE_DIR = your project root, build all paths from it
Circular imports → use string 'appname.ModelName' or lazy import inside a function -->
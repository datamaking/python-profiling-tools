Here's a step-by-step guide to create a Django project with integrated Python cProfile profiling:

### Step 1: Create Project Structure
```bash
django-admin startproject django_cprofile_example
cd django_cprofile_example
```

### Step 2: Create Required Files

1. **profiling_middleware.py** (in project directory)
```python
import cProfile
import os
from datetime import datetime
from django.conf import settings

class ProfilingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        self.profile_dir = os.path.join(settings.BASE_DIR, 'profiles')
        os.makedirs(self.profile_dir, exist_ok=True)

    def __call__(self, request):
        profiler = cProfile.Profile()
        profiler.enable()

        try:
            response = self.get_response(request)
        finally:
            profiler.disable()
            timestamp = datetime.now().strftime('%Y%m%d-%H%M%S-%f')
            path = request.path.strip('/').replace('/', '_') or 'root'
            filename = f"{timestamp}_{request.method}_{path}.prof"
            filepath = os.path.join(self.profile_dir, filename)
            profiler.dump_stats(filepath)
        
        return response
```

2. **views.py** (in project directory)
```python
from django.http import HttpResponse

def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

def sample_view(request):
    n = int(request.GET.get('n', 30))  # Default to n=30
    result = fibonacci(n)
    return HttpResponse(f"Fibonacci({n}) = {result}")
```

3. **Update urls.py**
```python
from django.urls import path
from .views import sample_view

urlpatterns = [
    path('fibonacci/', sample_view, name='fibonacci'),
]
```

4. **Update settings.py**
```python
MIDDLEWARE = [
    'django_cprofile_example.profiling_middleware.ProfilingMiddleware',
    # ... other default middleware
]

# Add at the bottom
PROFILE_DIR = os.path.join(BASE_DIR, 'profiles')
```

### Step 3: Project Structure
```
django_cprofile_example/
├── django_cprofile_example/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── views.py
│   ├── profiling_middleware.py
│   └── wsgi.py
├── manage.py
└── profiles/ (auto-created)
```

### Step 4: Run and Test
1. Start the server:
```bash
python manage.py runserver
```

2. Generate profile data by accessing:
```
http://localhost:8000/fibonacci/?n=35
```

3. Profile files will be created in `profiles/` directory with names like:
```
20240314-153045_GET_fibonacci.prof
```

### Step 5: Analyze Profiling Results

**Using snakeviz (visual tool):**
```bash
pip install snakeviz
snakeviz profiles/20240314-153045_GET_fibonacci.prof
```

**Using pstats (command-line):**
```python
import pstats
p = pstats.Stats('path/to/profile.prof')
p.sort_stats('cumtime').print_stats(10)
```

### Key Features:
- Automatic profile creation for every request
- Unique filenames with timestamp, method, and path
- Profile storage in dedicated directory
- Middleware-based non-intrusive profiling
- Sample Fibonacci view for testing

### Notes:
1. Profile files can grow quickly - regularly clean the profiles directory
2. Remove middleware in production using environment variables
3. Use `n=30-40` for meaningful profiling results in the sample view
4. Middleware captures all requests - filter using request.path if needed

This implementation provides full-stack profiling of Django requests, helping identify performance bottlenecks in your application.

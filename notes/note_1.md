Below is a detailed guide on how to monitor and profile memory (RAM) and CPU usage for various types of Python applications—including NLP code, PySpark jobs, and Django API services—using open source tools and libraries. Common tools include:

- **cProfile:** Python’s built‐in CPU profiler.
- **memory_profiler:** A module for tracking memory usage line by line.
- **psutil:** A library for retrieving information on system utilization (CPU, memory, etc.).

Each section below includes an explanation and sample code.

---

## 1. Profiling Python NLP Code

When working on NLP tasks (e.g., text preprocessing, vectorization, or model inference), it’s useful to know how much CPU time and memory are being consumed. Two main approaches are:

### A. CPU Profiling with cProfile

`cProfile` helps you understand which functions take the most CPU time. For example:

```python
import cProfile
import pstats
import io

def process_text_corpus(corpus):
    # Simulate an NLP task (e.g., tokenization, vectorization, etc.)
    tokens = [doc.lower().split() for doc in corpus]
    # Simulate further processing (dummy delay)
    for doc in tokens:
        [word[::-1] for word in doc]
    return tokens

if __name__ == "__main__":
    corpus = [
        "The quick brown fox jumps over the lazy dog",
        "Never jump over the lazy dog quickly",
        "Bright foxes leap over lazy dogs in summer"
    ]
    
    # Run cProfile on the process_text_corpus function
    profiler = cProfile.Profile()
    profiler.enable()
    process_text_corpus(corpus)
    profiler.disable()

    # Print profiling results sorted by cumulative time
    s = io.StringIO()
    sortby = 'cumulative'
    ps = pstats.Stats(profiler, stream=s).sort_stats(sortby)
    ps.print_stats(10)
    print(s.getvalue())
```

**Explanation:**  
- The above code wraps the NLP function call with `cProfile` to record CPU time per function call.  
- The printed report highlights the most expensive functions by cumulative time.

### B. Memory Profiling with memory_profiler and psutil

To track memory usage line by line, you can use the `memory_profiler` module. You can also use `psutil` for a simple periodic snapshot.

#### Example with memory_profiler

Install it via pip if needed:

```bash
pip install memory_profiler
```

Then add the `@profile` decorator (run the script with `python -m memory_profiler script.py`):

```python
from memory_profiler import profile

@profile
def process_text_corpus(corpus):
    tokens = [doc.lower().split() for doc in corpus]
    # Simulate additional processing
    processed = []
    for doc in tokens:
        processed.append([word[::-1] for word in doc])
    return processed

if __name__ == "__main__":
    corpus = [
        "The quick brown fox jumps over the lazy dog",
        "Never jump over the lazy dog quickly",
        "Bright foxes leap over lazy dogs in summer"
    ]
    process_text_corpus(corpus)
```

**Explanation:**  
- The `@profile` decorator will print out line-by-line memory usage, which helps identify memory-intensive operations.

#### Example with psutil for a Continuous Snapshot

```python
import psutil
import time

def monitor_resources(duration=5):
    process = psutil.Process()
    for _ in range(duration):
        mem = process.memory_info().rss / (1024 * 1024)  # in MB
        cpu = process.cpu_percent(interval=1)
        print(f"Memory usage: {mem:.2f} MB, CPU usage: {cpu:.2f}%")
        time.sleep(1)

if __name__ == "__main__":
    monitor_resources()
```

**Explanation:**  
- This snippet prints the memory and CPU usage of the current process every second for 5 seconds.  
- It’s useful for long-running tasks where you want a real-time view of resource usage.

---

## 2. Profiling PySpark Code

For large-scale data processing with PySpark, you typically leverage Spark’s built-in monitoring and UI, but you can also use Python-based profiling.

### A. Using Spark’s Web UI

- **Spark UI:** When you run a Spark job, Spark provides a web interface (by default at `http://localhost:4040`) where you can see executor-level CPU and memory usage, job stages, and more.  
- **Metrics:** Spark’s configuration allows you to integrate with tools like Ganglia, Prometheus, or Graphite for detailed monitoring.

### B. Using Python Profiling Tools in the Spark Driver

If you need to monitor the driver’s resource usage (or the portion of code you control outside the Spark executors), you can use `psutil` and `cProfile`.

#### Example: Monitoring Resource Usage Around a Spark Job

```python
from pyspark.sql import SparkSession
import psutil
import time

def run_spark_job(spark):
    # A sample Spark job that counts a large range
    df = spark.range(0, 1000000)
    count = df.filter("id % 2 == 0").count()
    print("Count of even numbers:", count)

if __name__ == "__main__":
    spark = SparkSession.builder.appName("ProfilingPySpark").getOrCreate()
    process = psutil.Process()

    # Initial resource snapshot
    initial_mem = process.memory_info().rss / (1024 * 1024)
    print("Initial memory usage:", initial_mem, "MB")

    # Run the Spark job
    run_spark_job(spark)

    # Snapshot after running the job
    final_mem = process.memory_info().rss / (1024 * 1024)
    print("Memory usage after job:", final_mem, "MB")

    spark.stop()
```

**Explanation:**  
- This code prints the memory usage of the Spark driver before and after a job using `psutil`.  
- For executor metrics, refer to the Spark UI, which gives detailed insights into CPU and memory usage per task.

---

## 3. Profiling a Python Django API Service

Profiling web services requires capturing metrics per HTTP request. You can use middleware-based profilers or integrate dedicated tools.

### A. Profiling with a Custom cProfile Middleware

Create a middleware to profile each view’s CPU usage:

```python
# profile_middleware.py
import cProfile, pstats, io
from django.utils.deprecation import MiddlewareMixin

class ProfileMiddleware(MiddlewareMixin):
    def process_view(self, request, view_func, view_args, view_kwargs):
        self.pr = cProfile.Profile()
        self.pr.enable()
    
    def process_response(self, request, response):
        self.pr.disable()
        s = io.StringIO()
        ps = pstats.Stats(self.pr, stream=s).sort_stats('cumulative')
        ps.print_stats(10)  # Top 10 functions
        print("Profiling results for request {}: \n{}".format(request.path, s.getvalue()))
        return response
```

**How to Use:**  
- Add `ProfileMiddleware` to your `MIDDLEWARE` list in `settings.py`.  
- Every request will be profiled, and the top CPU-consuming functions will be printed to the console.

### B. Profiling Memory with django-debug-toolbar or django-silk

- **django-debug-toolbar:**  
  Provides panels with SQL queries, cache usage, and even some memory profiling for each request.  
- **django-silk:**  
  Offers in-depth profiling of Django views including request/response time, SQL queries, and memory usage.  
- **Installation:**  
  Both tools are free and open source; install via pip and follow their configuration guides.

**Example (django-debug-toolbar):**

1. Install with:
   ```bash
   pip install django-debug-toolbar
   ```
2. Add it to `INSTALLED_APPS` and include its middleware in your `settings.py`:
   ```python
   INSTALLED_APPS = [
       # ... other apps
       'debug_toolbar',
   ]
   MIDDLEWARE = [
       # ... other middleware
       'debug_toolbar.middleware.DebugToolbarMiddleware',
   ]
   ```
3. Include its URL configuration in your `urls.py`:
   ```python
   import debug_toolbar
   from django.urls import path, include

   urlpatterns = [
       path('__debug__/', include(debug_toolbar.urls)),
       # ... your other url patterns
   ]
   ```
4. Run your Django server and navigate to a page—the toolbar will appear, showing SQL queries, timing, and some memory information.

**Explanation:**  
- These tools provide real-time insights during development and help you spot performance bottlenecks without modifying your code extensively.

---

## Summary

- **For Python NLP Code:** Use **cProfile** for CPU and **memory_profiler/psutil** for memory to understand your function-level performance.
- **For PySpark Code:** Leverage Spark’s built-in Web UI for executor-level metrics, and use **psutil** to profile the driver if needed.
- **For Django API Services:** Use a custom **cProfile middleware** to profile each request and integrate tools like **django-debug-toolbar** or **django-silk** for a comprehensive view of performance (including memory and SQL queries).

These open source tools are widely used and free, and they help you identify bottlenecks and optimize resource usage in your Python applications.

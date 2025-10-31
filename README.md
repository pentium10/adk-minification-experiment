# ADK Cold Start Optimizer

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

An experimental build tool to dramatically reduce Python application cold start times on serverless platforms like Google Cloud Run. This project was born out of a real-world struggle with a **30+ second cold start** using the `google-adk` framework, which was tracked in [this GitHub issue](https://github.com/google/adk-python/issues/2433).

This tool implements a "profiling and minification" strategy to reduce that startup time by over 50%.

**Read the full story behind this discovery on Medium:** [Hacking ADK's Importer: How We Slashed 24-Second Cold Start in Half](https://medium.com/p/def058a7c8dc)

## Disclaimer

This is an **unofficial and experimental** workaround. It is a pragmatic hack designed to solve a specific performance problem. It works by modifying library code, which may have unintended side effects and could break between library versions. Use this at your own risk.

---

## The Problem

Modern AI frameworks like Google's ADK are incredibly powerful, but they can be very large. The primary cause of slow cold starts in serverless environments is the framework's design, which eagerly imports almost its entire codebase at application startup. This creates a deep import cascade, forcing the Python interpreter to load and parse hundreds of files before your code can even run.

## The Solution

The result is a deployment package where all import statements still succeed, but they import lazy loading empty, feather-light files. The import cascade's performance cost is eliminated without fighting Python's import system.

---

## Getting Started

### Prerequisites

*   Your application's dependencies (e.g., `google-adk`) installed in a virtual environment.
*   The `build.py` script from this repository.

### Step 1: Build the Minified Package

```bash
python build.py
```

Your `./build` directory now contains the minified version of the library, ready for deployment.

### Step 2: Docker & Cloud Run Integration

To use the minified package, you must tell your container to look in the `build` directory first. This is done by setting the `PYTHONPATH` environment variable in your `Dockerfile`.

```dockerfile
# Dockerfile
FROM python:3.10-slim
WORKDIR /app

# 1. Install dependencies (gets the original google-adk)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 2. Copy your build script
COPY build.py .

# 3. Run the build script to create the minified /app/build directory
RUN python build.py

# 4. CRITICAL: Prepend the /app/build directory to the PYTHONPATH.
# This makes Python find our minified version before the original in site-packages.
ENV PYTHONPATH /app/build:$PYTHONPATH

# 5. Copy your application source code
COPY . .

#rest of your Dockerfile
```

---

## Configuration

The `build.py` script can be configured to process multiple libraries and handle fragile parts of the code with care.

### `PACKAGES_TO_PROCESS`

This tells the script which libraries to minify. You can gain significant additional speed by targeting other large dependencies that are also imported at startup.

```python
# The list of top-level packages to apply the minification process to.
PACKAGES_TO_PROCESS = [
    "google.adk",
    "google.cloud",
    # Add other large libraries you suspect are slowing down startup
]
```

### `SPECIAL_HANDLING_DIRS`

This is a critical "Do Not Touch" list. Some parts of a library are too fragile for our aggressive `__init__.py` rewrite. This configuration tells the build script to use a gentler approach (shimming submodules instead of the `__init__.py`) for these specific directories.

```python
# --- Directories requiring gentle handling (__init__.py is untouched) ---
# Format: {"package.name": ["dir1", "dir2"]}
SPECIAL_HANDLING_DIRS = {
    "google.adk": [
        "cli",           # The CLI entry point has rigid import expectations.
        "models",        # The ModelRegistry relies on eager loading to discover all models.
    ],
    "google.cloud.aiplatform": [],  # Add specific dirs if you find fragile parts.
}
```

If you encounter errors after minification, adding the problematic directory to this list is the first step in debugging.


## Disclaimer

This is an **unofficial and experimental** workaround. It is a pragmatic hack designed to solve a specific performance problem. It works by modifying library code, which may have unintended side effects and could break between library versions. Use this at your own risk.

## Contributing

Found a bug or have an improvement? Please fork and fix on your agents! Pull requests are welcome.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
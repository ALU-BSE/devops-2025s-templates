# Use an official, lightweight base image
# 'python:3.10-slim' is much smaller than the full 'python:3.10'
FROM python:3.10-slim

# Set the working directory inside the container
# All subsequent commands (COPY, RUN, CMD) will run from /app
WORKDIR /app

# --- Caching Optimization ---
# 1. Copy ONLY the dependencies file first
COPY requirements.txt .

# 2. Install dependencies
# Docker caches this step. As long as requirements.txt doesn't
# change, Docker won't re-run 'pip install' on every build.
RUN pip install --no-cache-dir -r requirements.txt

# 3. Copy the rest of your application code
# This comes *after* installing dependencies. Now, if you only
# change your .py files (and not requirements.txt), Docker
# will use the cached layer from the step above and build much faster.
COPY . .

# Expose the port the app runs on *inside* the container
# This is documentation for the user and other tools.
EXPOSE 8000

# The command to run your application when the container starts
# We use 0.0.0.0 in app.py to listen on all interfaces.
CMD ["python", "app.py"]

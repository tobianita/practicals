## Container Restart Loop

**Problem:**  
Backend container kept restarting.

**Symptoms:**

`docker ps` showed container restarting

Application logs looked fine


**Root Cause:**  
Health check required `curl`, but base image didn’t include it.

**Fix:**  
Updated Dockerfile to install curl and adjusted health check.

**Lesson Learned:**  
Always validate health check dependencies in minimal images.

## Health Check Passing Locally but Failing in Docker

**Problem:**
Application worked locally but failed Docker health checks.

**Root Cause:**

Health endpoint was not registered correctly in the web framework

Incorrect route definition caused 404 responses

**Fix:**

Corrected route configuration

Verified endpoint manually using curl inside the container

**Lesson Learned:**
Always test health endpoints inside the container, not just on the host.

## Merge Conflicts & Duplicate Identifiers

**Problem:**
Duplicate identifiers after merging changes from main branch.

**Root Cause:**

Overlapping changes in shared files

No rebase before merge

**Fix:**

Inspected git diff and commit history

Rebased branch and resolved conflicts manually

**Lesson Learned:**
Clean Git workflows reduce merge pain, rebase early, rebase often.

## CI Pipeline Failing Due to Syntax Errors

**Problem:**
CI failed during linting with E999 SyntaxError.

**Root Cause:**

Invalid syntax in configuration file

Incorrect key/value formatting

**Fix:**

Corrected file structure

Validated configuration locally before pushing

**Lesson Learned:**
CI catches issues early — but only if linting is strict and properly configured.

## Container Marked Healthy but App Not Reachable

**Problem:**
Container showed healthy, but curl localhost:3000 failed.

**Root Cause:**

Application was bound to 127.0.0.1 instead of 0.0.0.0

**Fix:**

Updated server to listen on 0.0.0.0

Rebuilt Docker image

**Lesson Learned:**
Containerized apps must bind to all interfaces to be accessible externally.

## Docker Compose Dependency Error

**Problem:**
Docker Compose failed with:
service "backend" depends on undefined service "db"

**Root Cause:**

Service name mismatch in depends_on

**Fix:**

Aligned service names correctly

Validated compose file before running

**Lesson Learned:**
Small configuration mismatches can block entire environments — naming consistency matters.






> Most of my DevOps growth came from things breaking.  
> This page is proof that I know how to fix them.


**CodeBuild caching** allows you to store parts of your build environment at the project level between builds so they don't have to be recreated from scratch every time — reducing build duration.

- Same CodeBuild project triggered by CodePipeline run 1 → cache saved
- Same CodeBuild project triggered by CodePipeline run 2 → cache reused
- Different CodeBuild project → separate cache entirely

Without caching every build starts completely fresh — Docker layers get pulled from ECR/Docker Hub, dependencies get downloaded from npm/maven/pip, source gets checked out — even if none of it changed since the last build.

With caching CodeBuild saves those expensive parts after the first build and reuses them on subsequent builds, so only what actually changed needs to be processed. The cache type controls **where** the cache is stored (local host or S3) and the cache mode controls **what** gets cached (Docker layers, source, or custom directories).


---

## Cache Types

### Local Cache
- Stored **on the CodeBuild build host itself**
- No network transfer — fastest possible cache retrieval
- **Not guaranteed** to persist between builds — only available if the same host is reused
- No additional cost
- Best when: one build at a time, frequent intervals (same host likely reused)

### S3 Cache
- Stored **in an S3 bucket**
- Persists regardless of which host picks up the build
- Network transfer cost on every cache read/write
- Additional S3 storage cost
- Best when: concurrent builds, infrequent builds, or when host reuse cannot be guaranteed

---

## Cache Modes

### Docker Layer Cache Mode
- Caches individual **Docker image layers**
- When a layer hasn't changed, it is not rebuilt or re-pulled
- Only changed layers are rebuilt
- Directly addresses slow builds caused by large Docker container builds
- Works with both Local and S3 cache types

### Source Cache Mode
- Caches the **source code directory**
- Reduces time spent on source checkout
- Does NOT help with Docker build times
- Useful when source checkout is the bottleneck (large repos)

### Custom Cache Mode
- Caches **arbitrary directories** you specify (e.g. node_modules, .m2, pip cache)
- You define the paths in buildspec.yml
- Useful for dependency caches that don't change often

---

## Choosing the Right Combination

| Scenario | Cache Type | Cache Mode |
|---|---|---|
| Large Docker container, one build at a time, frequent | **Local** | **Docker layer** |
| Large Docker container, concurrent builds | **S3** | **Docker layer** |
| Large Docker container, infrequent builds | **S3** | **Docker layer** |
| Slow source checkout, one build at a time | **Local** | **Source** |
| Slow source checkout, concurrent builds | **S3** | **Source** |
| Heavy dependencies (npm, maven, pip) | **S3 or Local** | **Custom** |

---

## The Key Decision Logic

Is the build Docker-based and slow due to image size? → Docker layer cache mode

Is there one build at a time running frequently? → Local cache (same host likely reused, faster, cheaper)

Are there concurrent builds or infrequent builds? → S3 cache (guaranteed persistence across different hosts)

Is source checkout the bottleneck (not Docker)? → Source cache mode

Are dependencies (npm/pip/maven) the bottleneck? → Custom cache mode with dependency directory paths

---

## Why "Least Amount of Time" Points to Local

- Local cache = no network round trip to fetch cache
- S3 cache = download from S3 before build, upload to S3 after build
- For frequent sequential builds on one project: local is always faster than S3

---

## buildspec.yml Cache Configuration
```yaml
cache:
  paths:
    - '/root/.m2/**/*'      # Maven dependencies (custom mode)
    - '/root/.npm/**/*'     # npm cache (custom mode)

# For Docker layer caching — configured on the CodeBuild project,
# not in buildspec.yml. Enable in project settings under Cache.
```

---

## Exam Traps

| Wrong assumption | Correct understanding |
|---|---|
| S3 is always better because it persists | Local is faster and sufficient for frequent sequential builds on same project |
| Local cache works for concurrent builds | Local cache is host-specific — concurrent builds on different hosts won't share it |
| Source cache helps Docker builds | Source cache only helps checkout time — use Docker layer mode for Docker build time |
| Docker layer cache needs S3 | Docker layer cache works with local cache — local is faster for single sequential builds |
| Cache always available on same host | Only likely with frequent sequential builds — not guaranteed |

---

## Summary — The One-Line Rules

- **Docker build bottleneck** → Docker layer cache mode
- **One build at a time, frequent** → Local cache
- **Concurrent or infrequent builds** → S3 cache  
- **Source checkout bottleneck** → Source cache mode
- **Dependency install bottleneck** → Custom cache mode
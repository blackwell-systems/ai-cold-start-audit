# Container Lifecycle Management

## `container build <tool-name> [--dockerfile PATH]`

Automate the container build workflow: auto-increment round number, check for reusable containers, build and start.

**Step 1: Auto-detect round number**
```bash
# Find highest existing round
HIGHEST=$(docker images --format '{{.Repository}}' | grep "^${TOOL_NAME}-r" | sed 's/.*-r//' | sort -n | tail -1)

if [ -z "$HIGHEST" ]; then
  NEXT_ROUND=1
else
  NEXT_ROUND=$((HIGHEST + 1))
fi

CONTAINER_NAME="${TOOL_NAME}-r${NEXT_ROUND}"
IMAGE_NAME="${TOOL_NAME}-r${NEXT_ROUND}"
```

**Step 2: Check for reusable container**
```bash
RUNNING=$(docker ps --filter name=^${CONTAINER_NAME}$ --format '{{.Names}}')

if [ -n "$RUNNING" ]; then
  echo "✓ Container $CONTAINER_NAME already running"
  echo "Reusing existing container. No build needed."
  echo ""
  echo "Next: /cold-start-audit run $TOOL_NAME --mode container $CONTAINER_NAME"
  exit 0
fi
```

**Step 3: Check for existing image (source unchanged)**
```bash
IMAGE_EXISTS=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep "^${IMAGE_NAME}:")

if [ -n "$IMAGE_EXISTS" ]; then
  echo "? Image $IMAGE_NAME exists. Options:"
  echo "  1. Reuse existing image (fast, use if source code unchanged)"
  echo "  2. Rebuild image (use if source code changed since last audit)"

  # Ask user or default to reuse
  read -p "Choice [1/2]: " CHOICE

  if [ "$CHOICE" = "1" ]; then
    echo "Starting container from existing image..."
    docker run -d --name $CONTAINER_NAME --rm $IMAGE_NAME sleep 3600
    echo "✓ Container $CONTAINER_NAME started"
    exit 0
  fi
fi
```

**Step 4: Build new image**
```bash
# Find Dockerfile
DOCKERFILE_PATH="${DOCKERFILE_ARG:-Dockerfile.sandbox}"
if [ ! -f "$DOCKERFILE_PATH" ]; then
  DOCKERFILE_PATH="docker/Dockerfile.sandbox"
fi

if [ ! -f "$DOCKERFILE_PATH" ]; then
  echo "✗ Error: Dockerfile not found at Dockerfile.sandbox or docker/Dockerfile.sandbox"
  echo ""
  echo "Container mode requires an audit-ready Dockerfile with:"
  echo "  - Multi-stage build (builder + runtime)"
  echo "  - Non-root user with sudo access"
  echo "  - Tool at /usr/local/bin/<tool-name>"
  echo ""
  echo "Options:"
  echo "  1. Generate: /dockerfile-sandbox-gen <tool-name>"
  echo "  2. Manual: Create Dockerfile.sandbox (see container.md for template)"
  echo "  3. Custom: Use --dockerfile PATH to specify a different file"
  echo ""
  echo "Contract details:"
  echo "  https://github.com/blackwell-systems/agentic-cold-start-audit/blob/main/sandbox-setup.md#dockerfile-pattern"
  exit 1
fi

echo "Building image $IMAGE_NAME from $DOCKERFILE_PATH..."
docker build -t $IMAGE_NAME -f $DOCKERFILE_PATH .

if [ $? -ne 0 ]; then
  echo "✗ Build failed"
  exit 1
fi

echo "✓ Image built: $IMAGE_NAME"
```

**Step 5: Start container**
```bash
echo "Starting container..."
docker run -d --name $CONTAINER_NAME --rm $IMAGE_NAME sleep 3600

if [ $? -ne 0 ]; then
  echo "✗ Failed to start container"
  exit 1
fi

echo "✓ Container started: $CONTAINER_NAME"
echo ""
echo "Next: /cold-start-audit run $TOOL_NAME --mode container $CONTAINER_NAME"
```

**Output example:**
```
Auto-detected round: brewprune-r8 (highest existing: brewprune-r7)

Building image brewprune-r8 from Dockerfile.sandbox...
[docker build output...]
✓ Image built: brewprune-r8

Starting container...
✓ Container started: brewprune-r8

Next: /cold-start-audit run brewprune --mode container brewprune-r8
```

## `container list <tool-name>`

Show all images and containers for a tool with status.

**Implementation:**
```bash
TOOL_NAME="$1"

echo "Images for $TOOL_NAME:"
echo ""
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}' | grep "^${TOOL_NAME}-r"

echo ""
echo "Containers for $TOOL_NAME:"
echo ""
docker ps -a --filter name="^${TOOL_NAME}-r" --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
```

**Output example:**
```
Images for brewprune:

REPOSITORY      TAG     SIZE    CREATED
brewprune-r7    latest  156MB   2 days ago
brewprune-r8    latest  157MB   5 hours ago

Containers for brewprune:

NAMES           STATUS                  IMAGE
brewprune-r7    Exited (0) 2 days ago   brewprune-r7
brewprune-r8    Up 5 hours              brewprune-r8
```

## `container cleanup <tool-name> [--keep-latest N]`

Remove old containers and images, keeping the latest N rounds.

**Implementation:**
```bash
TOOL_NAME="$1"
KEEP_LATEST="${KEEP_LATEST:-2}"  # Default: keep 2 most recent

# Get all rounds sorted
ALL_ROUNDS=$(docker images --format '{{.Repository}}' | grep "^${TOOL_NAME}-r" | sed 's/.*-r//' | sort -n)
TOTAL_ROUNDS=$(echo "$ALL_ROUNDS" | wc -l)

if [ "$TOTAL_ROUNDS" -le "$KEEP_LATEST" ]; then
  echo "✓ Only $TOTAL_ROUNDS rounds exist. No cleanup needed (keeping latest $KEEP_LATEST)."
  exit 0
fi

ROUNDS_TO_REMOVE=$(echo "$ALL_ROUNDS" | head -n $((TOTAL_ROUNDS - KEEP_LATEST)))

echo "Found $TOTAL_ROUNDS rounds. Keeping latest $KEEP_LATEST."
echo ""
echo "Rounds to remove:"
for ROUND in $ROUNDS_TO_REMOVE; do
  echo "  - ${TOOL_NAME}-r${ROUND}"
done
echo ""
read -p "Continue? [y/N]: " CONFIRM

if [ "$CONFIRM" != "y" ]; then
  echo "Cancelled."
  exit 0
fi

for ROUND in $ROUNDS_TO_REMOVE; do
  NAME="${TOOL_NAME}-r${ROUND}"

  # Remove container (if exists)
  docker rm -f $NAME 2>/dev/null && echo "✓ Removed container: $NAME" || echo "  (no container)"

  # Remove image
  docker rmi $NAME 2>/dev/null && echo "✓ Removed image: $NAME" || echo "✗ Failed to remove image: $NAME"
done

echo ""
echo "✓ Cleanup complete"
```

**Output example:**
```
Found 8 rounds. Keeping latest 2.

Rounds to remove:
  - brewprune-r1
  - brewprune-r2
  ...
  - brewprune-r6

Continue? [y/N]: y

✓ Removed container: brewprune-r1
✓ Removed image: brewprune-r1
  (no container)
✓ Removed image: brewprune-r2
...

✓ Cleanup complete
```

## Container Setup Reference

### Container Naming Convention

Containers follow the pattern **`{tool}-r{N}`** where N is the audit round number (1, 2, 3…).

**Auto-increment logic** — use the `container build` command which handles this automatically, or manually:

```bash
# Find the highest existing round number for this tool
docker images --format '{{.Repository}}' | grep '^{tool}-r' | sort -V | tail -1
# e.g. "brewprune-r7" → next round is brewprune-r8
```

If no existing images are found, start at r1.

### Container Lifecycle

Understanding the distinction between these three concepts is critical:

- **Dockerfile** (template) — Infrastructure definition, created once and reused across audit rounds. Only update when runtime dependencies or environment setup changes.
- **Image** (snapshot) — Built from the Dockerfile, captures the tool's compiled binary at a point in time. Rebuild this when the tool's source code changes.
- **Container** (running instance) — Ephemeral execution environment. Create a new one for each audit round with a round-specific name following the `{tool}-r{N}` pattern.

### Dockerfile Pattern (multi-stage build)

Create `Dockerfile.sandbox` once (or use `docker/Dockerfile.sandbox` if the project already has one):

```dockerfile
# Stage 1: build the tool
FROM golang:1.23 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/mytool ./cmd/mytool

# Stage 2: clean runtime environment
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl git sudo && rm -rf /var/lib/apt/lists/*
RUN useradd -m -s /bin/bash appuser && echo "appuser ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
USER appuser
# Install tool dependencies here
COPY --from=builder --chown=appuser:appuser /out/mytool /usr/local/bin/mytool
CMD ["/bin/bash"]
```

Adapt the builder stage and dependencies for the tool's language and requirements. Use real dependencies, not mocks — real package managers surface real edge cases.

### When to Update the Dockerfile

Only modify `Dockerfile.sandbox` when:
- Runtime dependencies change (new packages needed in the container)
- Tool installation method changes (different build flags, new subcommands to copy)
- Environment setup changes (different user permissions, PATH modifications)

Do NOT recreate the Dockerfile just because the tool's source code changed — rebuilding the image is sufficient.

### First-Time Setup

Use the automated command:

```bash
/cold-start-audit container build <tool-name>
```

Or manually:

```bash
# Create Dockerfile.sandbox (see pattern above)
# Build image with round-specific tag
docker build -t mytool-r1 -f Dockerfile.sandbox .
# Start container with round-specific name
docker run -d --name mytool-r1 --rm mytool-r1 sleep 3600
# Verify
docker ps --filter name=mytool-r1
```

### Subsequent Rounds

Use the automated workflow:

```bash
/cold-start-audit container build <tool-name>
# Auto-detects highest round, offers reuse if unchanged, builds if needed
```

**Key principle:** The Dockerfile.sandbox is stable infrastructure. Only the image (built binary) and container (running instance) change between rounds.

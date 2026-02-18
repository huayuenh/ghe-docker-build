# Docker Build Action

A comprehensive GitHub Action for building Docker images with support for multi-platform builds, build arguments, caching, and more.

## Features

- 🐳 **Docker Buildx integration** for advanced build features
- 🌍 **Multi-platform builds** (linux/amd64, linux/arm64, etc.)
- 🚀 **Build arguments** for parameterized builds
- 💾 **Layer caching** support for faster builds
- 🏷️ **Custom labels** for image metadata
- 🎯 **Multi-stage builds** with target stage selection
- 📦 **Push or load** images after building

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `context` | ✗ | `.` | Build context path |
| `dockerfile` | ✗ | `Dockerfile` | Path to Dockerfile |
| `app-name` | ✗ | Repository name | Application name (automatically uses repository name if not provided) |
| `tag` | ✗ | Auto-detected | Image tag (automatically determined from git ref/SHA if not provided) |
| `image-name` | ✗ | - | Full image name with tag (e.g., `myapp:latest`). If provided, overrides `app-name` and `tag` |
| `build-args` | ✗ | - | Build arguments in KEY=VALUE format, one per line |
| `platforms` | ✗ | - | Target platforms for multi-platform builds (e.g., `linux/amd64,linux/arm64`) |
| `cache-from` | ✗ | - | External cache sources (e.g., `type=registry,ref=user/app:cache`) |
| `cache-to` | ✗ | - | Cache export destination (e.g., `type=registry,ref=user/app:cache,mode=max`) |
| `push` | ✗ | `false` | Push the image after building (requires docker login) |
| `load` | ✗ | `true` | Load the image into docker daemon (cannot be used with multi-platform builds) |
| `labels` | ✗ | - | Image labels in KEY=VALUE format, one per line |
| `target` | ✗ | - | Target build stage (for multi-stage builds) |
| `no-cache` | ✗ | `false` | Do not use cache when building the image |
| `pull` | ✗ | `false` | Always attempt to pull a newer version of the base image |

### Smart Defaults

The action automatically determines the app name and tag if not provided:

- **App Name**: Uses the repository name (e.g., `my-repo` from `owner/my-repo`)
- **Tag**: Intelligently determined based on git context:
  - Git tags (e.g., `v1.0.0`): Uses the tag name
  - Main branch: Uses commit SHA
  - Other branches: Uses branch name
  - Fallback: Uses commit SHA or `latest`

## Outputs

| Output | Description |
|--------|-------------|
| `image-name` | Full name of the built image (e.g., `myapp:v1.0.0`) |
| `app-name` | Application name used for the image |
| `tag` | Tag used for the image |
| `image-id` | Image ID of the built image |
| `digest` | Image digest (if pushed) |
| `metadata` | Build metadata in JSON format |

## Usage Examples

### Simplest Build (Recommended)

Build with automatic app name and tag detection:

```yaml
- name: Build Docker image
  uses: ./docker-build-action
```

This will automatically:
- Use the repository name as the app name
- Determine the tag based on the git ref (tag name, branch name, or commit SHA)
- Output: `<repo-name>:<auto-tag>`

### Build with Custom App Name

Override the app name while keeping automatic tag detection:

```yaml
- name: Build Docker image
  uses: ./docker-build-action
  with:
    app-name: my-custom-app
```

### Build with Custom Tag

Override the tag while using the repository name:

```yaml
- name: Build Docker image
  uses: ./docker-build-action
  with:
    tag: v2.0.0
```

### Build with Both Custom App Name and Tag

```yaml
- name: Build Docker image
  uses: ./docker-build-action
  with:
    app-name: my-custom-app
    tag: v2.0.0
```

### Legacy: Full Image Name (Backward Compatible)

For backward compatibility, you can still provide the full image name:

```yaml
- name: Build Docker image
  uses: ./docker-build-action
  with:
    image-name: myapp:latest
```

### Build with Arguments

Build with custom build arguments (using automatic naming):

```yaml
- name: Build with arguments
  uses: ./docker-build-action
  with:
    build-args: |
      NODE_VERSION=18
      APP_ENV=production
      BUILD_DATE=${{ github.event.head_commit.timestamp }}
```

### Multi-Platform Build

Build for multiple platforms (using automatic naming):

```yaml
- name: Multi-platform build
  uses: ./docker-build-action
  with:
    platforms: linux/amd64,linux/arm64
    push: true
```

### Build with Caching

Use registry caching for faster builds (using automatic naming):

```yaml
- name: Build with cache
  id: build
  uses: ./docker-build-action
  with:
    cache-from: type=registry,ref=myregistry/${{ steps.build.outputs.app-name }}:cache
    cache-to: type=registry,ref=myregistry/${{ steps.build.outputs.app-name }}:cache,mode=max
```

### Multi-Stage Build

Build a specific stage from a multi-stage Dockerfile:

```yaml
- name: Build production stage
  uses: ./docker-build-action
  with:
    tag: prod
    target: production
```

### Build with Labels

Add custom labels to the image (automatic title from app name):

```yaml
- name: Build with labels
  uses: ./docker-build-action
  with:
    labels: |
      org.opencontainers.image.created=${{ github.event.head_commit.timestamp }}
      org.opencontainers.image.revision=${{ github.sha }}
```

Note: The action automatically adds `org.opencontainers.image.title` and `org.opencontainers.image.version` based on the app name and tag.

### Custom Dockerfile Location

Build using a Dockerfile in a different location:

```yaml
- name: Build from custom Dockerfile
  uses: ./docker-build-action
  with:
    context: ./backend
    dockerfile: ./backend/Dockerfile.prod
    app-name: myapp-backend
```

## Complete Workflow Example

Here's a complete example that builds and pushes an image to IBM Cloud Container Registry:

```yaml
name: Build and Push to IBM Cloud Container Registry

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  IBM_CLOUD_API_KEY: ${{ secrets.IBM_CLOUD_API_KEY }}
  IBM_CLOUD_REGION: us-south
  IBM_CLOUD_NAMESPACE: my-namespace

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Build Docker image
        id: build
        uses: ./docker-build-action
        with:
          platforms: linux/amd64,linux/arm64
          build-args: |
            COMMIT_SHA=${{ github.sha }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
          labels: |
            org.opencontainers.image.created=${{ github.event.head_commit.timestamp }}
            org.opencontainers.image.revision=${{ github.sha }}
      
      - name: Push to IBM Cloud Container Registry
        uses: ../  # Path to ibmcloud-cr-action
        with:
          apikey: ${{ env.IBM_CLOUD_API_KEY }}
          image: ${{ env.IBM_CLOUD_REGION }}.icr.io/${{ env.IBM_CLOUD_NAMESPACE }}/${{ steps.build.outputs.image-name }}
          local-image: ${{ steps.build.outputs.image-name }}
          action: push
          scan: true
          region: ${{ env.IBM_CLOUD_REGION }}
      
      - name: Display build info
        run: |
          echo "Built image: ${{ steps.build.outputs.image-name }}"
          echo "App name: ${{ steps.build.outputs.app-name }}"
          echo "Tag: ${{ steps.build.outputs.tag }}"
          echo "Registry: ${{ env.IBM_CLOUD_REGION }}.icr.io/${{ env.IBM_CLOUD_NAMESPACE }}/${{ steps.build.outputs.image-name }}"
```

This example demonstrates:
- **Automatic naming** - Uses repository name and git-based tag
- **Multi-platform build** - Builds for amd64 and arm64
- **IBM Cloud integration** - Pushes to IBM Cloud Container Registry with API key in env
- **Vulnerability scanning** - Scans image after push
- **Build outputs** - Uses action outputs for subsequent steps

## Integration with IBM Cloud Container Registry

This action works seamlessly with the IBM Cloud Container Registry action. See the workflow examples in the main repository for complete integration examples.

## Best Practices

1. **Use automatic naming**: Let the action determine app name and tag from git context
2. **Override when needed**: Use `app-name` or `tag` inputs for custom naming
3. **Enable caching**: Use registry caching for faster builds
4. **Multi-platform builds**: Build for multiple architectures when needed
5. **Build arguments**: Use build args for configuration, not secrets
6. **Labels**: Add metadata labels for better image management
7. **Layer optimization**: Order Dockerfile instructions from least to most frequently changing
8. **Use outputs**: Reference `steps.build.outputs.app-name` and `steps.build.outputs.tag` in subsequent steps

## Troubleshooting

### Build fails with "no space left on device"

- Clean up old Docker images and containers
- Use GitHub Actions cache cleanup
- Consider using external cache storage

### Multi-platform build is slow

- Use registry caching
- Build platforms separately if needed
- Consider using native runners for each platform

### Cannot load multi-platform builds

- Multi-platform builds cannot be loaded into the local Docker daemon
- Set `push: true` to push to a registry instead
- Or build for a single platform with `load: true`

## License

This project is licensed under the MIT License.

# Made with Bob
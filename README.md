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

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `context` | No | `.` | Build context path |
| `dockerfile` | No | `Dockerfile` | Path to Dockerfile |
| `image-name` | Yes | - | Name and tag for the built image (e.g., `myapp:latest`) |
| `build-args` | No | - | Build arguments in KEY=VALUE format, one per line |
| `platforms` | No | - | Target platforms for multi-platform builds (e.g., `linux/amd64,linux/arm64`) |
| `cache-from` | No | - | External cache sources (e.g., `type=registry,ref=user/app:cache`) |
| `cache-to` | No | - | Cache export destination (e.g., `type=registry,ref=user/app:cache,mode=max`) |
| `push` | No | `false` | Push the image after building (requires docker login) |
| `load` | No | `true` | Load the image into docker daemon (cannot be used with multi-platform builds) |
| `labels` | No | - | Image labels in KEY=VALUE format, one per line |
| `target` | No | - | Target build stage (for multi-stage builds) |
| `no-cache` | No | `false` | Do not use cache when building the image |
| `pull` | No | `false` | Always attempt to pull a newer version of the base image |

## Outputs

| Output | Description |
|--------|-------------|
| `image-name` | Full name of the built image |
| `image-id` | Image ID of the built image |
| `digest` | Image digest (if pushed) |
| `metadata` | Build metadata in JSON format |

## Usage Examples

### Basic Build

Build a simple Docker image:

```yaml
- name: Build Docker image
  uses: ./docker-build-action
  with:
    image-name: myapp:latest
```

### Build with Arguments

Build with custom build arguments:

```yaml
- name: Build with arguments
  uses: ./docker-build-action
  with:
    image-name: myapp:v1.0.0
    build-args: |
      NODE_VERSION=18
      APP_ENV=production
      BUILD_DATE=${{ github.event.head_commit.timestamp }}
```

### Multi-Platform Build

Build for multiple platforms:

```yaml
- name: Multi-platform build
  uses: ./docker-build-action
  with:
    image-name: myapp:latest
    platforms: linux/amd64,linux/arm64
    push: true
```

### Build with Caching

Use registry caching for faster builds:

```yaml
- name: Build with cache
  uses: ./docker-build-action
  with:
    image-name: myapp:latest
    cache-from: type=registry,ref=myregistry/myapp:cache
    cache-to: type=registry,ref=myregistry/myapp:cache,mode=max
```

### Multi-Stage Build

Build a specific stage from a multi-stage Dockerfile:

```yaml
- name: Build production stage
  uses: ./docker-build-action
  with:
    image-name: myapp:prod
    target: production
```

### Build with Labels

Add custom labels to the image:

```yaml
- name: Build with labels
  uses: ./docker-build-action
  with:
    image-name: myapp:latest
    labels: |
      org.opencontainers.image.title=MyApp
      org.opencontainers.image.version=${{ github.ref_name }}
      org.opencontainers.image.created=${{ github.event.head_commit.timestamp }}
      org.opencontainers.image.revision=${{ github.sha }}
```

### Custom Dockerfile Location

Build using a Dockerfile in a different location:

```yaml
- name: Build from custom Dockerfile
  uses: ./docker-build-action
  with:
    context: ./backend
    dockerfile: ./backend/Dockerfile.prod
    image-name: myapp-backend:latest
```

## Complete Workflow Example

Here's a complete example that builds and pushes an image:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: myusername/myapp
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
      
      - name: Build and push
        uses: ./docker-build-action
        with:
          image-name: ${{ steps.meta.outputs.tags }}
          platforms: linux/amd64,linux/arm64
          push: true
          cache-from: type=registry,ref=myusername/myapp:cache
          cache-to: type=registry,ref=myusername/myapp:cache,mode=max
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            VERSION=${{ github.ref_name }}
            COMMIT_SHA=${{ github.sha }}
```

## Integration with IBM Cloud Container Registry

This action works seamlessly with the IBM Cloud Container Registry action. See the workflow examples in the main repository for complete integration examples.

## Best Practices

1. **Use specific tags**: Avoid using `latest` in production
2. **Enable caching**: Use registry caching for faster builds
3. **Multi-platform builds**: Build for multiple architectures when needed
4. **Build arguments**: Use build args for configuration, not secrets
5. **Labels**: Add metadata labels for better image management
6. **Layer optimization**: Order Dockerfile instructions from least to most frequently changing

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
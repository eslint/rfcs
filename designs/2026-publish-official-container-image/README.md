- Repo: eslint/eslint
- Start Date: 2026-01-13
- RFC PR: (leave this empty, to be filled in later)
- Authors: 6543

# Official Container Image for ESLint

## Summary

Publish and maintain official OCI-compliant container images for ESLint to provide a standardized, secure, and well-maintained container for running ESLint in containerized environments. The images will be published to GitHub Container Registry and updated with each ESLint release.

## Motivation

Currently, there are numerous unofficial ESLint Docker images on Docker Hub (https://hub.docker.com/search?q=eslint) and GitHub Container Registry (https://github.com/search?q=eslint+package_type%3Acontainer&type=registrypackages), leading to several challenges:

1. **Fragmentation**: Users must choose between many unofficial images with varying quality, maintenance levels, and security practices
2. **Security concerns**: Unofficial images may not be regularly updated or may contain vulnerabilities
3. **Integration**: Projects like Woodpecker CI and other tools who are containerized need reliable, official ESLint containers to build on top of them
4. **Trust and reliability**: An official image provides assurance of quality and ongoing maintenance from the ESLint team

By providing an official container image, we enable:

- Consistent ESLint execution across different environments
- Easier integration with CI/CD pipelines
- Reduced security risks through official, maintained images
- Better developer experience for containerized workflows

## Detailed Design

### Base Images

We will provide two image variants:

1. **Alpine-based** (default/primary):
   - Smaller image size (~50-80MB compressed)
   - Based on `node:alpine` official image
   - Suitable for most use cases
   - Tagged as: `eslint:<version>-alpine` and `eslint:<version>`

2. **Debian-based**:
   - Larger but more compatible
   - Based on `node:slim` official image
   - Better compatibility with native dependencies
   - Tagged as: `eslint:<version>-debian`

### Registry

Images will be published to **GitHub Container Registry** (ghcr.io):

- Repository: `ghcr.io/eslint/eslint`
- Free for public open-source projects
- Integrates well with GitHub Actions

Docker Hub would also be nice but that will likely trigger additional costs if you pull/push to much (rate limits)

### Image Structure

```dockerfile
# Example Alpine-based Dockerfile
FROM node:20-alpine

# Install ESLint globally
RUN npm install -g eslint@<VERSION>

# Create working directory
WORKDIR /work

# Set entrypoint
ENTRYPOINT ["eslint"]
```

### Versioning Strategy

Images will be tagged with:

- **Full version**: `9.17.0`, `9.17.0-alpine`, `9.17.0-debian`
- **Minor version**: `9.17`, `9.17-alpine`, `9.17-debian`
- **Major version**: `9`, `9-alpine`, `9-debian`
- **Latest**: `latest`, `latest-alpine`, `latest-debian`

### Build and Release Process

1. **Automated builds**: Use GitHub Actions to build images on each ESLint release
2. **Multi-architecture support**: Build for `linux/amd64` and `linux/arm64`
3. **Release workflow**:
   - Triggered by new ESLint release tag
   - Build all image variants
   - Push to registry with appropriate tags

### Usage Examples

```bash
# Run ESLint on current directory
docker run --rm -v $(pwd):/work ghcr.io/eslint/eslint .

# With custom config
docker run --rm -v $(pwd):/work ghcr.io/eslint/eslint --config .eslintrc.json .

# Specific version
docker run --rm -v $(pwd):/work ghcr.io/eslint/eslint:9.17.0 .

# In CI/CD (example)
docker run --rm -v $(pwd):/work ghcr.io/eslint/eslint:9 --format json .
```

## Documentation

### Required Documentation Updates

1. **ESLint Documentation Site**:
   - New section: "Using ESLint with Docker"
   - Installation instructions for Docker usage
   - CI/CD integration examples

2. **GitHub Repository**:
   - Update main README with Docker installation option
   - Add Dockerfile to repository
   - Add CI/CD pipeline to release container images

3. **Blog Announcement**:
   - Official blog post announcing the container image
   - Explain motivation and use cases
   - Provide migration guide from unofficial images

4. **Image Documentation**:
   - README on GitHub Container Registry
   - Tags and versioning explanation
   - Security and update policy (same as [repo](https://github.com/eslint/eslint/security))

## Drawbacks

1. **Support expectations**: Users may expect official support for Docker-specific issues
2. **Complexity**: Adds another release artifact to manage
3. **Security responsibility**: Team becomes responsible for container security
4. **Breaking changes**: Future Node.js or base image updates may require breaking changes
5. **Limited use case**: Not all ESLint users need Docker images

## Backwards Compatibility Analysis

This is a new feature addition and does not affect existing ESLint functionality. There are no backwards compatibility concerns because:

- Existing npm package distribution remains unchanged
- Current users are not affected unless they choose to adopt Docker
- No changes to ESLint core functionality
- No deprecation of existing installation methods

The only consideration is that users currently relying on unofficial images may need to update their workflows to use the official image, but this is optional and beneficial.

## Alternatives

### Alternative 1: Docker Hub Instead of GitHub Container Registry

**Pros**: More discoverable, widely used
**Cons**: Requires separate account management, potential rate limiting issues, less integrated with GitHub workflow
**Decision**: GitHub Container Registry is better aligned with the project's GitHub-hosted nature

### Alternative 2: Provide Only Dockerfile, No Published Images

**Pros**: Minimal maintenance, users build their own
**Cons**: Defeats the purpose of providing an official image, users still face the same fragmentation issue
**Decision**: Publishing images provides the most value

### Alternative 3: Support Only Alpine or Only Debian

**Pros**: Simpler maintenance, fewer variants
**Cons**: Limits user choice, some users need specific base OS
**Decision**: Offering both provides flexibility while remaining manageable

### Alternative 4: Multi-tool Image (Include formatters, linters, etc.)

**Pros**: One-stop solution for code quality
**Cons**: Large image size, scope creep, not focused
**Decision**: Keep focused on ESLint only, single responsibility

### Prior Art

Similar projects with official Docker images:

- **Terraform**: Official images on Docker Hub and ghcr.io
- **Hadolint**: Official image on ghcr.io
- **ShellCheck**: Official images on Docker Hub
- **Node.js**: Official images serve as base for our implementation

## Open Questions

1. **Node.js version strategy**: Which Node.js versions should we support? Only LTS? Only current?
2. **Plugin support**: Should we provide image variants with popular plugins pre-installed?
3. **Config file handling**: Should we include any default configurations?
4. **Update frequency**: Should we rebuild images for security updates between ESLint releases?
5. **Deprecation policy**: How long should we maintain older image versions?

## Help Needed

- **Docker expertise**: Team members or contributors with Docker/container experience
- **CI/CD setup**: Help setting up GitHub Actions workflow for multi-architecture builds

## Frequently Asked Questions

### Q: Will this replace the npm package?

**A**: No, the Docker image is an alternative installation method. The npm package remains the primary distribution method.

### Q: What ESLint plugins will be included?

**A**: Initially, only core ESLint will be included. Users can install plugins by extending the image or mounting node_modules.

### Q: How do I use my project's dependencies with the Docker image?

**A**: Mount your project directory including node_modules, or extend the base image to install your dependencies.

### Q: Will the image include a specific Node.js version?

**A**: Yes, images will be based on Node.js LTS versions and updated when new LTS versions are released.

### Q: How often will images be updated?

**A**: Images will be built for every ESLint release and potentially rebuilt for critical security updates.

### Q: Can I use this in production?

**A**: Yes, the images are designed for both development and production CI/CD environments.

### Q: Are the images OCI-compliant?

**A**: Yes, all images follow OCI standards and work with Docker, Podman, containerd, and other OCI-compliant runtimes.

## Related Discussions

- Original Issue: https://github.com/eslint/eslint/issues/20430
- Docker Hub ESLint search: https://hub.docker.com/search?q=eslint (shows ecosystem fragmentation)

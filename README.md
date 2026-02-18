# IBM Cloud Container Registry GitHub Action

A comprehensive GitHub Action for managing container images in IBM Cloud Container Registry. This action supports pushing, pulling, tagging, retagging images, managing namespaces, and running vulnerability scans.

## Features

- 🚀 **Push images** to IBM Cloud Container Registry
- 📥 **Pull images** from IBM Cloud Container Registry
- 🏷️ **Tag images** with additional tags
- 🔄 **Retag images** to move tags between versions
- 🗑️ **Delete images** from the registry
- 📦 **Manage namespaces** (create, delete, list)
- 🔒 **Vulnerability scanning** with IBM Cloud Vulnerability Advisor (enabled by default)
- ⚙️ **Configurable scan behavior** - choose whether to fail on vulnerabilities
- 🔄 **Automatic retry logic** - waits up to 5 minutes for scan completion
- 🌍 **Multi-region support** with automatic region detection
- ✅ **Comprehensive error handling** and logging
- 🔧 **Uses IBM Cloud CLI marketplace action** for streamlined setup

## Prerequisites

- IBM Cloud account with Container Registry access
- IBM Cloud API key with appropriate permissions
- Docker image built and available locally (for push operations)

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `apikey` | ✓ | - | IBM Cloud API key for authentication |
| `image` | ✗* | - | Full image path (e.g., `us.icr.io/namespace/image:tag`). Required for push, pull, tag, retag, and delete actions |
| `local-image` | ✗ | - | Local image name to tag and push (e.g., `myapp:latest`). If specified, this image will be tagged with the target image path before pushing |
| `action` | ✓ | - | Operation to perform: `push`, `pull`, `tag`, `retag`, `delete`, or `namespace` |
| `scan` | ✗ | `true` | Enable vulnerability scanning after push/pull operations |
| `scan-fail-on-vulnerability` | ✗ | `true` | Fail the build if FAIL status is returned from vulnerability scan |
| `region` | ✗ | Auto-detect | IBM Cloud region (us-south, eu-gb, etc.). Auto-detected from image path if not specified |
| `source-tag` | ✗* | - | Source tag for retag operation |
| `target-tag` | ✗* | - | Target tag for tag/retag operations |
| `namespace` | ✗* | - | Namespace name for namespace operations |
| `namespace-action` | ✗* | - | Namespace operation: `create`, `delete`, or `list` |

**Note:** ✗* indicates conditionally required based on the `action` parameter.

## Outputs

| Output | Description |
|--------|-------------|
| `scan-result` | Vulnerability scan results in JSON format |
| `image-digest` | Image digest after push/pull operation |
| `namespaces` | List of namespaces (for namespace list action) |
| `operation-status` | Status of the operation (success/failure) |

## Usage Examples

### Push Image

Push a Docker image to IBM Cloud Container Registry:

```yaml
- name: Push to IBM Cloud Container Registry
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
```

### Push Locally Built Image

Build and push a local Docker image to IBM Cloud Container Registry:

```yaml
- name: Build Docker image
  run: docker build -t myapp:latest .

- name: Push to IBM Cloud Container Registry
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    local-image: myapp:latest
    action: push
```

This will automatically tag your local `myapp:latest` image as `us.icr.io/my-namespace/my-app:v1.0.0` before pushing it to the registry.

### Push Image with Vulnerability Scanning

Push an image and run a vulnerability scan:

```yaml
- name: Push and scan image
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
    scan: true
```

### Pull Image

Pull an image from IBM Cloud Container Registry:

```yaml
- name: Pull from IBM Cloud Container Registry
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: pull
```

### Tag Image

Add a new tag to an existing image:

```yaml
- name: Tag image as latest
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: tag
    target-tag: latest
```

### Retag Image

Move a tag from one version to another:

```yaml
- name: Promote staging to production
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app
    action: retag
    source-tag: staging
    target-tag: production
```

### Delete Image

Delete an image from the registry:

```yaml
- name: Delete old image
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:old-tag
    action: delete
```

### Create Namespace

Create a new namespace in IBM Cloud Container Registry:

```yaml
- name: Create namespace
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    action: namespace
    namespace-action: create
    namespace: my-new-namespace
    region: us-south
```

### List Namespaces

List all namespaces in your IBM Cloud account:

```yaml
- name: List namespaces
  uses: ./ibmcloud-cr-action
  id: list-ns
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    action: namespace
    namespace-action: list
    region: us-south

- name: Display namespaces
  run: echo "${{ steps.list-ns.outputs.namespaces }}"
```

### Delete Namespace

Delete a namespace (this will remove all images in the namespace):

```yaml
- name: Delete namespace
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    action: namespace
    namespace-action: delete
    namespace: old-namespace
    region: us-south
```

## Complete Workflow Example

Here's a simple example showing how to build and push an image:

```yaml
name: Build and Push to IBM Cloud

on:
  push:
    branches: [main]

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
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
          labels: |
            org.opencontainers.image.created=${{ github.event.head_commit.timestamp }}
      
      - name: Push to IBM Cloud Container Registry
        uses: ./
        with:
          apikey: ${{ env.IBM_CLOUD_API_KEY }}
          image: ${{ env.IBM_CLOUD_REGION }}.icr.io/${{ env.IBM_CLOUD_NAMESPACE }}/${{ steps.build.outputs.image-name }}
          local-image: ${{ steps.build.outputs.image-name }}
          action: push
          scan: true
          region: ${{ env.IBM_CLOUD_REGION }}
```

## Related Actions

For a complete CI/CD pipeline, see these complementary actions:
- [Docker Build Action](./docker-build-action) - Build Docker images with smart defaults
- [Deploy Action](./deploy-action) - Deploy to Kubernetes or OpenShift
- [Commit Status Action](./commit-status-action) - Set GitHub commit status

See [.github/workflows](./.github/workflows) for complete workflow examples.

## Supported Regions

The action supports automatic region detection from the image path. Supported regions include:

| Registry Domain | Region Code | Region Name |
|----------------|-------------|-------------|
| `us.icr.io` | `us-south` | US South (Dallas) |
| `eu.icr.io` | `eu-gb` | UK South (London) |
| `uk.icr.io` | `uk-south` | UK South (London) |
| `au.icr.io` | `au-syd` | Sydney |
| `jp.icr.io` | `jp-tok` | Tokyo |
| `de.icr.io` | `eu-de` | Frankfurt |

If the region cannot be detected from the image path, you can specify it explicitly using the `region` input.

## Vulnerability Scanning

Vulnerability scanning is **enabled by default** for push and pull operations. The action will:

1. Initiate IBM Cloud Vulnerability Advisor scan on the image
2. Poll for scan completion every 10 seconds (up to 5 minutes)
3. Parse scan results and check status (OK, WARN, FAIL, UNSUPPORTED, INCOMPLETE, UNSCANNED)
4. Output scan results in JSON format
5. Set the `scan-result` output with detailed findings

### Scan Status Behavior

- **OK**: No vulnerabilities found - build passes
- **WARN**: Warnings found - build passes
- **UNSUPPORTED**: Image type not supported for scanning - build passes
- **FAIL**: Critical vulnerabilities found - build fails (configurable)
- **INCOMPLETE/UNSCANNED**: Scan still in progress - action retries

### Configuring Scan Behavior

**Enable/Disable Scanning:**
```yaml
- name: Push without scanning
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
    scan: false  # Disable vulnerability scanning
```

**Allow Build to Continue Despite Vulnerabilities:**
```yaml
- name: Push and scan (don't fail on vulnerabilities)
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
    scan: true
    scan-fail-on-vulnerability: false  # Report but don't fail
```

**Access Scan Results:**
```yaml
- name: Push and scan
  id: push-scan
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
    scan: true

- name: Check scan results
  run: |
    echo "Scan results: ${{ steps.push-scan.outputs.scan-result }}"
```

### Retry Logic

The vulnerability scan includes automatic retry logic:
- Polls every 10 seconds for scan completion
- Maximum wait time: 5 minutes (30 attempts)
- Continues while status is INCOMPLETE or UNSCANNED
- Exits immediately when scan completes with final status

## Error Handling

The action includes comprehensive error handling:

- **Input validation**: Validates all required inputs before execution
- **Authentication errors**: Clear messages for API key or login failures
- **Image not found**: Checks if images exist before operations
- **Network failures**: Handles connection issues gracefully
- **Operation failures**: Provides detailed error messages with exit codes

## Security Best Practices

1. **Store API keys securely**: Always use GitHub Secrets for the IBM Cloud API key
2. **Use least privilege**: Grant only necessary permissions to the API key
3. **Enable vulnerability scanning**: Use `scan: true` for production images
4. **Review scan results**: Check vulnerability reports before deploying
5. **Use specific tags**: Avoid using `latest` tag in production

## Troubleshooting

### Authentication Failed

If you see authentication errors:
- Verify your IBM Cloud API key is correct
- Ensure the API key has Container Registry permissions
- Check that the region is correct

### Image Not Found

If push fails with "image not found":
- Ensure the Docker image is built before pushing
- Verify the image name matches exactly
- Check that Docker is running

### Namespace Already Exists

When creating a namespace that already exists:
- The action will report the namespace exists and continue
- Use the `list` action to check existing namespaces first

### Region Detection Issues

If region auto-detection fails:
- Specify the region explicitly using the `region` input
- Ensure the image path follows the format: `<region>.icr.io/namespace/image:tag`


## License

This project is licensed under the MIT License.

# P5.js Creative Coding Platform

Goldentooth's journey into creative computing required a platform for hosting and showcasing interactive p5.js sketches. The `p5js-sketches` project emerged as a Kubernetes-native solution that combines modern DevOps practices with artistic expression, providing a robust foundation for creative coding experiments and demonstrations.

## Project Overview

### Vision and Purpose

The p5js-sketches platform serves multiple purposes within the Goldentooth ecosystem:

- **Creative Expression**: A canvas for computational art and interactive visualizations
- **Educational Demos**: Showcase machine learning algorithms and mathematical concepts
- **Technical Exhibition**: Demonstrate Kubernetes deployment patterns for static content
- **Community Sharing**: Provide a gallery format for browsing and discovering sketches

### Architecture Philosophy

The platform embraces cloud-native principles while optimizing for the unique constraints of a Raspberry Pi cluster:

- **Container-Native**: Docker-based deployments with multi-architecture support
- **GitOps Workflow**: Code-to-deployment automation via Argo CD
- **Edge-Optimized**: Resource limits tailored for ARM64 Pi hardware
- **Automated Content**: CI/CD pipeline for preview generation and deployment

## Technical Architecture

### Core Components

The platform consists of several integrated components:

**Static File Server**
- **Base**: nginx optimized for ARM64 Raspberry Pi hardware
- **Content**: p5.js sketches with HTML, JavaScript, and assets
- **Security**: Non-root container with read-only filesystem
- **Performance**: Tuned for low-memory Pi environments

**Storage Foundation**
- **Backend**: local-path storage provisioner
- **Capacity**: 10Gi persistent volume for sketch content
- **Limitation**: Single-replica deployment (ReadWriteOnce constraint)
- **Future**: Ready for migration to SeaweedFS distributed storage

**Networking Integration**
- **Load Balancer**: MetalLB for external access
- **DNS**: external-dns automatic hostname management
- **SSL**: Future integration with cert-manager and Step-CA

### Container Configuration

The deployment leverages advanced Kubernetes security features:

```yaml
# Security hardening
security:
  runAsNonRoot: true
  runAsUser: 101              # nginx user
  runAsGroup: 101
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true

# Resource optimization for Pi hardware
resources:
  requests:
    memory: "32Mi"
    cpu: "50m"
  limits:
    memory: "64Mi"
    cpu: "100m"
```

### Deployment Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   GitHub Repo   │───▶│   Argo CD       │───▶│  Kubernetes     │
│   p5js-sketches │    │   GitOps        │    │  Deployment     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                                              │
         ▼                                              ▼
┌─────────────────┐                            ┌─────────────────┐
│ GitHub Actions  │                            │    nginx Pod    │
│ Preview Gen     │                            │  serving static │
└─────────────────┘                            │     content     │
                                               └─────────────────┘
```

## Automated Preview Generation System

### The Challenge

p5.js sketches are interactive, dynamic content that can't be represented by static screenshots. The platform needed a way to automatically generate compelling preview images that capture the essence of each sketch's visual output.

### Solution: Headless Browser Automation

The preview generation system uses Puppeteer for sophisticated browser automation:

**Technology Stack**
- **Puppeteer v21.5.0**: Headless Chrome automation
- **GitHub Actions**: CI/CD execution environment
- **Node.js**: Runtime environment for capture scripts
- **Canvas Capture**: Direct p5.js canvas element extraction

**Capture Process**
```javascript
const CONFIG = {
  sketches_dir: './sketches',
  capture_delay: 10000,           // Wait for sketch initialization
  animation_duration: 3000,       // Record animation period
  viewport: { width: 600, height: 600 },
  screenshot_options: {
    type: 'png',
    clip: { x: 0, y: 0, width: 400, height: 400 }  // Crop to canvas
  }
};
```

### Advanced Capture Features

**Sketch Lifecycle Awareness**
- **Initialization Delay**: Configurable per-sketch startup time
- **Animation Sampling**: Capture representative frames from animations
- **Canvas Detection**: Automatic identification of p5.js canvas elements
- **Error Handling**: Graceful fallback for problematic sketches

**GitHub Actions Integration**
```yaml
on:
  push:
    paths:
      - 'sketches/**'      # Trigger on sketch modifications
  workflow_dispatch:       # Manual execution capability
    inputs:
      force_regenerate:    # Regenerate all previews
      capture_delay:       # Configurable timing
```

**Automated Workflow**
1. **Trigger Detection**: Sketch files modified or manual dispatch
2. **Environment Setup**: Node.js, Puppeteer browser installation
3. **Dependency Caching**: Optimize build times with browser cache
4. **Preview Generation**: Execute capture script across all sketches
5. **Change Detection**: Identify new or modified preview images
6. **Auto-Commit**: Commit generated images back to repository
7. **Artifact Upload**: Preserve previews for debugging and archives

## Sketch Organization and Metadata

### Directory Structure

Each sketch follows a standardized organization pattern:

```
sketches/
├── linear-regression/
│   ├── index.html          # Entry point with p5.js setup
│   ├── sketch.js          # Main p5.js code
│   ├── style.css          # Styling and layout
│   ├── metadata.json      # Sketch configuration
│   ├── preview.png        # Auto-generated preview (400x400)
│   └── libraries/         # p5.js and extensions
│       ├── p5.min.js
│       └── p5.sound.min.js
└── robbie-the-robot/
    ├── index.html
    ├── main.js            # Entry point
    ├── robot.js           # Agent implementation
    ├── simulation.js      # GA evolution logic
    ├── world.js           # Environment simulation
    ├── ga-worker.js       # Web Worker for GA
    ├── metadata.json
    ├── preview.png
    └── libraries/
```

### Metadata Configuration

Each sketch includes rich metadata for gallery display and capture configuration:

```json
{
  "title": "Robby GA with Worker",
  "description": "Genetic algorithm simulation where robots learn to collect cans in a grid world using neural network evolution",
  "isAnimated": true,
  "captureDelay": 30000,
  "lastUpdated": "2025-08-04T19:06:01.506Z"
}
```

**Metadata Fields**
- **title**: Display name for gallery
- **description**: Detailed explanation of the sketch concept
- **isAnimated**: Indicates dynamic content requiring longer capture
- **captureDelay**: Custom initialization time in milliseconds
- **lastUpdated**: Automatic timestamp for version tracking

## Example Sketches

### Linear Regression Visualization

A educational demonstration of machine learning fundamentals:

**Purpose**: Interactive visualization of gradient descent optimization
**Features**:
- Real-time data point plotting
- Animated regression line fitting
- Loss function visualization
- Parameter adjustment controls

**Technical Implementation**:
- Single-file sketch with mathematical calculations
- Real-time chart updates using p5.js drawing primitives
- Interactive mouse controls for data manipulation

### Robbie the Robot - Genetic Algorithm

A sophisticated multi-agent simulation demonstrating evolutionary computation:

**Purpose**: Showcase genetic algorithms learning optimal can-collection strategies
**Features**:
- Multi-generational population evolution
- Neural network-based agent decision making
- Web Worker-based GA computation for performance
- Real-time fitness and generation statistics

**Technical Architecture**:
- **Main Thread**: p5.js rendering and user interaction
- **Web Worker**: Genetic algorithm computation (ga-worker.js)
- **Modular Design**: Separate files for robot, simulation, and world logic
- **Performance Optimization**: Efficient canvas rendering for multiple agents

## Deployment Integration

### Helm Chart Configuration

The platform uses Helm for templated Kubernetes deployments:

```yaml
# Chart.yaml
apiVersion: 'v2'
name: 'p5js-sketches'
description: 'P5.js Sketch Server - Static file server for hosting p5.js sketches'
type: 'application'
version: '0.0.1'
```

**Key Templates**:
- **Deployment**: nginx container with security hardening
- **Service**: LoadBalancer with MetalLB integration
- **ConfigMap**: nginx configuration optimization
- **Namespace**: Isolated environment for sketch server
- **ServiceAccount**: RBAC configuration for security

### Argo CD GitOps Integration

The platform deploys automatically via Argo CD:

**Repository Structure**:
- **Source**: `github.com/goldentooth/p5js-sketches`
- **Target**: `p5js-sketches` namespace
- **Sync Policy**: Automatic deployment on git changes
- **Health Checks**: Kubernetes-native readiness and liveness probes

**Deployment URL**: `https://p5js-sketches.services.k8s.goldentooth.net/`

## Gallery and User Experience

### Automated Gallery Generation

The platform includes sophisticated gallery generation:

**Features**:
- **Responsive Grid**: CSS Grid layout optimized for various screen sizes
- **Preview Integration**: Auto-generated preview images with fallbacks
- **Metadata Display**: Title, description, and technical details
- **Interactive Navigation**: Direct links to individual sketches
- **Search and Filter**: Future enhancement for large sketch collections

**Template System**:
```html
<!-- Gallery template with dynamic sketch injection -->
<div class="gallery-grid">
  {{#each sketches}}
  <div class="sketch-card">
    <img src="{{preview}}" alt="{{title}}" loading="lazy">
    <h3>{{title}}</h3>
    <p>{{description}}</p>
    <a href="{{url}}" class="sketch-link">View Sketch</a>
  </div>
  {{/each}}
</div>
```

### Performance Optimizations

**Client-Side Optimizations**:
- **Lazy Loading**: Progressive image loading for large galleries
- **Responsive Images**: Appropriate sizing for device capabilities
- **Cached Assets**: Efficient p5.js library delivery
- **Minimal Dependencies**: Lightweight gallery JavaScript

**Server-Side Optimizations**:
- **nginx Tuning**: Optimized for Pi hardware constraints
- **Gzip Compression**: Reduced bandwidth usage
- **Static Asset Caching**: Efficient browser cache policies
- **Connection Pooling**: Optimized for concurrent access

## Security and Hardening

### Container Security

The deployment implements comprehensive security measures:

**Filesystem Security**:
- **Read-Only Root**: Immutable container filesystem
- **Explicit Write Paths**: Limited writable directories for nginx
- **Non-Root Execution**: nginx runs as unprivileged user (101)

**Process Security**:
- **No New Privileges**: Prevents privilege escalation
- **Process Isolation**: Private tmp directories
- **System Protection**: Limited access to host system resources

**Network Security**:
- **Service Mesh Ready**: Prepared for future Istio integration
- **TLS Termination**: Ready for Step-CA certificate integration
- **Network Policies**: Future implementation for micro-segmentation

### Content Security

**Input Validation**:
- **Sketch Validation**: Automated checks for malicious content
- **File Type Restrictions**: Limited to approved extensions
- **Size Limits**: Prevent resource exhaustion attacks

**CI/CD Security**:
- **Automated Scanning**: Container image vulnerability assessment
- **Dependency Auditing**: npm audit integration
- **Secret Management**: GitHub token security for auto-commits

## Integration with Goldentooth Ecosystem

### Infrastructure Dependencies

The platform leverages existing Goldentooth infrastructure:

**Networking**:
- **MetalLB**: Load balancer IP assignment
- **external-dns**: Automatic DNS record management
- **HAProxy**: Potential future ingress integration

**Storage**:
- **local-path**: Current single-node storage solution
- **SeaweedFS Ready**: Migration path to distributed storage
- **Backup Integration**: Future automated content backup

**Monitoring**:
- **Prometheus**: Application and nginx metrics collection
- **Grafana**: Gallery usage and performance dashboards
- **Alerting**: Monitoring for sketch server availability

### Command Integration

Platform management integrates with goldentooth tooling:

```bash
# Check p5js-sketches deployment status
kubectl get pods -n p5js-sketches

# View application logs
kubectl logs -n p5js-sketches deployment/p5js-sketches

# Check storage utilization
kubectl get pvc -n p5js-sketches

# Restart deployment (refresh content)
kubectl rollout restart deployment/p5js-sketches -n p5js-sketches

# Argo CD sync status
kubectl get application gitops-repo-p5js-sketches -n argocd
```

# Nextflow Workflow Management System

## Overview

After successfully establishing a robust Slurm HPC cluster with comprehensive monitoring and observability, the next logical step was to add a modern workflow management system. Nextflow provides a powerful solution for data-intensive computational pipelines, enabling scalable and reproducible scientific workflows using software containers.

This chapter documents the installation and integration of Nextflow 24.10.0 with the existing Slurm cluster, complete with Singularity container support, shared storage integration, and module system configuration.

## The Challenge

While our Slurm cluster was fully functional for individual job submission, we lacked a sophisticated workflow management system that could:

- **Orchestrate Complex Pipelines**: Chain multiple computational steps with dependency management
- **Provide Reproducibility**: Ensure consistent results across different execution environments
- **Support Containers**: Leverage containerized software for portable and consistent environments
- **Integrate with Slurm**: Seamlessly submit jobs to our existing cluster scheduler
- **Enable Scalability**: Automatically parallelize workflows across cluster nodes

Modern bioinformatics and data science workflows often require hundreds of interconnected tasks, making manual job submission impractical and error-prone.

## Implementation Approach

The solution involved creating a comprehensive Nextflow installation that integrates deeply with our existing infrastructure:

### 1. **Architecture Design**
- **Shared Storage Integration**: Install Nextflow on NFS to ensure cluster-wide accessibility
- **Slurm Executor**: Configure native Slurm executor for distributed job execution
- **Container Runtime**: Leverage existing Singularity installation for reproducible environments
- **Module System**: Integrate with Lmod for consistent environment management

### 2. **Installation Strategy**
- **Java Runtime**: Install OpenJDK 17 as a prerequisite across all compute nodes
- **Centralized Installation**: Single installation on shared storage accessible by all nodes
- **Configuration Templates**: Create reusable configuration for common workflow patterns
- **Example Workflows**: Provide ready-to-run examples for testing and learning

## Technical Implementation

### New Ansible Role Creation

Created `goldentooth.setup_nextflow` role with comprehensive installation logic:

```yaml
# Install Java OpenJDK (required for Nextflow)
- name: 'Install Java OpenJDK (required for Nextflow)'
  ansible.builtin.apt:
    name:
      - 'openjdk-17-jdk'
      - 'openjdk-17-jre'
    state: 'present'

# Download and install Nextflow
- name: 'Download Nextflow binary'
  ansible.builtin.get_url:
    url: "https://github.com/nextflow-io/nextflow/releases/download/v{{ slurm.nextflow_version }}/nextflow"
    dest: "{{ slurm.nfs_base_path }}/nextflow/{{ slurm.nextflow_version }}/nextflow"
    owner: 'slurm'
    group: 'slurm'
    mode: '0755'
```

### Slurm Executor Configuration

Created comprehensive Nextflow configuration optimized for our cluster:

```groovy
// Nextflow Configuration for Goldentooth Cluster
process {
    executor = 'slurm'
    queue = 'general'

    // Default resource requirements
    cpus = 1
    memory = '1GB'
    time = '1h'

    // Enable Singularity containers
    container = 'ubuntu:20.04'

    // Process-specific configurations
    withLabel: 'small' {
        cpus = 1
        memory = '2GB'
        time = '30m'
    }

    withLabel: 'large' {
        cpus = 4
        memory = '8GB'
        time = '6h'
    }
}

// Slurm executor configuration
executor {
    name = 'slurm'
    queueSize = 100
    submitRateLimit = '10/1min'

    clusterOptions = {
        "--account=default " +
        "--partition=\${task.queue} " +
        "--job-name=nf-\${task.hash}"
    }
}
```

### Container Integration

Configured Singularity integration for reproducible workflows:

```groovy
singularity {
    enabled = true
    autoMounts = true
    envWhitelist = 'SLURM_*'

    // Cache directory on shared storage
    cacheDir = '/mnt/nfs/slurm/singularity/cache'

    // Mount shared directories
    runOptions = '--bind /mnt/nfs/slurm:/mnt/nfs/slurm'
}
```

### Module System Integration

Extended the existing Lmod system with a Nextflow module:

```lua
-- Nextflow Workflow Management System
whatis("Nextflow workflow management system 24.10.0")

-- Load required Java module (dependency)
depends_on("java/17")

-- Add Nextflow to PATH
prepend_path("PATH", "/mnt/nfs/slurm/nextflow/24.10.0")

-- Set Nextflow environment variables
setenv("NXF_HOME", "/mnt/nfs/slurm/nextflow/24.10.0")
setenv("NXF_WORK", "/mnt/nfs/slurm/nextflow/workspace")

-- Enable Singularity by default
setenv("NXF_SINGULARITY_CACHEDIR", "/mnt/nfs/slurm/singularity/cache")
```

### Example Pipeline

Created a comprehensive hello-world pipeline demonstrating cluster integration:

```groovy
#!/usr/bin/env nextflow

nextflow.enable.dsl=2

// Pipeline parameters
params {
    greeting = 'Hello'
    names = ['World', 'Goldentooth', 'Slurm', 'Nextflow']
    output_dir = './results'
}

process sayHello {
    tag "$name"
    label 'small'
    publishDir params.output_dir, mode: 'copy'

    container 'ubuntu:20.04'

    input:
    val name

    output:
    path "${name}_greeting.txt"

    script:
    """
    echo "${params.greeting} ${name}!" > ${name}_greeting.txt
    echo "Running on node: \$(hostname)" >> ${name}_greeting.txt
    echo "Slurm Job ID: \${SLURM_JOB_ID:-'Not running under Slurm'}" >> ${name}_greeting.txt
    """
}

workflow {
    names_ch = Channel.fromList(params.names)
    greetings_ch = sayHello(names_ch)

    workflow.onComplete {
        log.info "Pipeline completed successfully!"
        log.info "Results saved to: ${params.output_dir}"
    }
}
```

## Deployment Results

### Installation Success

The deployment was executed successfully across all Slurm compute nodes:

```bash
cd /Users/nathan/Projects/goldentooth/ansible
ansible-playbook -i inventory/hosts playbooks/setup_nextflow.yaml --limit slurm_compute
```

**Installation Summary:**
- ✅ **Java OpenJDK 17** installed on 9 compute nodes
- ✅ **Nextflow 24.10.0** downloaded and configured
- ✅ **Slurm executor** configured with resource profiles
- ✅ **Singularity integration** enabled with shared cache
- ✅ **Module file** created and integrated with Lmod
- ✅ **Example pipeline** deployed and tested

### Verification Output

```
Nextflow Installation Test:
      N E X T F L O W
      version 24.10.0 build 5928
      created 27-10-2024 18:36 UTC (14:36 GMT-04:00)
      cite doi:10.1038/nbt.3820
      http://nextflow.io

Installation paths:
- Nextflow: /mnt/nfs/slurm/nextflow/24.10.0
- Config: /mnt/nfs/slurm/nextflow/24.10.0/nextflow.config
- Examples: /mnt/nfs/slurm/nextflow/24.10.0/examples
- Workspace: /mnt/nfs/slurm/nextflow/workspace
```

## Configuration Management

### Usage Workflow

Users can now access Nextflow through the module system:

```bash
# Load the Nextflow environment
module load Nextflow/24.10.0

# Run the example pipeline
nextflow run /mnt/nfs/slurm/nextflow/24.10.0/examples/hello-world.nf

# Run with development profile (reduced resources)
nextflow run pipeline.nf -profile dev

# Run with custom configuration
nextflow run pipeline.nf -c custom.config
```

### Resource Profiles

The installation includes several predefined resource profiles:

- **small**: 1 CPU, 2GB RAM, 30 minutes
- **medium**: 2 CPUs, 4GB RAM, 2 hours
- **large**: 4 CPUs, 8GB RAM, 6 hours
- **gpu**: GPU queue with GRES allocation (prepared for future GPU integration)

### Execution Profiles

- **dev**: Development mode with reduced resources and queue limits
- **prod**: Production mode with full retry logic and larger queue sizes
- **debug**: Verbose logging and comprehensive reporting enabled

## Integration with Existing Stack

### Slurm Cluster Integration

Nextflow seamlessly integrates with our existing Slurm infrastructure:

- **Native Executor**: Uses Slurm's `sbatch` for job submission
- **Queue Management**: Automatically selects appropriate partitions
- **Resource Allocation**: Respects cluster resource limits and policies
- **Job Monitoring**: Integrates with existing Slurm monitoring tools

### Storage System Integration

- **NFS Shared Storage**: Workflow data accessible across all compute nodes
- **Container Cache**: Shared Singularity cache reduces download overhead
- **Workspace Management**: Centralized workspace for all workflow executions

### Observability Stack Integration

- **Prometheus Metrics**: Workflow execution metrics can be exported
- **Grafana Dashboards**: Potential for workflow performance visualization
- **Log Aggregation**: Integration with existing Loki logging infrastructure

## Performance Impact

### Resource Utilization

- **Memory Overhead**: Minimal Java runtime overhead (~100MB per workflow)
- **Storage Usage**: ~50MB for Nextflow installation, variable for containers
- **Network Impact**: Container pulls only occur once per cache

### Scalability Benefits

- **Parallel Execution**: Automatic parallelization of independent workflow tasks
- **Resource Optimization**: Dynamic resource allocation based on task requirements
- **Queue Management**: Intelligent job submission to minimize cluster congestion

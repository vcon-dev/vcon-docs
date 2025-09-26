---
description: Tracer Functionality in vCon Server
---

# 🩻 Conserver Tracers

### Overview

Tracer functionality in the vCon Server provides a powerful mechanism for observability, auditing, and compliance tracking as vCons (virtual conversations) flow through processing chains. Unlike processing links that transform or analyze vCon data, tracers are non-intrusive monitoring components that observe and record data flow without modifying the vCon content itself.

### Key Concepts

#### What are Tracers?

Tracers are side-effect modules that execute at specific points in the vCon processing pipeline to:

* Monitor data flow between processing links
* Create audit trails for compliance and security
* Track provenance and data lineage
* Generate observability metrics without affecting the main processing flow

#### Tracers vs. Links

| Aspect            | Links                          | Tracers                             |
| ----------------- | ------------------------------ | ----------------------------------- |
| Purpose           | Transform/process vCon data    | Observe/monitor data flow           |
| Data Modification | Can modify vCon content        | Never modify vCon content           |
| Execution Timing  | Sequential in processing chain | Execute before/after each link      |
| Return Value      | Return vCon UUID for chaining  | Return boolean success status       |
| Failure Impact    | Can stop processing chain      | Failures don't stop main processing |

### Execution Model

#### Tracer Invocation Points

Tracers are executed at three critical points in the processing pipeline:

1. Before First Link (link\_index = -1)

* Executes when a vCon enters the processing chain
* Records initial state and metadata

2. After Each Link (link\_index = 0, 1, 2, ...)

* Executes after each processing link completes
* Captures data transformations and flow

3. Chain Completion

* Executes when the entire processing chain finishes
* Records final state and completion metrics

#### Execution Flow

```
# Simplified execution flow
for link_index, link_name in enumerate(links):
    if link_index == 0:
        # Execute tracers before first link
        _process_tracers(vcon_id, vcon_id, links, -1)
    
    # Execute the processing link
    result = execute_link(link_name, vcon_id)
    
    # Execute tracers after link completion
    _process_tracers(result, vcon_id, links, link_index)

```

### Available Tracer Modules

#### 1. JLINC Zero-Knowledge Auditing

The JLINC tracer provides cryptographic signing and zero-knowledge audit capabilities for tamper-proof data provenance.

**Features**

* Cryptographic Signing: Creates tamper-proof signatures for vCon data
* Zero-Knowledge Auditing: Enables secure third-party auditing without exposing sensitive data
* Entity Management: Automatically creates and manages JLINC entities for each processing stage
* Data Hashing: Optionally hash vCon data for privacy-preserving audit trails
* Archive Integration: Stores audit records in external archive systems

#### Configuration

```
tracers:
  jlinc:
    module: tracers.jlinc
    options:
      data_store_api_url: http://jlinc-server:9090
      data_store_api_key: your_data_store_api_key
      archive_api_url: http://jlinc-server:9090
      archive_api_key: your_archive_api_key
      system_prefix: VCONTest
      agreement_id: 00000000-0000-0000-0000-000000000000
      hash_event_data: True
      dlq_vcon_on_error: True
```

**How JLINC Tracer Works**

**Entity Creation**: Creates JLINC entities for each processing stage

* System entity: {system\_prefix}-system@{domain}
* Link entities: {system\_prefix}-{link\_name}@{domain}

**Event Processing**: For each vCon transition:

* Retrieves vCon data from Redis
* Creates sender/recipient entities based on link context
* Optionally hashes vCon data for privacy
* Sends event to JLINC API for cryptographic signing

**Audit Trail**: Creates immutable audit records with:

* Cryptographic signatures
* Data hashes (if enabled)
* Metadata (vCon UUIDs, link information)
* Timestamps and provenance information

### Configuration

#### Basic Tracer Configuration

```yaml
tracers:
  tracer_name:
    module: tracers.module_name
    options:
      # Tracer-specific configuration options
```

#### Multiple Tracers

You can configure multiple tracers to run simultaneously:

```yaml
tracers:
  jlinc_audit:
    module: tracers.jlinc
    options:
      # JLINC configuration
      
  compliance_logger:
    module: tracers.compliance
    options:
      # Compliance logging configuration
      
  metrics_collector:
    module: tracers.metrics
    options:
```

### Tracer Interface

#### Required Function Signature

All tracer modules must implement a run function with this signature:

```python
def run(
    in_vcon_uuid: str,      # Input vCon UUID
    out_vcon_uuid: str,     # Output vCon UUID  
    tracer_name: str,       # Name of this tracer instance
    links: list[str],       # List of all links in the chain
    link_index: int,        # Current link index (-1 for pre-chain)
    opts: dict = {}         # Tracer configuration options
) -> bool:
    """
    Execute tracer logic for a vCon processing step.
    
    Args:
        in_vcon_uuid: UUID of vCon entering the processing step
        out_vcon_uuid: UUID of vCon exiting the processing step
        tracer_name: Name of this tracer instance from config
        links: Complete list of links in the processing chain
        link_index: Index of current link (-1 for pre-chain execution)
        opts: Tracer-specific configuration options
        
    Returns:
        bool: True if tracer executed successfully, False otherwise
    """
```

#### Implementation Example

```python
from lib.logging_utils import init_logger
from lib.vcon_redis import VconRedis

logger = init_logger(__name__)

default_options = {
    "api_url": "http://example.com/api",
    "api_key": "",
    "enabled": True
}

def run(in_vcon_uuid, out_vcon_uuid, tracer_name, links, link_index, opts=default_options):
    """Example tracer implementation"""
    
    if not opts.get("enabled", True):
        logger.debug(f"Tracer {tracer_name} is disabled")
        return True
    
    try:
        # Get vCon data
        vcon_redis = VconRedis()
        vcon_obj = vcon_redis.get_vcon(out_vcon_uuid)
        
        if not vcon_obj:
            logger.error(f"Could not retrieve vCon {out_vcon_uuid}")
            return False
        
        # Process tracer logic
        logger.info(f"Executing {tracer_name} tracer for vCon {out_vcon_uuid}")
        
        # Your tracer logic here
        # - Send data to external systems
        # - Create audit records
        # - Generate metrics
        # - Log compliance information
        
        return True
        
    except Exception as e:
        logger.error(f"Tracer {tracer_name} failed: {e}")
        return False
        
        
```

### Use Cases

#### 1. Compliance and Auditing

* GDPR Compliance: Track data processing for privacy regulations
* SOX Compliance: Audit financial conversation processing
* HIPAA Compliance: Monitor healthcare conversation handling

#### 2. Security and Integrity

* Data Provenance: Track data lineage and transformations
* Tamper Detection: Cryptographic verification of data integrity
* Access Logging: Record who accessed what data when

#### 3. Observability and Monitoring

* Performance Metrics: Track processing times and throughput
* Error Tracking: Monitor failures and exceptions
* Business Metrics: Count conversations, analyze patterns

#### 4. Data Governance

* Retention Tracking: Monitor data lifecycle and expiration
* Data Classification: Track sensitive data handling
* Cross-Border Transfers: Monitor international data flows

### Best Practices

#### 1. Non-Blocking Design

* Tracers should never block the main processing flow
* Handle errors gracefully without affecting vCon processing
* Use asynchronous operations where possible

#### 2. Performance Considerations

* Keep tracer execution time minimal
* Cache frequently accessed data
* Use efficient data serialization

#### 3. Error Handling

* Log errors but don't raise exceptions
* Return boolean status for success/failure
* Implement retry logic for external API calls

#### 4. Privacy and Security

* Hash or encrypt sensitive data before external transmission
* Follow data minimization principles
* Implement proper authentication and authorization

#### 5. Configuration Management

* Provide sensible defaults
* Validate configuration options
* Support environment-specific settings

### Monitoring and Debugging

#### Logging

Tracers automatically log their execution with structured logging:

```python
logger.info(
    "Completed tracer %s (module: %s) for vCon: %s in %s seconds",
    tracer_name,
    tracer_module_name,
    out_vcon_uuid,
    tracer_processing_time,
    extra={
        "tracer_processing_time": tracer_processing_time,
        "tracer_name": tracer_name,
        "tracer_module_name": tracer_module_name
    }
)
```

#### Metrics

Tracer execution is automatically tracked with:

* Processing time per tracer
* Success/failure rates
* vCon throughput metrics

#### Debugging

* Enable debug logging for detailed tracer execution
* Use tracer-specific configuration for testing
* Monitor external API responses and errors

### Future Extensions

The tracer system is designed to be extensible. Potential future tracer modules could include:

* DataTrails Integration: Blockchain-based audit trails
* SIEM Integration: Security information and event management
* Custom Analytics: Business intelligence and reporting
* Data Loss Prevention: Monitor for sensitive data exposure
* Performance Profiling: Detailed performance analysis

### Conclusion

Tracer functionality provides a powerful, non-intrusive way to add observability, compliance, and security monitoring to vCon processing pipelines. By executing alongside the main processing flow without affecting it, tracers enable comprehensive data governance and audit capabilities while maintaining system performance and reliability.The modular design allows for easy extension with custom tracer implementations, making it possible to integrate with any external system or compliance framework while maintaining the core principle of non-interference with vCon processing.

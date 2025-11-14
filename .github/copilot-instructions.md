# DQL Core Use Cases

This document outlines 7 fundamental DQL use cases with practical query examples for observability and monitoring.

## Quick Use Case Overview

| Use Case | What It Does | Key Benefit |
|----------|--------------|-------------|
| **0. Event Analysis** | Review various events captured from deployments, Kubernetes, automatically detected by Davis AI, and more. | Events represent the current (and past) state of the environment across many different entities to give context. |
| **1. Error & Exception Analysis** | Find and analyze errors across logs and traces | Rapid troubleshooting and root cause identification |
| **2. Performance Monitoring** | Track CPU, latency, and service response times | SLA monitoring and performance optimization |
| **3. Kubernetes Observability** | Monitor pod health and container resource usage | Cluster health and resource optimization |
| **4. Service Dependency Analysis** | Trace transactions across distributed services | Understand complex microservices architectures |
| **5. Business Event Analytics** | Track user actions and conversion funnels | Measure business KPIs and user behavior |
| **6. Vulnerability Management** | Identify and prioritize security vulnerabilities | Risk-based security remediation |
| **7. Smartscape Topology** | Discover entities and infrastructure relationships | Complete environment visibility and dependency mapping |

---

### Tips and Tricks
- Sampling is especially important with log and span data. Here is an example
  - ```fetch logs, from:now()-30d, samplingRatio:100 | limit 1000```
  - ```fetch spans, from:now()-2h, samplingRatio:10 | limit 100```

## 0. Event Analysis

**Purpose**: Analyze events ingested into Dynatrace and those automatically detected by Davis AI. Events give great context to the current (and past) state of the environment.

**Use Cases**:
- Identify any problems in current applications/systems
- Correlate any relevant deployment events
- Identify trends in behavior over time looking at event patterns

### Query 1: Basic Event Discovery

**Purpose**: Grab any type of event and look at relevant fields to understand different dimensions that can be used for more advanced analysis.

### DQL Query

```dql
fetch events, from:now()-7d
```

**What this shows**:
- Recent events
- Understand basic event structure to identify available dimensions

### Query 2: Advanced Problem Analysis

**Purpose**: Focus on problems detected by Davis AI. These events represent actual problems that should be addressed.

### DQL Query

```dql
fetch events
| filter event.kind == "DAVIS_PROBLEM"
| filter event.status == "ACTIVE"
| filter dt.davis.is_duplicate == false
| fields event.id, display_id, affected_entity_ids, event.category, event.status, event.description, root_cause_entity_id, root_cause_entity_name
```

**What this shows**:
- Active problems detected by Davis AI
- Understand what problems are currently (or historically) occurring that need to be addressed.

## 1. Error and Exception Analysis

**Purpose**: Monitor and analyze application errors, exceptions, and issues across logs, spans, and events to quickly identify problems and their root causes.

**Use Cases**:
- Identify error trends and spikes across services
- Analyze exception stack traces and error messages
- Correlate errors across different data sources (logs, traces, events)
- Track error rates by service, environment, or deployment
- Find the most common error types and their frequency

**Key DQL Patterns**:
- Filter by log level (ERROR, SEVERE)
- Expand span.events to access exception details
- Aggregate error counts with summarize
- Group by service, host, or error type

### Query 1: Basic Error Log Discovery

**Purpose**: Understand the basic structure of error logs and see recent error messages.

### DQL Query

```dql
fetch logs
| filter loglevel == "ERROR"
| fields timestamp, loglevel, log.source, content
| limit 10
```

**What this shows**:
- Recent error log entries with timestamps
- Log sources (Container Output, Journald, etc.)
- Full error message content
- Useful for quick error investigation and understanding log structure

### Query 2: Advanced Error Analysis with Aggregation

**Purpose**: Identify which services/namespaces have the most errors and prioritize troubleshooting efforts.

### DQL Query

```dql
fetch logs
| filter loglevel == "ERROR" or loglevel == "SEVERE"
| summarize error_count = count(), by:{k8s.namespace.name, log.source, loglevel}
| sort error_count desc
| limit 20
```

**What this shows**:
- Total error counts grouped by Kubernetes namespace
- Breakdown by log source (Container vs System logs)
- Sorted by highest error count for prioritization
- Identifies problematic services requiring immediate attention

**Insights**:
- Prioritize namespaces with disproportionately high error counts
- Compare error rates across namespaces to identify outliers
- Track error trends over time to detect degradation
- Use error distribution to guide troubleshooting efforts

---

## 2. Performance Monitoring and Metrics

**Purpose**: Track service response times, resource utilization, and performance metrics over time to ensure applications meet SLA requirements and identify performance degradation.

**Use Cases**:
- Monitor service response time percentiles (p50, p95, p99)
- Track CPU, memory, and disk utilization trends
- Compare performance across different time windows
- Identify slow endpoints or database queries
- Detect performance anomalies and degradation

**Key DQL Patterns**:
- Timeseries aggregation for time-based charts
- Percentile calculations for latency analysis
- Scalar metrics for current values
- Group by service or host for comparison

### Query 1: Basic Host CPU Usage

**Purpose**: Monitor current CPU utilization across all hosts to identify resource constraints.

### DQL Query

```dql
timeseries cpu_usage = avg(dt.host.cpu.usage, scalar:true),
  by:{dt.entity.host},
  from:now()-5m
| fieldsAdd host_name = entityName(dt.entity.host)
| fields host_name, cpu_usage
| sort cpu_usage desc
| limit 10
```

**What this shows**:
- Current average CPU usage percentage per host (scalar value)
- Human-readable host names via `entityName()`
- Sorted by highest CPU utilization
- Identifies hosts under high load

**Insights**:
- Identify hosts approaching capacity thresholds (>70-80%)
- Detect CPU bottlenecks requiring capacity planning
- Compare CPU usage across hosts to identify imbalances
- Track trends over time to predict capacity needs

### Query 2: Advanced Service Latency Analysis with Percentiles

**Purpose**: Analyze service response times across percentiles to identify slow services and SLA violations.

### DQL Query

```dql
fetch spans, from:now()-1h
| summarize {
    p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99),
    request_count = count()},
    by:{service_name = entityName(dt.entity.service)}
| sort p95 desc
| limit 10
```

**What this shows**:
- P50 (median), P95, and P99 latency for each service
- Total request count per service
- Identifies outliers and tail latency issues
- Duration in nanoseconds (divide by 1,000,000 for milliseconds)

**Insights**:
- Identify services with high tail latency (large P99-P50 gap indicates variability)
- Prioritize optimization efforts on services with both high latency and high request volume
- Services with multi-second P95 latency may indicate architectural issues
- Low-latency services handling high request volume demonstrate good optimization
- Use percentiles to set and monitor SLA targets

---


## 3. Kubernetes and Container Observability

**Purpose**: Monitor Kubernetes clusters, pods, containers, and their resource usage to ensure cluster health and optimize resource allocation.

**Use Cases**:
- Track pod restarts and container failures
- Monitor resource consumption by namespace or pod
- Identify unhealthy containers or nodes
- Analyze logs from specific Kubernetes workloads
- Track deployment events and rollout status

**Key DQL Patterns**:
- Filter by k8s.* fields (namespace, pod, cluster)
- Aggregate by Kubernetes dimensions
- Join container metrics with application logs
- Track state changes and events

### Query 1: Basic Kubernetes Namespace Activity

**Purpose**: Get an overview of log activity and pod distribution across Kubernetes namespaces.

### DQL Query

```dql
fetch logs, from:now()-1h
| filter isNotNull(k8s.namespace.name)
| summarize {
    pod_count = countDistinct(k8s.pod.name),
    log_count = count()},
    by:{k8s.cluster.name, k8s.namespace.name}
| sort log_count desc
| limit 10
```

**What this shows**:
- Number of unique pods per namespace
- Total log volume by namespace
- Cluster and namespace breakdown
- Identifies most active namespaces

**Insights**:
- High log volume may indicate active applications or excessive logging
- Single-pod namespaces with high volume may need scaling
- System namespaces (kube-system) typically have lower, stable log volume
- Use as baseline to detect unusual activity spikes

### Query 2: Advanced Pod Health and Error Analysis

**Purpose**: Identify problematic pods with high error rates and analyze health issues at the pod level.

### DQL Query

```dql
fetch logs, from:now()-1h
| filter isNotNull(k8s.namespace.name)
| summarize {
    error_count = countIf(loglevel == "ERROR" or loglevel == "SEVERE"),
    warning_count = countIf(loglevel == "WARN" or loglevel == "WARNING"),
    total_logs = count(),
    error_rate = countIf(loglevel == "ERROR" or loglevel == "SEVERE") * 100.0 / count()},
    by:{k8s.cluster.name, k8s.namespace.name, k8s.pod.name}
| filter error_count > 0
| sort error_count desc
| limit 15
```

**What this shows**:
- Error counts and rates per pod
- Warning counts for additional context
- Total log volume to understand scale
- Calculated error percentage for severity assessment

**Insights**:
- Pods with error rates >10% likely have critical issues requiring immediate attention
- Compare error rates across pods to identify outliers
- High error count with low error rate may indicate high traffic, not necessarily problems
- Error rates <1% typically indicate healthy pod operation
- Use error patterns to determine if pod restart, scaling, or code fixes are needed

---

## 4. Service Dependency and Trace Analysis

**Purpose**: Analyze distributed traces, understand service dependencies, and troubleshoot transaction flows across microservices architectures.

**Use Cases**:
- Trace individual requests across multiple services
- Identify slow service calls in a transaction
- Map service-to-service communication patterns
- Analyze span duration and identify bottlenecks
- Find failed transactions and their error points

**Key DQL Patterns**:
- Filter spans by trace.id for specific transactions
- Expand span.events for detailed trace events
- Join spans with service metadata
- Calculate span duration and latency

### Query 1: Basic Slowest Spans Analysis

**Purpose**: Identify the slowest individual operations across all services to find performance bottlenecks.

### DQL Query

```dql
fetch spans, from:now()-30m
| fields trace.id, span.name, duration, service = entityName(dt.entity.service), span.kind
| sort duration desc
| limit 10
```

**What this shows**:
- Individual span operations sorted by duration
- Service name and operation type (client/server)
- Trace ID for further investigation
- Duration in nanoseconds (600 seconds = 600,000,000,000 ns)

**Insights**:
- Long-duration spans may indicate streaming connections, background jobs, or slow operations
- Differentiate between architectural patterns (streaming) vs performance problems
- Client/server span pairs help identify which side is slow
- Use trace IDs to drill into full transaction context
- Filter out expected long-running operations to find actual bottlenecks

### Query 2: Advanced Trace Complexity Analysis

**Purpose**: Identify complex distributed traces with many service calls to understand transaction flows and dependencies.

### DQL Query

```dql
fetch spans, from:now()-1h
| summarize {
    span_count = count(),
    avg_duration = avg(duration),
    max_duration = max(duration)},
    by:{trace.id, root_service = entityName(dt.entity.service)}
| filter span_count > 5
| sort span_count desc
| limit 10
```

**What this shows**:
- Number of spans (service calls) per trace
- Average and maximum duration within each trace
- Root service that initiated the transaction
- Identifies most complex transaction patterns

**Insights**:
- High span counts (>20) indicate complex microservices architectures
- Compare average vs max duration to identify bottleneck spans
- Deep call chains increase latency and failure risk
- Consider optimization opportunities:
  - Batching multiple service calls
  - Caching frequently accessed data
  - Reducing service mesh overhead
  - Simplifying overly complex transaction flows
- Monitor for cascade failures in traces with many dependencies

---

## 5. Business Event Analytics

**Purpose**: Track and analyze custom business events, user actions, and application-specific metrics to understand business KPIs and user behavior.

**Use Cases**:
- Track conversion funnel metrics (views → clicks → purchases)
- Calculate revenue and transaction volumes
- Analyze user behavior patterns
- Monitor business KPIs in real-time
- Segment analysis by product, region, or user type

**Key DQL Patterns**:
- Fetch bizevents for custom business data
- Aggregate amounts, counts, and revenues
- Time-based aggregation for trends
- Filter by event.type for specific business actions

### Query 1: Basic Business Event Discovery

**Purpose**: Understand what business events are being tracked and their volume to identify key user actions.

### DQL Query

```dql
fetch bizevents, from:now()-1h
| summarize event_count = count(), by:{event.type}
| sort event_count desc
| limit 10
```

**What this shows**:
- All business event types being tracked
- Volume of each event type
- Most frequent user actions
- Business activity patterns

**Insights**:
- High-volume events indicate key user interactions
- Low-volume critical events (e.g., purchases) may need special attention
- Event distribution reveals product usage patterns
- Compare event ratios to identify funnel drop-offs
- Unexpected event volumes may indicate tracking issues or user behavior changes

### Query 2: Advanced Conversion Funnel Analysis

**Purpose**: Calculate conversion rates across the user journey from homepage to checkout completion.

### DQL Query

```dql
fetch bizevents, from:now()-1h
| summarize
    total_events = count(),
    checkout_count = countIf(event.type == "astroshop.web.checkout_success"),
    cart_views = countIf(event.type == "astroshop.web.cart"),
    product_views = countIf(event.type == "astroshop.web.products"),
    home_views = countIf(event.type == "astroshop.web.home")
| fieldsAdd
    conversion_rate = (checkout_count * 100.0) / home_views,
    cart_to_checkout = (checkout_count * 100.0) / cart_views
```

**What this shows**:
- Complete user journey funnel from home to checkout
- Conversion rates at each stage
- Drop-off points in the purchase flow
- Overall business health metrics

**Insights**:
- Calculate home-to-checkout conversion to measure overall effectiveness
- Multiple product views per home view indicates browsing behavior
- High cart view count relative to checkouts may indicate abandonment
- Compare conversion rates across time periods to track improvements
- Use funnel analysis to:
  - Identify drop-off points requiring optimization
  - Measure impact of UI/UX changes
  - Set and monitor conversion rate goals
  - Detect unusual behavior patterns (bots, testing)

---

## 6. Vulnerability and Security Risk Management

**Purpose**: Identify vulnerabilities and misconfigurations in production environments and prioritize them by risk of exploitation to guide remediation efforts.

**Use Cases**:
- Discover open security vulnerabilities across all services
- Prioritize vulnerabilities by exploitation risk and severity
- Track CVE (Common Vulnerabilities and Exposures) identifiers
- Identify affected entities and their exposure status
- Assess Davis Security Score and risk factors
- Monitor vulnerability trends and remediation progress

**Key DQL Patterns**:
- Fetch security.events for vulnerability and posture management data
- Filter by event.type for specific security findings
- Dedup vulnerabilities by display_id
- Analyze by severity and affected entities
- Monitor vulnerability resolution status

### DQL Query

```dql
fetch security.events
| filter event.provider == "Dynatrace"
| filter event.type == "VULNERABILITY_STATE_REPORT_EVENT"
| filter event.level == "VULNERABILITY"
| filter vulnerability.resolution.status == "OPEN"
     AND vulnerability.mute.status != "MUTED"
| dedup {vulnerability.display_id, affected_entity.id}, sort:{timestamp desc}
| fieldsAdd vulnerability.risk.level=if(vulnerability.risk.score>=9,"CRITICAL",
                                     else:if(vulnerability.risk.score>=7,"HIGH",
                                     else:if(vulnerability.risk.score>=4,"MEDIUM",
                                     else:if(vulnerability.risk.score>=0.1,"LOW",
                                     else:"NONE"))))
| sort {vulnerability.risk.score, direction:"descending"}, {affected_entities, direction:"descending"}
| limit 50
```

**What this shows**:
- Open, unmuted vulnerabilities sorted by exploitation risk
- Risk level calculated from Davis Security Score (CRITICAL, HIGH, MEDIUM, LOW)
- Vulnerabilities with highest risk scores appear first
- Deduplicated by vulnerability and affected entity
- Shows latest state of each vulnerability

**Insights**:
- **Focus on high-risk vulnerabilities first**: Risk score combines severity, exploitability, and exposure
- Vulnerabilities with **public exploits** and **internet exposure** are highest priority
- **Davis Security Score** considers real-world context (not just CVE severity)
- Track which entities have multiple vulnerabilities for holistic remediation
- Monitor risk score trends to measure security posture improvement

### Query 2: Specific CVE Impact Analysis

**Purpose**: Analyze the impact of a specific CVE across all affected entities to coordinate targeted remediation efforts.

### DQL Query

```dql
fetch security.events
| filter event.provider == "Dynatrace"
| filter event.type == "VULNERABILITY_STATE_REPORT_EVENT"
| filter event.level == "ENTITY"
| dedup {vulnerability.display_id, affected_entity.id}, sort:{timestamp desc}
| filter in("CVE-2023-41419",vulnerability.references.cve)
| filter vulnerability.resolution.status == "OPEN"
     AND vulnerability.parent.mute.status != "MUTED"
     AND vulnerability.mute.status != "MUTED"
```

**What this shows**:
- All entities affected by a specific CVE
- Risk assessment for each affected entity
- Davis Security Score and risk level per entity
- Exposure and exploit status
- Deduplicated to show latest state per entity

**Insights**:
- **Identify blast radius** of a specific CVE across your environment
- Prioritize patching entities with **highest risk scores** or **internet exposure**
- Coordinate remediation: patch similar entities together (e.g., all instances of same service)
- Track CVE-specific remediation progress
- Use for:
  - **Emergency patching** when critical CVEs are announced
  - **Compliance reporting** for specific vulnerabilities
  - **Impact assessment** before applying patches
  - **Coordinated disclosure** response planning

**Usage**: Replace `CVE-2023-41419` with any CVE identifier you're investigating.

**Note**: These queries require **Dynatrace Application Security** module. If no results are returned, Application Security may not be enabled in your environment.

## 7. Smartscape Environment Discovery and Topology Mapping

**Purpose**: Get a comprehensive overview of the monitored environment, understand entity relationships, and map infrastructure topology for dependency analysis.

**Use Cases**:
- Discover all monitored entities and their counts
- Map service dependencies and communication paths
- Understand infrastructure relationships (service → process → host)
- Analyze multi-hop dependency chains
- Generate topology views for specific entity types
- Track entity relationships for impact analysis

**Key DQL Patterns**:
- Fetch dt.entity.* for entity discovery
- Summarize entity counts by type
- Use entity.name and id for identification
- Cross-reference with logs/spans for operational context

### Query 1: Basic Environment Inventory

**Purpose**: Get a complete inventory of all monitored services in your environment.

### DQL Query

```dql
fetch dt.entity.service
| fields entity.name, id
| sort entity.name
| limit 100
```

**What this shows**:
- All monitored services with their names
- Dynatrace entity IDs for each service
- Service naming patterns (Kubernetes, cloud functions, etc.)
- Total service count in environment

**Insights**:
- Use for comprehensive service inventory and documentation
- Identify naming conventions to understand service organization
- Spot orphaned or misconfigured services (unexpected naming)
- Track service proliferation and growth over time
- Entity IDs enable correlation with logs, spans, and metrics
- Compare service counts across environments (dev vs prod)

### Query 2: Advanced Multi-Entity Type Discovery

**Purpose**: Discover all entities across your infrastructure, and their relationships for comprehensive topology understanding.

### DQL Query

```dql
fetch dt.entity.host
| summarize host_count = count()
| fieldsAdd entity_type = "HOST"
```

**Alternative queries for other entity types**:

### DQL Query

```dql
smartscapeEdges "*" | limit 100
```

### DQL Query

```dql
smartscapeNodes "*" | limit 100
```

**What smartscapeNodes shows**:
- All monitored entities with full metadata
- Entity types: Kubernetes resources, AWS resources, services, hosts, etc.
- Entity properties: configurations, tags, labels, annotations
- Lifetime information (when entities were first/last seen)
- Cloud provider details (AWS ARN, region, availability zone)
- Kubernetes details (cluster, namespace, object definitions)

**What smartscapeEdges shows**:
- Relationships between entities (source → target)
- Relationship types:
  - `belongs_to`: Entity hierarchy (Ingress → Namespace → Cluster)
  - `routes_to`: Traffic routing (Ingress → Service)
  - `runs_on`: Compute placement (Subnet → Availability Zone)
  - `is_attached_to`: Network connections (Subnet → VPC)
  - `calls`: Service-to-service communication

**Expected Results**:

**Nodes** (entities):
- Infrastructure: Hosts, process groups, containers
- Kubernetes: Clusters, namespaces, services, ingresses, pods
- Cloud providers: AWS/Azure/GCP resources (subnets, VPCs, zones)
- Applications: Services, databases, message queues
- Full metadata including configurations, tags, and labels

**Edges** (relationships):
- `belongs_to`: Hierarchical relationships (Ingress → Namespace → Cluster)
- `routes_to`: Traffic routing (Ingress → Service, Load Balancer → Backend)
- `runs_on`: Compute placement (Container → Host, Subnet → Availability Zone)
- `is_attached_to`: Network connections (Subnet → VPC, Volume → Instance)
- `calls`: Service communication (Service A → Service B)

**Insights**:
- **Complete topology map**: Understand how all components connect
- **Dependency analysis**: Trace impact of changes through relationships
- **Blast radius calculation**: Identify what's affected when an entity fails
- **Multi-cloud visibility**: See AWS, Azure, Kubernetes resources together
- **Relationship types** guide troubleshooting:
  - `belongs_to` → Organizational hierarchy
  - `routes_to` → Traffic flow paths
  - `runs_on` → Compute dependencies
  - `calls` → Service communication patterns
- **Use for**:
  - Migration planning (understand dependencies before moving)
  - Impact analysis (what breaks if this entity goes down)
  - Security review (identify unexpected connections)
  - Cost optimization (find unused or orphaned resources)



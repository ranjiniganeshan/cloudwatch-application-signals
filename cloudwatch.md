Application Signals is an AWS CloudWatch feature that provides automatic discovery and monitoring of your application's services without requiring manual instrumentation configuration. It's designed for observability of microservices architectures.
How It Works
1. Auto-Instrumentation
Application Signals uses auto-instrumentation agents (for Java, Python, or .NET applications) that automatically:
Capture application metrics
Generate distributed traces via AWS X-Ray
Discover service dependencies
When you enable Application Signals on an EKS cluster, it installs:
The CloudWatch agent for collecting metrics
Language-specific auto-instrumentation agents that inject tracing code into your applications at runtime
2. Service Discovery
Once enabled, Application Signals automatically discovers and populates a list of services without additional setup. It identifies:
Individual microservices in your application
The API operations each service exposes
Dependencies between services (including AWS services and third-party services)
3. Metrics Collection
Application Signals collects standard application metrics including:
Latency - How long operations take
Availability - Service uptime and error rates
Request/response patterns - Traffic flowing through your services
4. Distributed Tracing
Using X-Ray integration, it:
Traces requests as they flow through multiple microservices
Correlates traces with metric anomalies
Allows you to drill down from a metric spike to the actual traces that caused it
5. Service Level Objectives (SLOs)
You can define SLOs for your services, such as:
"Availability for Searching an Owner"
"Latency for Registering an Owner"
These help you monitor if your services are meeting performance targets.
Architecture Flow (using the Pet Clinic example)
┌─────────────────────────────────────────────────────────────────┐│                        Amazon EKS Cluster                        ││  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              ││  │ API Gateway │──│  Customers  │──│    Vets     │              ││  │   Service   │  │   Service   │  │   Service   │              ││  └──────┬──────┘  └─────────────┘  └─────────────┘              ││         │              │                  │                      ││         └──────────────┴──────────────────┘                      ││                        │                                         ││              ┌─────────▼─────────┐                              ││              │  Auto-Instrument  │                              ││              │      Agent        │                              ││              └─────────┬─────────┘                              │└────────────────────────┼────────────────────────────────────────┘                         │         ┌───────────────┼───────────────┐         ▼               ▼               ▼   ┌──────────┐   ┌──────────┐   ┌──────────┐   │CloudWatch│   │  X-Ray   │   │CloudWatch│   │  Metrics │   │  Traces  │   │   Logs   │   └────┬─────┘   └────┬─────┘   └────┬─────┘        │              │              │        └──────────────┼──────────────┘                       ▼              ┌────────────────┐              │  Application   │              │    Signals     │              │   Dashboard    │              └────────────────┘
Key Benefits
Zero-code instrumentation - No need to modify your application code
Unified view - See services, operations, and dependencies in one place
Correlation - Link metrics to traces to quickly identify root causes
SLO monitoring - Track service performance against defined objectives
Required IAM Permissions
Application Signals needs these policies:
AWSXrayWriteOnlyAccess - To send trace data
CloudWatchAgentServerPolicy - To send metrics
Additional permissions for cloudwatch:PutMetricData, xray:GetServiceGraph, logs:StartQuery, etc.
This approach is particularly useful for microservices architectures where manually instrumenting dozens of services would be time-consuming and error-prone.
docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Signals-Enable-EKS.html
  refer this and explain internally how it is collecting metrics, logs and traces

Based on the AWS documentation, let me explain the internal mechanisms of how Application Signals collects metrics, logs, and traces.
Internal Architecture of Application Signals Data Collection
1. The Core Technology: OpenTelemetry Auto-Instrumentation
Application Signals uses AWS Distro for OpenTelemetry (ADOT) as its foundation. When you enable Application Signals, the system injects ADOT auto-instrumentation SDKs into your application pods.
┌─────────────────────────────────────────────────────────────────────────┐│                           Application Pod                                ││  ┌──────────────────────────────────────────────────────────────────┐  ││  │                    Your Application Code                          │  ││  │                   (Java/Python/.NET/Node.js)                      │  ││  └───────────────────────────┬──────────────────────────────────────┘  ││                              │                                          ││  ┌───────────────────────────▼──────────────────────────────────────┐  ││  │              ADOT Auto-Instrumentation Agent                      │  ││  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐   │  ││  │  │   Metrics   │  │   Traces    │  │   Context Propagation   │   │  ││  │  │  Collector  │  │  Collector  │  │   (X-Ray, W3C, B3)      │   │  ││  │  └─────────────┘  └─────────────┘  └─────────────────────────┘   │  ││  └───────────────────────────┬──────────────────────────────────────┘  │└──────────────────────────────┼──────────────────────────────────────────┘                               │ OTLP Protocol                               ▼                    ┌──────────────────────┐                    │   CloudWatch Agent   │                    │  (DaemonSet/Sidecar) │                    └──────────┬───────────┘                               │           ┌───────────────────┼───────────────────┐           ▼                   ▼                   ▼    ┌────────────┐     ┌────────────┐     ┌────────────┐    │ CloudWatch │     │   X-Ray    │     │ CloudWatch │    │  Metrics   │     │  Service   │     │    Logs    │    └────────────┘     └────────────┘     └────────────┘
2. How Instrumentation Gets Injected
When you add the annotation to your workload:
annotations:  instrumentation.opentelemetry.io/inject-java: "true"    # For Java  instrumentation.opentelemetry.io/inject-python: "true"  # For Python  instrumentation.opentelemetry.io/inject-dotnet: "true"  # For .NET  instrumentation.opentelemetry.io/inject-nodejs: "true"  # For Node.js
The CloudWatch Observability EKS Add-on uses a Kubernetes Mutating Admission Webhook that:
Intercepts pod creation requests
Modifies the pod spec to inject an init container with the ADOT SDK
Adds environment variables that configure the instrumentation
Restarts the pod with the instrumentation enabled
3. Data Collection Mechanisms
Metrics Collection
The ADOT agent automatically captures application-level metrics by hooking into:
What's Instrumented	Metrics Captured
HTTP frameworks (Spring, Flask, Express, ASP.NET)	Request count, latency, error rates
Database clients (JDBC, SQLAlchemy, etc.)	Query duration, connection pool stats
HTTP clients	Outbound call latency, status codes
AWS SDK calls	Service call duration, errors
These metrics are exported via OTLP (OpenTelemetry Protocol) to the CloudWatch Agent:
Endpoint: http://cloudwatch-agent.amazon-cloudwatch:4316/v1/metricsProtocol: OTLP over HTTP/protobuf
Traces Collection
Distributed tracing works by:
Span Creation: The ADOT agent creates spans for each operation (HTTP request, DB query, etc.)
Context Propagation: Trace context is propagated across service boundaries using headers
Sampling: X-Ray sampler determines which traces to capture
Traces Endpoint: http://cloudwatch-agent.amazon-cloudwatch:4316/v1/tracesSampler Endpoint: http://cloudwatch-agent.amazon-cloudwatch:2000
The configuration uses multiple propagators for compatibility:
OTEL_PROPAGATORS='tracecontext,baggage,b3,xray'
This means traces can be correlated across services using:
W3C Trace Context - Standard trace propagation
X-Ray - AWS native tracing format
B3 - Zipkin compatibility
Logs Collection
Logs are correlated with traces through:
Trace ID and Span ID injection into log records
CloudWatch Agent collecting logs and enriching them with metadata
Log groups associated with services for correlation in the console
4. The CloudWatch Agent's Role
The CloudWatch Agent (deployed as a DaemonSet via the add-on) acts as the central collector:
┌────────────────────────────────────────────────────────┐│                  CloudWatch Agent                       ││  ┌──────────────────────────────────────────────────┐  ││  │            OTLP Receiver (Port 4316)              │  ││  │    Receives metrics and traces from ADOT SDKs    │  ││  └──────────────────────┬───────────────────────────┘  ││                         │                              ││  ┌──────────────────────▼───────────────────────────┐  ││  │              Processing Pipeline                  │  ││  │  - Batching                                       │  ││  │  - Attribute enrichment (k8s metadata)           │  ││  │  - Sampling decisions                            │  ││  └──────────────────────┬───────────────────────────┘  ││                         │                              ││  ┌──────────────────────▼───────────────────────────┐  ││  │              Exporters                            │  ││  │  - CloudWatch Metrics API                        │  ││  │  - X-Ray API                                     │  ││  │  - CloudWatch Logs API                           │  ││  └──────────────────────────────────────────────────┘  │└────────────────────────────────────────────────────────┘
5. Environment Variables That Drive Collection
For Node.js ESM (manual setup), these environment variables control behavior:
Variable	Purpose
OTEL_AWS_APPLICATION_SIGNALS_ENABLED=true	Enables Application Signals features
OTEL_TRACES_SAMPLER='xray'	Uses X-Ray sampling rules
OTEL_EXPORTER_OTLP_PROTOCOL='http/protobuf'	Sets the wire protocol
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT	Where to send traces
OTEL_AWS_APPLICATION_SIGNALS_EXPORTER_ENDPOINT	Where to send metrics
OTEL_RESOURCE_ATTRIBUTES	Kubernetes metadata (pod, node, deployment, namespace)
6. Kubernetes Metadata Enrichment
The system automatically enriches telemetry with Kubernetes context:
env:  - name: OTEL_RESOURCE_ATTRIBUTES_POD_NAME    valueFrom:      fieldRef:        fieldPath: metadata.name  - name: OTEL_RESOURCE_ATTRIBUTES_NODE_NAME    valueFrom:      fieldRef:        fieldPath: spec.nodeName  - name: OTEL_RESOURCE_ATTRIBUTES_DEPLOYMENT_NAME    valueFrom:      fieldRef:        fieldPath: metadata.labels['app']  - name: POD_NAMESPACE    valueFrom:      fieldRef:        fieldPath: metadata.namespace
This metadata allows Application Signals to:
Group metrics by deployment/service
Correlate pods to services
Show topology views
7. Data Flow Summary
┌─────────────┐    ┌─────────────┐    ┌─────────────┐│  Service A  │───▶│  Service B  │───▶│  Service C  ││  (Python)   │    │   (Java)    │    │  (Node.js)  │└──────┬──────┘    └──────┬──────┘    └──────┬──────┘       │                  │                  │       │ OTLP             │ OTLP             │ OTLP       ▼                  ▼                  ▼┌─────────────────────────────────────────────────────┐│              CloudWatch Agent (DaemonSet)            ││         amazon-cloudwatch namespace, port 4316       │└──────────────────────────┬──────────────────────────┘                           │          ┌────────────────┼────────────────┐          ▼                ▼                ▼   ┌────────────┐   ┌────────────┐   ┌────────────┐   │ CloudWatch │   │   X-Ray    │   │ CloudWatch │   │  Metrics   │   │  Traces    │   │    Logs    │   │            │   │            │   │            │   │ - Latency  │   │ - Spans    │   │ - App logs │   │ - Errors   │   │ - Maps     │   │ - Trace ID │   │ - Requests │   │ - Segments │   │ correlation│   └─────┬──────┘   └─────┬──────┘   └─────┬──────┘         │                │                │         └────────────────┼────────────────┘                          ▼              ┌────────────────────┐              │ Application Signals│              │     Console        │              │  (Unified View)    │              └────────────────────┘
Key Takeaways
Zero-code instrumentation - The ADOT SDK is injected via Kubernetes admission webhooks
OTLP standard - All telemetry uses OpenTelemetry Protocol for interoperability
Centralized collection - CloudWatch Agent aggregates all signals before sending to AWS
Automatic correlation - Trace IDs link metrics, traces, and logs together
Kubernetes-native - Pod/deployment metadata enriches all telemetry automatically

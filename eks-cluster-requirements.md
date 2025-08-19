# EKS Cluster Requirements Specification

## Overview
This document outlines the requirements for an Amazon EKS cluster designed to support a high-performance, scalable application with real-time capabilities, AI integration, and database management.

## Cluster Specifications

### Base EKS Configuration
- **Kubernetes Version**: 1.28 or later
- **Region**: us-west-2 (Oregon)
- **Node Groups**: 
  - Service-specific node groups for optimal resource allocation
  - Single AZ deployment for cost optimization
  - Auto-scaling enabled per service requirements

### Networking
- **VPC**: Custom VPC with public and private subnets
- **Security Groups**: Strict ingress/egress rules
- **Load Balancer**: Application Load Balancer (ALB) with SSL termination
- **Ingress Controller**: AWS Load Balancer Controller

### Node Groups Configuration
- **Database Node Group**: 
  - Instance types: c5.2xlarge, c5.4xlarge
  - Purpose: Neo4j and Redis clusters
  - Auto-scaling: min 2, no max limit
  - AZ: us-west-2a

- **WebSocket Service Node Group**:
  - Instance types: c5.large, c5.xlarge
  - Purpose: WebSocket service and HAProxy
  - Auto-scaling: min 2, no max limit
  - AZ: us-west-2a

- **Conversation Navigator Node Group**:
  - Instance types: c5.xlarge, c5.2xlarge
  - Purpose: Conversation Navigator with Gemini integration
  - Auto-scaling: min 1, no max limit
  - AZ: us-west-2a

- **Call Initiation Service Node Group**:
  - Instance types: c5.large, c5.xlarge
  - Purpose: Call Initiation Service with ElevenLabs integration
  - Auto-scaling: min 2, no max limit
  - AZ: us-west-2a

## Component Specifications

### 1. Neo4j Database Cluster

#### Configuration
- **Deployment Type**: StatefulSet with persistent volumes
- **Replicas**: 
  - 1 Core (write) instance
  - 2-10 Read Replicas (auto-scaled)
- **Storage**: 
  - Core: 100GB GP3 SSD
  - Read Replicas: 50GB GP3 SSD each
- **Resources**:
  - Core: 4 CPU, 16GB RAM
  - Read Replicas: 2 CPU, 8GB RAM each

#### Autoscaling Configuration
```yaml
HorizontalPodAutoscaler:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Object
      object:
        metric:
          name: query_latency_p95
        target:
          type: AverageValue
          averageValue: 100ms
  minReplicas: 2
  scaleUpBehavior:
    stabilizationWindowSeconds: 60
  scaleDownBehavior:
    stabilizationWindowSeconds: 300
```

#### Monitoring Requirements
- Query latency tracking (P95, P99)
- CPU and memory utilization
- Connection pool metrics
- Transaction throughput

### 2. Redis Database Cluster

#### Configuration
- **Deployment Type**: StatefulSet with Redis Cluster mode
- **Replicas**: 1-10 nodes (auto-scaled)
- **Storage**: 20GB GP3 SSD per node
- **Resources**: 2 CPU, 4GB RAM per node

#### Autoscaling Configuration
```yaml
HorizontalPodAutoscaler:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 85
    - type: Object
      object:
        metric:
          name: redis_commands_per_sec
        target:
          type: AverageValue
          averageValue: 10000
  minReplicas: 1
```

#### Monitoring Requirements
- Commands per second
- Memory usage and eviction rate
- Hit/miss ratio
- Network I/O

### 3. HAProxy Load Balancer?

#### Configuration
- **Deployment Type**: Deployment with 2-3 replicas
- **Resources**: 1 CPU, 2GB RAM
- **Purpose**: WebSocket connection routing and session affinity

#### Features
- Session affinity based on client IP
- Health checks for WebSocket endpoints
- SSL termination
- Real-time metrics endpoint

#### Configuration Example
```yaml
haproxy.cfg:
  global:
    maxconn 50000
    log stdout format raw local0 info
  
  defaults:
    mode tcp
    timeout connect 5s
    timeout client 50s
    timeout server 50s
  
  frontend websocket_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/server.pem
    default_backend websocket_backend
    stick-table type ip size 200k expire 30m
    stick on src
  
  backend websocket_backend
    balance roundrobin
    stick match src
    stick store-request src
    server websocket1 websocket-service-1:8080 check
    server websocket2 websocket-service-2:8080 check
    server websocket3 websocket-service-3:8080 check
```

### 4. WebSocket Service

#### Configuration
- **Deployment Type**: Deployment with auto-scaling
- **Replicas**: 2-20 (auto-scaled)
- **Resources**: 1 CPU, 2GB RAM per pod
- **Port**: 8080

#### Autoscaling Configuration
```yaml
HorizontalPodAutoscaler:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Object
      object:
        metric:
          name: websocket_connections
        target:
          type: AverageValue
          averageValue: 5
  minReplicas: 2
  scaleUpBehavior:
    stabilizationWindowSeconds: 30
  scaleDownBehavior:
    stabilizationWindowSeconds: 300
```

#### Features
- WebSocket connection management
- Session persistence
- Real-time message broadcasting
- Connection health monitoring

#### Dependencies
- Neo4j database access
- Redis for session storage
- HAProxy for load balancing

### 5. Conversation Navigator (Google Gemini Integration)

#### Configuration
- **Deployment Type**: Deployment with auto-scaling
- **Replicas**: 2-10 (auto-scaled)
- **Resources**: 2 CPU, 4GB RAM per pod
- **Port**: 8081

#### Autoscaling Configuration
```yaml
HorizontalPodAutoscaler:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Object
      object:
        metric:
          name: gemini_requests_per_minute
        target:
          type: AverageValue
          averageValue: 100
  minReplicas: 2
```

#### Features
- Google Gemini API integration
- Request queuing and rate limiting
- Response caching in Redis
- Neo4j data retrieval and storage

#### Dependencies
- Google Gemini API credentials
- Neo4j database access
- Redis for caching and queuing

### 6. Call Initiation Service (ElevenLabs Integration)

#### Configuration
- **Deployment Type**: Deployment with auto-scaling
- **Replicas**: 1-15 (auto-scaled)
- **Resources**: 1 CPU, 2GB RAM per pod
- **Port**: 8082

#### Autoscaling Configuration
```yaml
HorizontalPodAutoscaler:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
    - type: Object
      object:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 50
  minReplicas: 1
```

#### Features
- ElevenLabs API integration
- RESTful API endpoints
- Authentication and authorization
- Rate limiting
- Request/response logging

#### Dependencies
- ElevenLabs API credentials
- Neo4j database access
- Internet access for external API calls

## Monitoring and Observability

### Metrics Collection
- **Prometheus**: Cluster-wide metrics collection
- **Grafana**: Dashboard and alerting
- **Custom Metrics**: Application-specific metrics via Prometheus adapter

### Logging
- **Fluentd/Fluent Bit**: Log aggregation
- **Elasticsearch**: Log storage and search
- **Kibana**: Log visualization

### Alerting
- **AlertManager**: Alert routing and grouping
- **Slack/Email**: Notification channels

## Security Requirements

### Network Security
- All inter-service communication encrypted (mTLS)
- VPC with private subnets for databases
- Security groups with minimal required access
- WAF for API protection

### Secrets Management
- **AWS Secrets Manager**: External API credentials
- **Kubernetes Secrets**: Internal service credentials
- **Vault**: Optional for advanced secret management

### RBAC
- Service accounts for each component
- Minimal required permissions
- Regular permission audits

## Backup and Disaster Recovery

### Database Backups
- **Neo4j**: Automated daily backups to S3
- **Redis**: RDB snapshots every 6 hours
- **Cross-region replication**: Critical data

### Application Data
- Persistent volume snapshots
- Configuration backups
- Stateful application data replication

## Performance Requirements

### Latency Targets
- API response time: < 200ms (P95)
- WebSocket message delivery: < 50ms
- Database query latency: < 100ms (P95)
- Conversation Navigator response: < 5 seconds

### Throughput Targets
- API requests: 1000 RPS
- WebSocket connections: 10,000 concurrent
- Database transactions: 5000 TPS
- Conversation Navigator requests: 100 requests/minute

## Cost Optimization

### Resource Management
- Spot instances for non-critical workloads
- Reserved instances for databases
- Auto-scaling based on actual usage
- Resource limits and requests optimization

### Monitoring
- Cost tracking and alerting
- Resource utilization optimization
- Regular cost reviews

## Deployment Strategy

### CI/CD Pipeline
- **GitOps**: ArgoCD or Flux for deployment
- **Container Registry**: ECR with image scanning
- **Testing**: Automated testing before deployment
- **Rollback**: Quick rollback capabilities

### Environment Strategy
- **Development**: Single-node clusters
- **Staging**: Full-scale testing
- **Production**: Single AZ (us-west-2a) with service-specific node groups

## Maintenance and Updates

### Kubernetes Updates
- Regular security patches
- Version upgrade strategy
- Node rotation procedures

### Application Updates
- Blue-green deployments
- Canary releases
- Feature flags for gradual rollouts

## Compliance and Governance

### Data Protection
- GDPR compliance for user data
- Data encryption at rest and in transit
- Audit logging for all data access

### Access Control
- IAM roles and policies
- Kubernetes RBAC
- Regular access reviews

## Support and Documentation

### Documentation Requirements
- Architecture diagrams
- Runbooks for common issues
- API documentation
- Deployment guides

### Support Escalation
- 24/7 monitoring
- On-call rotation
- Escalation procedures
- Vendor support contacts

---

## Next Steps for DevOps Team

1. **Review and validate** all requirements
2. **Estimate resource costs** and timeline
3. **Create detailed implementation plan**
4. **Set up monitoring and alerting**
5. **Implement security measures**
6. **Create backup and DR procedures**
7. **Document operational procedures**
8. **Plan capacity and scaling strategies**

## Contact Information

For questions or clarifications regarding these requirements, please contact:
- **Technical Lead**: [Name]
- **Product Owner**: [Name]
- **Architecture Team**: [Email]

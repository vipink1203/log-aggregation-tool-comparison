# Deployment Guide: Log Aggregation with S3 Backend

This guide provides practical deployment recommendations for implementing log aggregation solutions with S3 backend storage using OpenObserve, Vector, or Elastic.

## Table of Contents

- [OpenObserve Deployment](#openobserve-deployment)
- [Vector Deployment](#vector-deployment)
- [Elasticsearch Deployment](#elasticsearch-deployment)
- [Hybrid Deployment Scenarios](#hybrid-deployment-scenarios)
- [S3 Configuration Best Practices](#s3-configuration-best-practices)
- [Monitoring and Maintenance](#monitoring-and-maintenance)

## OpenObserve Deployment

### Quick Start Deployment

For a simple deployment to evaluate OpenObserve:

```bash
# Using Docker
docker run -d \
  -p 5080:5080 \
  -e ZO_ROOT_USER_EMAIL=root@example.com \
  -e ZO_ROOT_USER_PASSWORD=Complexpass#123 \
  -e ZO_S3_BUCKET_NAME=my-logs-bucket \
  -e ZO_S3_REGION=us-east-1 \
  -e AWS_ACCESS_KEY_ID=<your-access-key> \
  -e AWS_SECRET_ACCESS_KEY=<your-secret-key> \
  --name openobserve \
  openobserve/openobserve:latest
```

### Production Deployment Recommendations

For production, a more robust deployment is recommended:

1. **Kubernetes Deployment**:
   - Use the Helm chart for OpenObserve
   - Set up proper resource limits and requests
   - Configure persistent volumes for local buffer storage

2. **High Availability Setup**:
   - Deploy multiple instances across availability zones
   - Use a load balancer for distribution
   - Configure proper health checks

3. **S3 Configuration**:
   - Create a dedicated S3 bucket with appropriate lifecycle policies
   - Set up proper IAM roles with least privilege access
   - Consider enabling versioning for data protection

4. **Security Considerations**:
   - Implement TLS for all communications
   - Use proper authentication methods
   - Set up network policies to restrict access

### Sample Kubernetes Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openobserve
  namespace: observability
spec:
  replicas: 3
  selector:
    matchLabels:
      app: openobserve
  template:
    metadata:
      labels:
        app: openobserve
    spec:
      containers:
      - name: openobserve
        image: openobserve/openobserve:latest
        ports:
        - containerPort: 5080
        env:
        - name: ZO_ROOT_USER_EMAIL
          valueFrom:
            secretKeyRef:
              name: openobserve-secrets
              key: admin-email
        - name: ZO_ROOT_USER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: openobserve-secrets
              key: admin-password
        - name: ZO_S3_BUCKET_NAME
          value: "logs-production"
        - name: ZO_S3_REGION
          value: "us-east-1"
        # Use IAM roles for service accounts in production
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: openobserve-data-pvc
```

## Vector Deployment

### Agent Deployment

For collecting logs from individual hosts:

```bash
# Install Vector
curl --proto '=https' --tlsv1.2 -sSf https://sh.vector.dev | bash

# Configure Vector as an agent
cat <<EOF > /etc/vector/vector.toml
[sources.file_logs]
type = "file"
include = ["/var/log/**/*.log"]
ignore_older = 86400

[transforms.parse_logs]
type = "remap"
inputs = ["file_logs"]
source = '''
. = parse_syslog!(string!(.message)) ?? .
'''

[sinks.s3_logs]
type = "aws_s3"
inputs = ["parse_logs"]
bucket = "my-logs-bucket"
region = "us-east-1"
compression = "gzip"
encoding.codec = "json"
batch.max_bytes = 10485760

# For partitioning by date and service
key_prefix = "logs/%Y/%m/%d/{{ service_name }}/"
EOF

# Start Vector
systemctl start vector
```

### Aggregator Deployment

For a centralized Vector aggregator:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vector-aggregator
  namespace: observability
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vector-aggregator
  template:
    metadata:
      labels:
        app: vector-aggregator
    spec:
      containers:
      - name: vector
        image: timberio/vector:latest-alpine
        ports:
        - containerPort: 9000
          name: vector-tcp
        volumeMounts:
        - name: config
          mountPath: /etc/vector
        - name: data
          mountPath: /var/lib/vector
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
      volumes:
      - name: config
        configMap:
          name: vector-aggregator-config
      - name: data
        persistentVolumeClaim:
          claimName: vector-data-pvc
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: vector-aggregator-config
  namespace: observability
data:
  vector.toml: |
    [sources.vector_tcp]
    type = "vector"
    address = "0.0.0.0:9000"
    
    [transforms.enhance_logs]
    type = "remap"
    inputs = ["vector_tcp"]
    source = '''
    .timestamp = now()
    .environment = get_env_var("ENVIRONMENT") ?? "production"
    .cluster = get_env_var("CLUSTER_NAME") ?? "unknown"
    '''
    
    [sinks.s3_archive]
    type = "aws_s3"
    inputs = ["enhance_logs"]
    bucket = "my-logs-bucket"
    region = "us-east-1"
    compression = "gzip"
    encoding.codec = "json"
    batch.max_bytes = 10485760
    key_prefix = "logs/%Y/%m/%d/${ENVIRONMENT}/${CLUSTER_NAME}/"
    
    # Optional: forward to OpenObserve
    [sinks.openobserve]
    type = "http"
    inputs = ["enhance_logs"]
    uri = "http://openobserve:5080/api/default/default/_bulk"
    encoding.codec = "json"
    auth.strategy = "basic"
    auth.user = "${OPENOBSERVE_USER}"
    auth.password = "${OPENOBSERVE_PASSWORD}"
    compression = "gzip"
```

### Serverless Deployment with Lambda

For processing logs directly from S3 events:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  VectorLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: vector-s3-processor
      Runtime: provided.al2
      Handler: bootstrap
      Role: !GetAtt VectorLambdaRole.Arn
      Code:
        S3Bucket: vector-lambda-deployment
        S3Key: vector-lambda.zip
      MemorySize: 1024
      Timeout: 300
      Environment:
        Variables:
          VECTOR_CONFIG: |
            [sources.s3]
            type = "aws_s3"
            
            [transforms.parse_logs]
            type = "remap"
            inputs = ["s3"]
            source = '''
            # Custom parsing logic
            '''
            
            [sinks.processed_s3]
            type = "aws_s3"
            inputs = ["parse_logs"]
            bucket = "processed-logs-bucket"
            region = "us-east-1"
            compression = "gzip"
            encoding.codec = "json"

  VectorLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: S3Access
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                Resource:
                  - arn:aws:s3:::my-logs-bucket/*
                  - arn:aws:s3:::processed-logs-bucket/*
```

## Elasticsearch Deployment

### AWS OpenSearch Service Deployment

The easiest way to deploy Elasticsearch with S3 integration is using AWS OpenSearch Service (formerly Elasticsearch Service):

1. **Create a Domain**:
   - Go to AWS OpenSearch Service console
   - Create a domain with appropriate instance types
   - Enable UltraWarm and Cold Storage for S3 integration

2. **Configure Storage Tiers**:
   - **Hot Storage**: EBS-backed for active data
   - **UltraWarm**: S3-backed for less frequently accessed data
   - **Cold Storage**: S3-backed for archival data

3. **Index Lifecycle Management**:
   - Create policies to move indices between tiers automatically
   - Example policy:
     ```json
     {
       "policy": {
         "phases": {
           "hot": {
             "min_age": "0ms",
             "actions": {
               "rollover": {
                 "max_age": "1d",
                 "max_size": "50gb"
               }
             }
           },
           "warm": {
             "min_age": "7d",
             "actions": {
               "migrate": {
                 "enabled": true
               }
             }
           },
           "cold": {
             "min_age": "30d",
             "actions": {
               "migrate": {
                 "enabled": true
               }
             }
           },
           "delete": {
             "min_age": "90d",
             "actions": {
               "delete": {}
             }
           }
         }
       }
     }
     ```

4. **Connect Vector to Elasticsearch**:
   ```toml
   [sinks.elasticsearch]
   type = "elasticsearch"
   inputs = ["parsed_logs"]
   endpoints = ["https://my-domain.us-east-1.es.amazonaws.com"]
   bulk.index = "logs-%Y-%m-%d"
   bulk.action = "index"
   compression = "gzip"
   auth.strategy = "aws"
   aws.region = "us-east-1"
   ```

## Hybrid Deployment Scenarios

### Vector + OpenObserve

This combined approach uses Vector's collection capabilities with OpenObserve's storage efficiency:

1. **Vector as the Collection Layer**:
   - Deploy Vector agents on all nodes
   - Configure to collect, parse, and enhance logs

2. **OpenObserve as the Storage Layer**:
   - Configure Vector to send to OpenObserve
   - Use OpenObserve for querying and visualization

3. **Direct S3 Archives**:
   - Optionally configure Vector to also send raw logs to S3
   - Implement lifecycle policies on the S3 bucket

4. **Sample Configuration**:
   ```toml
   # In Vector config
   [sinks.openobserve]
   type = "http"
   inputs = ["parsed_logs"]
   uri = "http://openobserve:5080/api/default/default/_bulk"
   encoding.codec = "json"
   
   [sinks.s3_archive]
   type = "aws_s3"
   inputs = ["parsed_logs"]
   bucket = "archive-logs-bucket"
   region = "us-east-1"
   ```

### Vector + Lambda + OpenObserve

For serverless processing of logs:

1. **Vector Agents**: Send logs to S3
2. **Lambda Function**: Triggered by S3 events, processes logs
3. **OpenObserve**: Receives processed logs for querying

This pattern allows for cost-effective processing that scales with volume.

## S3 Configuration Best Practices

### Bucket Configuration

1. **Lifecycle Policies**:
   ```json
   {
     "Rules": [
       {
         "Status": "Enabled",
         "Prefix": "logs/",
         "Transitions": [
           {
             "Days": 30,
             "StorageClass": "STANDARD_IA"
           },
           {
             "Days": 90,
             "StorageClass": "GLACIER"
           }
         ],
         "Expiration": {
           "Days": 365
         }
       }
     ]
   }
   ```

2. **Partitioning Strategy**:
   - Time-based partitioning is essential for log data
   - Consider a structure like: `logs/<year>/<month>/<day>/<service>/<instance>`
   - This improves query performance and enables efficient lifecycle management

3. **Compression**:
   - Always enable compression (GZIP, Zstandard, etc.)
   - Significantly reduces storage costs and improves performance

4. **File Format**:
   - Use structured formats like Parquet for analytical workloads
   - JSON Lines for better compatibility
   - Avoid plain text for large-scale deployments

### IAM Configuration

Create appropriate IAM policies:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-logs-bucket",
        "arn:aws:s3:::my-logs-bucket/*"
      ]
    }
  ]
}
```

For production, consider:
- Using IAM Roles instead of access keys
- Implementing least privilege access
- Setting up bucket policies for additional security

## Monitoring and Maintenance

### Monitoring the Log Pipeline

1. **Essential Metrics to Track**:
   - Ingest rate (logs/second)
   - S3 storage growth
   - Error rates
   - Query performance
   - API latency

2. **Health Checks**:
   - Set up probes for each component
   - Monitor end-to-end data flow
   - Implement alerting for pipeline failures

3. **Dashboards**:
   - Create operational dashboards for the logging infrastructure
   - Include cost metrics to track expenses

### Regular Maintenance Tasks

1. **Data Validation**:
   - Periodically verify data integrity
   - Ensure logs are properly parsed and stored

2. **Performance Tuning**:
   - Adjust batch sizes and buffer settings
   - Optimize compression and partitioning

3. **Cost Optimization**:
   - Review storage usage and implement further savings
   - Consider archiving older data to Glacier

4. **Security Updates**:
   - Keep all components updated
   - Rotate access credentials regularly
   - Review access patterns for suspicious activity

### Backup and Disaster Recovery

1. **Backup Strategy**:
   - For OpenObserve and Vector, configuration backups are essential
   - For Elasticsearch, regular snapshots to S3

2. **Disaster Recovery Plan**:
   - Document recovery procedures
   - Test restoration processes regularly
   - Consider multi-region replication for critical logs

## Conclusion

This deployment guide provides a starting point for implementing log aggregation with S3 backend storage. The most appropriate solution depends on your specific requirements:

- **OpenObserve**: Best for cost-efficiency and simplicity with S3 native storage
- **Vector**: Excellent for high-performance collection and routing
- **Elasticsearch**: Powerful for advanced search capabilities with tiered storage

For most organizations seeking to leverage S3 as their primary log storage, a combination of Vector for collection and OpenObserve for storage and querying provides an optimal balance of performance, cost-efficiency, and functionality.

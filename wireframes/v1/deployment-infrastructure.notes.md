# Deployment Infrastructure - Documentation Notes

## Overview

ElevenLabs-Bernie supports both local development and production deployment on AWS. The system uses Docker containers for consistency and requires GPU instances for AI model inference.

## Development Environment

### Prerequisites

- Node.js 18+
- Python 3.10+
- Docker (optional)
- NVIDIA GPU with CUDA (for model inference)
- 16GB+ RAM

### Quick Start

```bash
# 1. Clone repository
git clone https://github.com/user/ElevenLabs-Bernie.git
cd ElevenLabs-Bernie

# 2. Setup frontend
cd elevenlabs-clone-frontend
npm install
cp .env.example .env
# Edit .env with your values
npx prisma db push
npm run dev

# 3. Setup AI services (separate terminals)
cd StyleTTS2
pip install -r requirements.txt
uvicorn api:app --host 0.0.0.0 --port 8000

cd seed-vc
pip install -r requirements.txt
uvicorn api:app --host 0.0.0.0 --port 8001

cd Make-An-Audio
pip install -r requirements.txt
uvicorn api:app --host 0.0.0.0 --port 8002

# 4. Start Inngest
cd elevenlabs-clone-frontend
npx inngest-cli dev
```

### Local URLs

| Service | URL |
|---------|-----|
| Frontend | http://localhost:3000 |
| StyleTTS2 API | http://localhost:8000 |
| Seed-VC API | http://localhost:8001 |
| Make-An-Audio API | http://localhost:8002 |
| Inngest Dashboard | http://localhost:8288 |
| Prisma Studio | http://localhost:5555 |

### Environment Variables (Development)

```bash
# elevenlabs-clone-frontend/.env
NODE_ENV=development
DATABASE_URL="file:./dev.db"
AUTH_SECRET=your-dev-secret

# AWS (use real or localstack)
AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
AWS_REGION=us-east-1
S3_BUCKET_NAME=elevenlabs-clone

# AI Service Routes
STYLETTS2_API_ROUTE=http://localhost:8000
SEED_VC_API_ROUTE=http://localhost:8001
MAKE_AN_AUDIO_API_ROUTE=http://localhost:8002
BACKEND_API_KEY=dev-api-key
```

## Docker Development

### Docker Compose

```bash
docker-compose up
```

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  frontend:
    build: ./elevenlabs-clone-frontend
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/elevenlabs
      - STYLETTS2_API_ROUTE=http://styletts2-api:8000
      - SEED_VC_API_ROUTE=http://seedvc-api:8000
      - MAKE_AN_AUDIO_API_ROUTE=http://make-an-audio-api:8000
    depends_on:
      - db
      - styletts2-api
      - seedvc-api
      - make-an-audio-api

  styletts2-api:
    build:
      context: ./StyleTTS2
      dockerfile: Dockerfile.api
    ports:
      - "8000:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    environment:
      - API_KEY=${BACKEND_API_KEY}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

  seedvc-api:
    build:
      context: ./seed-vc
      dockerfile: Dockerfile.api
    ports:
      - "8001:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  make-an-audio-api:
    build:
      context: ./Make-An-Audio
      dockerfile: Dockerfile.api
    ports:
      - "8002:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=elevenlabs
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

## Production Environment

### AWS Architecture

```
Internet
    ↓
Route 53 (DNS)
    ↓
CloudFront (CDN) [optional]
    ↓
Application Load Balancer (HTTPS)
    ↓
    ├─→ Frontend EC2 (t3.medium)
    │      └─→ RDS PostgreSQL
    │      └─→ Inngest Cloud
    │
    └─→ GPU EC2 (g4dn.xlarge)
           ├─→ StyleTTS2 Container
           ├─→ Seed-VC Container
           └─→ Make-An-Audio Container
                  └─→ S3 Bucket
```

### EC2 Instance Requirements

#### Frontend Instance
- **Type**: t3.medium or larger
- **OS**: Amazon Linux 2 or Ubuntu
- **Storage**: 20GB EBS
- **Security Group**: Ports 3000, 22

#### GPU Instance
- **Type**: g4dn.xlarge (minimum) or g4dn.2xlarge
- **GPU**: NVIDIA T4
- **VRAM**: 16GB
- **OS**: Deep Learning AMI (Ubuntu)
- **Storage**: 100GB+ EBS (for models)
- **Security Group**: Ports 8000-8002, 22

### IAM Roles

#### EC2 Role (elevenlabs-clone-ec2)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::elevenlabs-clone",
        "arn:aws:s3:::elevenlabs-clone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
```

### S3 Bucket Configuration

**Bucket**: `elevenlabs-clone`

**Structure**:
```
elevenlabs-clone/
├── styletts2-output/
│   └── <uuid>.wav
├── seedvc-outputs/
│   └── <uuid>.wav
├── make-an-audio-outputs/
│   └── <uuid>.wav
└── uploads/
    └── <uuid>.wav
```

**Bucket Policy**:
- Private by default
- Access via presigned URLs only
- CORS enabled for browser uploads

**CORS Configuration**:
```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": ["https://yourdomain.com"],
    "ExposeHeaders": ["ETag"]
  }
]
```

### RDS Configuration

- **Engine**: PostgreSQL 15 or MySQL 8
- **Instance**: db.t3.micro (dev) or db.t3.medium (prod)
- **Storage**: 20GB gp3
- **Multi-AZ**: Yes for production
- **Backup**: 7 days retention

### Environment Variables (Production)

```bash
# Frontend
NODE_ENV=production
DATABASE_URL=postgresql://user:pass@rds-endpoint:5432/elevenlabs
AUTH_SECRET=production-secret-32-chars-min

# AWS
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1
S3_BUCKET_NAME=elevenlabs-clone

# AI Services
STYLETTS2_API_ROUTE=http://gpu-instance:8000
SEED_VC_API_ROUTE=http://gpu-instance:8001
MAKE_AN_AUDIO_API_ROUTE=http://gpu-instance:8002
BACKEND_API_KEY=production-api-key
```

## CI/CD Pipeline

### GitHub Actions Workflow

**.github/workflows/deploy.yml**:
```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build and push frontend
        run: |
          docker build -t $ECR_REGISTRY/frontend:${{ github.sha }} ./elevenlabs-clone-frontend
          docker push $ECR_REGISTRY/frontend:${{ github.sha }}

      - name: Build and push AI services
        run: |
          for service in StyleTTS2 seed-vc Make-An-Audio; do
            docker build -t $ECR_REGISTRY/$service:${{ github.sha }} ./$service -f ./$service/Dockerfile.api
            docker push $ECR_REGISTRY/$service:${{ github.sha }}
          done

      - name: Deploy to EC2
        run: |
          # SSH and pull new images
          # Restart containers
```

### Deployment Script

```bash
#!/bin/bash
# deploy.sh

# Pull latest images
docker-compose pull

# Stop and remove old containers
docker-compose down

# Start new containers
docker-compose up -d

# Run database migrations
docker-compose exec frontend npx prisma migrate deploy

# Health check
curl -f http://localhost:3000/api/health || exit 1
```

## Monitoring & Logging

### CloudWatch Metrics

- EC2 CPU, Memory, Network
- RDS connections, CPU, storage
- S3 request counts, data transfer
- Custom metrics for generation counts

### Logging

```bash
# Frontend logs
docker logs frontend -f

# AI service logs
docker logs styletts2-api -f

# Combined logs
docker-compose logs -f
```

### Health Checks

```bash
# Frontend
curl http://localhost:3000/api/health

# AI Services
curl http://localhost:8000/health
curl http://localhost:8001/health
curl http://localhost:8002/health
```

## Scaling Considerations

### Vertical Scaling

- **GPU Instance**: Upgrade to g4dn.2xlarge or p3.2xlarge
- **Database**: Increase RDS instance size
- **Frontend**: Upgrade EC2 instance type

### Horizontal Scaling

1. **Frontend**: Add EC2 instances behind ALB
2. **AI Services**: Multiple GPU instances with load balancer
3. **Database**: Read replicas for queries

### Cost Optimization

- **Spot Instances**: For AI services (with restart logic)
- **Reserved Instances**: For consistent workloads
- **S3 Lifecycle**: Archive old audio files
- **Right-sizing**: Monitor and adjust instance sizes

## Disaster Recovery

### Backup Strategy

1. **Database**: RDS automated backups (7 days)
2. **S3**: Versioning enabled
3. **Code**: GitHub repository
4. **Configs**: Stored in Secrets Manager

### Recovery Steps

1. Launch new EC2 instances from AMI
2. Restore RDS from snapshot
3. Pull Docker images from ECR
4. Update DNS/Load Balancer
5. Verify service health

## Security Hardening

### Network

- Private subnets for database and AI services
- NAT Gateway for outbound traffic
- Security groups with minimal rules
- VPC Flow Logs enabled

### Application

- HTTPS everywhere (ACM certificates)
- Secrets in AWS Secrets Manager
- Rotate API keys regularly
- Enable CloudTrail for audit

### Updates

- Regular OS patching
- Docker image updates
- Dependency vulnerability scanning

## Related Diagrams

- [Architecture Overview](./architecture-overview.mermaid.md) - System design
- [Entry Points](./entry-points.mermaid.md) - How to run
- [Data Flow](./data-flow.mermaid.md) - Request cycles

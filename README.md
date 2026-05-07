# AWS AIF-C01 Study Guide

A comprehensive, interactive study program for the AWS AI Fundamentals (AIF-C01) certification exam.

## Overview

This is a 4-day intensive study program covering:
- Machine Learning fundamentals (types, algorithms, architectures)
- AWS AI/ML services (SageMaker, Rekognition, Comprehend, and 15+ more)
- Model training, evaluation, and deployment
- MLOps best practices
- Exam preparation and practice questions

## Project Structure

```
aif-c01/
├── day1/              # Day 1: ML Fundamentals & Architecture
│   └── index.html     # Interactive study guide with quizzes
├── day2/              # Day 2: AWS AI/ML Services
├── day3/              # Day 3: Model Deployment & Inference
├── day4/              # Day 4: MLOps & Advanced Topics
├── infrastructure/    # AWS deployment configuration
│   ├── s3-setup.md
│   ├── cloudfront-setup.md
│   └── backend-setup.md
└── docs/              # Documentation and guides
```

## Getting Started

### Accessing the Study Guide

Open `day1/index.html` in a web browser to start the interactive study guide.

**Features:**
- Interactive click-to-reveal quizzes
- Progress tracking
- Exam preparation tips
- 50-item cheat sheet
- Self-assessment Q&As
- Built-in timer

### Local Development

```bash
# Clone the repository
git clone https://github.com/yourusername/aif-c01.git
cd aif-c01

# No build process needed - open day1/index.html directly in browser
# Or use a local server:
python -m http.server 8000
# Then visit http://localhost:8000/day1/
```

## Deployment

This guide is deployed on AWS using:
- **Frontend**: S3 + CloudFront for static hosting
- **Backend**: AWS Lambda + API Gateway (serverless)
- **Domain**: day1.aif.atla.in

See `infrastructure/` directory for setup documentation.

## Study Timeline

- **Day 1**: Machine learning fundamentals, neural networks, and AWS service overview
- **Day 2**: Deep dive into AWS AI/ML services (SageMaker, Rekognition, Comprehend, etc.)
- **Day 3**: Model deployment patterns, inference modes, and serverless options
- **Day 4**: MLOps, monitoring, and comprehensive exam preparation

## Navigation

Each day's guide includes:
- Sticky navigation for easy topic jumping
- Progress bar to track completion
- Key concepts and intuitive analogies
- Practical examples
- Self-test Q&As

## Resources

- [AWS AIF-C01 Exam Guide](https://aws.amazon.com/certification/certified-ai-practitioner/)
- Study materials organized for first-time learners
- No prior ML experience required

---

Last updated: 2026-05-07

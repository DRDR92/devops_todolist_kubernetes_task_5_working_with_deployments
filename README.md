# Django ToDo List - Kubernetes Deployment

A to-do list web application deployed to Kubernetes with auto-scaling capabilities.

## Project Overview
Django-based ToDo application with user authentication, REST API, and interactive UI.
This is a Kubernetes deployment project from Mate Academy demonstrating production-ready configurations.

## What I Implemented
- **Kubernetes Deployment** with RollingUpdate strategy
- **Horizontal Pod Autoscaler (HPA)** - scales between 2-5 pods based on CPU and Memory
- **Resource management** - properly configured requests and limits
- **Namespace isolation** - deployed in `mateapp` namespace
- **Complete deployment instructions** with reasoning for all choices

## Technologies
- Django 4+
- Kubernetes (Deployments, HPA, Services)
- Docker
- Python 3.8+

## Key Features
- Rolling updates with zero downtime
- Auto-scaling based on CPU (70%) and Memory (80%) metrics
- Resource-aware pod scheduling
- Proper liveness and readiness probes

## Quick Start (Local Development)
```bash
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

## Kubernetes Deployment
```bash
kubectl create namespace mateapp
kubectl apply -f deployment.yml
kubectl apply -f hpa.yml
kubectl port-forward svc/todolist 8000:8000 -n mateapp
```

## Project Structure
- `deployment.yml` - Kubernetes Deployment manifest with rolling updates
- `hpa.yml` - Horizontal Pod Autoscaler configuration
- `INSTRUCTION.md` - Detailed deployment guide with reasoning

## Key Learnings
- Deployment strategies and zero-downtime updates
- Resource management and pod scheduling
- Auto-scaling based on metrics
- Namespace isolation and RBAC concepts

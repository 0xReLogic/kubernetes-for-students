# Kubernetes for Students

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![GitHub issues](https://img.shields.io/github/issues/0xReLogic/kubernetes-for-students)](https://github.com/0xReLogic/kubernetes-for-students/issues)
[![GitHub stars](https://img.shields.io/github/stars/0xReLogic/kubernetes-for-students)](https://github.com/0xReLogic/kubernetes-for-students/stargazers)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Made for Students](https://img.shields.io/badge/Made%20for-Students-blue)](https://github.com/0xReLogic/kubernetes-for-students)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/0xReLogic/kubernetes-for-students)

> A hands-on learning journey through Kubernetes - because nobody learns distributed systems by reading 12 pages of YAML

## Who Is This For?

You're a computer science or software engineering student who:
- Heard about "Kubernetes" but still confused why we need an orchestra for containers
- Already learned Docker and thinking "okay I can run containers, now what?"
- Preparing for interviews or internships at tech companies using cloud-native stacks
- Want to understand DevOps but afraid of drowning in technical documentation
- Have a class assignment or capstone project requiring distributed system deployment

## Why Learn Kubernetes?

**Market Reality:**
- 94% of organizations use containers in production (CNCF Survey 2024)
- Kubernetes skills can increase salary by 25-40% for entry-level DevOps roles
- Top companies hiring: Google, Amazon, Microsoft, Netflix, Spotify, Uber
- Average Kubernetes engineer salary: $120k-180k (US market, 2025)

**Career Opportunities:**
- DevOps Engineer
- Cloud Engineer
- Site Reliability Engineer (SRE)
- Platform Engineer
- Full-stack Developer (modern stack)

Learning Kubernetes opens doors. Companies are literally throwing money at people who understand container orchestration.

## What Makes This Different?

**Not:**
- Official documentation used by production engineers (that already exists)
- Copy-paste YAML tutorials without understanding (internet has plenty of those)
- CKA certification speedrun (different goal)

**But:**
- Learning path that invites you to **experiment**, not just read
- Analogies that relate to everyday experiences
- Focus on "why" before "how"
- Hands-on challenges that make you truly understand, not just follow step-by-step

## How to Learn?

Each chapter has this structure:

1. **The Story** - Real-world analogy for the concept
2. **The Reality** - What's actually happening in Kubernetes
3. **The Experiment** - You run it yourself and see results
4. **The Breakdown** - Understand why it works (or fails)
5. **The Challenge** - Exercise to ensure you get it

## Learning Path

### Part 1: Foundation
Before you get lost in the cluster, you need to understand the map.

- **[Chapter 0: Setup Your Playground](./chapters/00-setup/)** - Install tools, setup minikube, don't get stuck at step 1
- **[Chapter 1: First Contact](./chapters/01-first-contact/)** - Meet the cluster without YAML overload
- **[Chapter 2: Pods - The Basic Unit](./chapters/02-pods/)** - Container vs Pod, and why you need to distinguish them

### Part 2: Core Concepts
These are components you'll use 90% of the time working with Kubernetes.

- **[Chapter 3: Deployments - The Copy Machine](./chapters/03-deployments/)** - Scaling, rolling updates, and self-healing that's not magic
- **[Chapter 4: Services - The Traffic Director](./chapters/04-services/)** - How pods communicate when their IP addresses keep changing
- **[Chapter 5: ConfigMaps & Secrets - Configuration Done Right](./chapters/05-config/)** - Externalize configuration the professional way
- **[Chapter 6: Persistent Storage - Data That Survives](./chapters/06-storage/)** - Volumes, PVCs, and storage that outlives pods
- **[Chapter 7: Ingress - The Front Door](./chapters/07-ingress/)** - Production-ready external access with SSL and routing
- **[Chapter 8: Debugging & Troubleshooting - When Things Go Wrong](./chapters/08-debugging/)** - Systematic approach to fix any Kubernetes issue
- **[Chapter 9: Best Practices - Production-Ready Kubernetes](./chapters/09-best-practices/)** - Security, performance, and reliability patterns
- **[Chapter 10: Capstone Projects - Build Real Applications](./chapters/10-capstone-projects/)** - Three complete projects from beginner to advanced

## Prerequisites

What you need:
- **Docker basics** - At least know how to build and run containers
- **Command line** - Comfortable in terminal, not afraid to type commands
- **Basic networking** - Understand IP addresses, ports, HTTP requests
- **Laptop** - 8GB RAM minimum (16GB recommended), dual-core CPU or better
- **Time** - 2-4 hours per chapter if you actually do the experiments

What's NOT required:
- Production DevOps experience
- Cloud certifications
- Dedicated server or cloud credits

## Quick Start

```bash
# Clone this repo
git clone https://github.com/0xReLogic/kubernetes-for-students.git
cd kubernetes-for-students

# Start from setup
cd chapters/00-setup
cat README.md
```

## Learning Philosophy

### 1. Experiment First, Theory Second
You'll break things first, then understand why it matters. More effective than reading theory for 2 hours then forgetting everything.

### 2. Real Failures, Real Learning
Error messages are the best teachers. Each chapter has "Common Mistakes" designed for you to encounter.

### 3. Progressive Complexity
Start simple, gradually add layers. No surprises like "oh turns out you need advanced networking for chapter 2".

### 4. Context Over Memorization
You don't need to memorize all kubectl commands. Just understand concepts, the rest can be googled or AI-assisted.

## Repository Structure

```
kubernetes-for-students/
├── chapters/           # Main learning content
│   ├── 00-setup/
│   ├── 01-first-contact/
│   ├── 02-pods/
│   └── ...
├── examples/           # YAML files and code samples
├── challenges/         # Hands-on exercises with solutions
├── resources/          # Cheatsheets, links, references
└── troubleshooting/    # Common errors and how to fix
```

## Contributing

This repo is open source and built for the student community. If you find:
- Typos or errors
- Unclear explanations
- Experiments that don't work on your setup
- Ideas for new chapters or challenges

Feel free to open an issue or submit a PR. Check [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## Credits & Resources

This repo is inspired by various sources:
- [Kubernetes Official Docs](https://kubernetes.io/docs/) - The ultimate reference
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) - Deep dive for serious learners
- Campus Expert program experiences from various universities
- Trial and error from hundreds of students who've learned K8s

## License

MIT License - use as much as you want, learn as much as you can, share with your friends.

## Start Now

Don't overthink it. Just start:

```bash
cd chapters/00-setup
```

The best time to learn Kubernetes was yesterday. The second best time is now.

---

**Maintained by**: Allen Elzayn 
**Last Updated**: October 2025  
**Status**: Active Development

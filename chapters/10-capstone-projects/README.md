# Chapter 10: Capstone Projects

## The Grand Finale

Remember when you first opened Chapter 0? You were just learning what a Pod was. Now look at you - you understand Deployments, Services, ConfigMaps, Secrets, Persistent Storage, Ingress, debugging strategies, and production best practices. You're ready to build something real.

This chapter contains three capstone projects that combine everything you've learned. Each project increases in complexity, and all of them simulate real-world scenarios you'll encounter in your career.

## Before You Start

### Prerequisites
- Completed Chapters 0-9
- A working minikube cluster
- kubectl installed and configured
- Basic understanding of at least one programming language
- Patience and coffee (optional, but recommended)

### What Makes a Good Capstone Project?

Think of these projects like cooking a full meal versus following a recipe for one dish. You've learned all the techniques (chapters 1-9), now it's time to combine them creatively.

A good capstone project:
- Uses multiple Kubernetes concepts together
- Solves a realistic problem
- Has moving parts that need to communicate
- Requires troubleshooting when things go wrong
- Makes you think about production concerns

---

## Project 1: Personal Blog Platform (Beginner)

### The Story

Your friend is a writer who wants a personal blog. They don't want to pay for WordPress hosting, and they trust you to build something simple but reliable. You decide to use Kubernetes because it's easier to scale if the blog becomes popular.

### What You'll Build

A simple blog platform with:
- A frontend web server (NGINX serving static HTML)
- A backend API (simple REST API for blog posts)
- A database (MySQL for storing posts)
- Persistent storage (so blog posts don't disappear)

### Architecture Diagram

```
┌────────────────────────────────────────────────┐
│              Browser (Users)                   │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│              Ingress Controller                │
│         (Routes traffic to services)           │
└────────┬────────────────────┬───────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────┐  ┌─────────────────┐
│  Frontend       │  │  API Service    │
│  (NGINX)        │  │  (Node.js)      │
│  Port: 80       │  │  Port: 3000     │
└─────────────────┘  └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  MySQL Database │
                     │  Port: 3306     │
                     │  + PVC Storage  │
                     └─────────────────┘
```

### Learning Objectives

By the end of this project, you'll be able to:
- Deploy a multi-tier application
- Connect frontend to backend to database
- Use ConfigMaps for configuration
- Use Secrets for database passwords
- Set up persistent storage
- Configure Ingress for external access
- Debug connection issues between services

### Step 1: Database Layer

Create the MySQL database with persistent storage.

**mysql-pvc.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: blog-mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**mysql-secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: blog-mysql-secret
type: Opaque
stringData:
  root-password: "RootPass123"
  database: "blog_db"
  user: "blog_user"
  password: "BlogPass123"
```

**mysql-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-mysql
  labels:
    app: blog
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blog
      tier: database
  template:
    metadata:
      labels:
        app: blog
        tier: database
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: blog-mysql-secret
              key: root-password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: blog-mysql-secret
              key: database
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: blog-mysql-secret
              key: user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: blog-mysql-secret
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -u
            - root
            - -p$MYSQL_ROOT_PASSWORD
            - -e
            - "SELECT 1"
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: blog-mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: blog-mysql
  labels:
    app: blog
    tier: database
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: blog
    tier: database
```

### Step 2: Backend API

Create a simple REST API that connects to MySQL.

**api-configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: blog-api-config
data:
  DB_HOST: "blog-mysql"
  DB_PORT: "3306"
  API_PORT: "3000"
```

**api-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-api
  labels:
    app: blog
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blog
      tier: backend
  template:
    metadata:
      labels:
        app: blog
        tier: backend
    spec:
      containers:
      - name: api
        image: node:18-alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            cat > /app/server.js << 'EOF'
            const express = require('express');
            const mysql = require('mysql2/promise');
            const app = express();
            
            app.use(express.json());
            
            const dbConfig = {
              host: process.env.DB_HOST,
              port: process.env.DB_PORT,
              user: process.env.DB_USER,
              password: process.env.DB_PASSWORD,
              database: process.env.DB_NAME
            };
            
            let pool;
            
            async function initDB() {
              pool = mysql.createPool(dbConfig);
              const connection = await pool.getConnection();
              
              await connection.query(`
                CREATE TABLE IF NOT EXISTS posts (
                  id INT AUTO_INCREMENT PRIMARY KEY,
                  title VARCHAR(255) NOT NULL,
                  content TEXT NOT NULL,
                  author VARCHAR(100) NOT NULL,
                  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
              `);
              
              connection.release();
              console.log('Database initialized');
            }
            
            app.get('/health', (req, res) => {
              res.json({ status: 'healthy', timestamp: new Date().toISOString() });
            });
            
            app.get('/posts', async (req, res) => {
              try {
                const [rows] = await pool.query('SELECT * FROM posts ORDER BY created_at DESC');
                res.json(rows);
              } catch (error) {
                console.error('Error fetching posts:', error);
                res.status(500).json({ error: 'Failed to fetch posts' });
              }
            });
            
            app.post('/posts', async (req, res) => {
              try {
                const { title, content, author } = req.body;
                const [result] = await pool.query(
                  'INSERT INTO posts (title, content, author) VALUES (?, ?, ?)',
                  [title, content, author]
                );
                res.status(201).json({ id: result.insertId, title, content, author });
              } catch (error) {
                console.error('Error creating post:', error);
                res.status(500).json({ error: 'Failed to create post' });
              }
            });
            
            const port = process.env.API_PORT || 3000;
            
            initDB().then(() => {
              app.listen(port, '0.0.0.0', () => {
                console.log(`Blog API running on port ${port}`);
              });
            }).catch(err => {
              console.error('Failed to initialize database:', err);
              process.exit(1);
            });
            EOF
            
            cd /app
            npm init -y
            npm install express mysql2
            node server.js
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: blog-api-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: blog-api-config
              key: DB_PORT
        - name: API_PORT
          valueFrom:
            configMapKeyRef:
              name: blog-api-config
              key: API_PORT
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: blog-mysql-secret
              key: user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: blog-mysql-secret
              key: password
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: blog-mysql-secret
              key: database
        ports:
        - containerPort: 3000
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: blog-api
  labels:
    app: blog
    tier: backend
spec:
  type: ClusterIP
  ports:
  - port: 3000
    targetPort: 3000
  selector:
    app: blog
    tier: backend
```

### Step 3: Frontend

Create a simple HTML frontend.

**frontend-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-frontend
  labels:
    app: blog
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blog
      tier: frontend
  template:
    metadata:
      labels:
        app: blog
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            cat > /usr/share/nginx/html/index.html << 'EOF'
            <!DOCTYPE html>
            <html lang="en">
            <head>
              <meta charset="UTF-8">
              <meta name="viewport" content="width=device-width, initial-scale=1.0">
              <title>My Kubernetes Blog</title>
              <style>
                * { margin: 0; padding: 0; box-sizing: border-box; }
                body {
                  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
                  line-height: 1.6;
                  color: #333;
                  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                  min-height: 100vh;
                  padding: 20px;
                }
                .container {
                  max-width: 800px;
                  margin: 0 auto;
                  background: white;
                  border-radius: 12px;
                  box-shadow: 0 10px 40px rgba(0,0,0,0.2);
                  overflow: hidden;
                }
                .header {
                  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                  color: white;
                  padding: 40px;
                  text-align: center;
                }
                .header h1 { font-size: 2.5em; margin-bottom: 10px; }
                .header p { opacity: 0.9; }
                .content { padding: 40px; }
                .post-form {
                  background: #f8f9fa;
                  padding: 30px;
                  border-radius: 8px;
                  margin-bottom: 40px;
                }
                .form-group { margin-bottom: 20px; }
                .form-group label {
                  display: block;
                  margin-bottom: 8px;
                  font-weight: 600;
                  color: #555;
                }
                .form-group input, .form-group textarea {
                  width: 100%;
                  padding: 12px;
                  border: 2px solid #e0e0e0;
                  border-radius: 6px;
                  font-size: 16px;
                  transition: border-color 0.3s;
                }
                .form-group input:focus, .form-group textarea:focus {
                  outline: none;
                  border-color: #667eea;
                }
                .form-group textarea { min-height: 150px; resize: vertical; }
                .btn {
                  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                  color: white;
                  border: none;
                  padding: 14px 30px;
                  border-radius: 6px;
                  font-size: 16px;
                  font-weight: 600;
                  cursor: pointer;
                  transition: transform 0.2s, box-shadow 0.2s;
                }
                .btn:hover {
                  transform: translateY(-2px);
                  box-shadow: 0 5px 20px rgba(102, 126, 234, 0.4);
                }
                .posts { margin-top: 40px; }
                .post {
                  background: white;
                  border: 2px solid #e0e0e0;
                  border-radius: 8px;
                  padding: 25px;
                  margin-bottom: 20px;
                  transition: transform 0.2s, box-shadow 0.2s;
                }
                .post:hover {
                  transform: translateY(-2px);
                  box-shadow: 0 5px 20px rgba(0,0,0,0.1);
                }
                .post-title {
                  font-size: 1.5em;
                  color: #667eea;
                  margin-bottom: 10px;
                }
                .post-meta {
                  color: #888;
                  font-size: 0.9em;
                  margin-bottom: 15px;
                }
                .post-content {
                  color: #555;
                  line-height: 1.8;
                }
                .loading {
                  text-align: center;
                  padding: 40px;
                  color: #888;
                  font-size: 1.2em;
                }
                .error {
                  background: #fee;
                  border: 2px solid #fcc;
                  color: #c33;
                  padding: 15px;
                  border-radius: 6px;
                  margin-bottom: 20px;
                }
                .success {
                  background: #efe;
                  border: 2px solid #cfc;
                  color: #3c3;
                  padding: 15px;
                  border-radius: 6px;
                  margin-bottom: 20px;
                }
              </style>
            </head>
            <body>
              <div class="container">
                <div class="header">
                  <h1>My Kubernetes Blog</h1>
                  <p>Powered by Kubernetes, Built with Love</p>
                </div>
                
                <div class="content">
                  <div class="post-form">
                    <h2 style="margin-bottom: 20px; color: #667eea;">Write a New Post</h2>
                    <div id="message"></div>
                    <form id="postForm">
                      <div class="form-group">
                        <label for="title">Title</label>
                        <input type="text" id="title" placeholder="Enter post title..." required>
                      </div>
                      <div class="form-group">
                        <label for="author">Author</label>
                        <input type="text" id="author" placeholder="Your name..." required>
                      </div>
                      <div class="form-group">
                        <label for="content">Content</label>
                        <textarea id="content" placeholder="Write your post content..." required></textarea>
                      </div>
                      <button type="submit" class="btn">Publish Post</button>
                    </form>
                  </div>
                  
                  <div class="posts">
                    <h2 style="margin-bottom: 20px; color: #667eea;">Recent Posts</h2>
                    <div id="postsContainer">
                      <div class="loading">Loading posts...</div>
                    </div>
                  </div>
                </div>
              </div>
              
              <script>
                const API_URL = '/api';
                
                async function loadPosts() {
                  try {
                    const response = await fetch(`${API_URL}/posts`);
                    const posts = await response.json();
                    
                    const container = document.getElementById('postsContainer');
                    
                    if (posts.length === 0) {
                      container.innerHTML = '<div class="loading">No posts yet. Be the first to write one!</div>';
                      return;
                    }
                    
                    container.innerHTML = posts.map(post => `
                      <div class="post">
                        <div class="post-title">${escapeHtml(post.title)}</div>
                        <div class="post-meta">
                          By ${escapeHtml(post.author)} • ${new Date(post.created_at).toLocaleString()}
                        </div>
                        <div class="post-content">${escapeHtml(post.content)}</div>
                      </div>
                    `).join('');
                  } catch (error) {
                    console.error('Error loading posts:', error);
                    document.getElementById('postsContainer').innerHTML = 
                      '<div class="error">Failed to load posts. Please check your API connection.</div>';
                  }
                }
                
                document.getElementById('postForm').addEventListener('submit', async (e) => {
                  e.preventDefault();
                  
                  const title = document.getElementById('title').value;
                  const author = document.getElementById('author').value;
                  const content = document.getElementById('content').value;
                  const messageDiv = document.getElementById('message');
                  
                  try {
                    const response = await fetch(`${API_URL}/posts`, {
                      method: 'POST',
                      headers: { 'Content-Type': 'application/json' },
                      body: JSON.stringify({ title, author, content })
                    });
                    
                    if (response.ok) {
                      messageDiv.innerHTML = '<div class="success">Post published successfully!</div>';
                      document.getElementById('postForm').reset();
                      setTimeout(() => { messageDiv.innerHTML = ''; }, 3000);
                      loadPosts();
                    } else {
                      throw new Error('Failed to publish post');
                    }
                  } catch (error) {
                    console.error('Error creating post:', error);
                    messageDiv.innerHTML = '<div class="error">Failed to publish post. Please try again.</div>';
                  }
                });
                
                function escapeHtml(text) {
                  const div = document.createElement('div');
                  div.textContent = text;
                  return div.innerHTML;
                }
                
                loadPosts();
                setInterval(loadPosts, 30000);
              </script>
            </body>
            </html>
            EOF
            
            cat > /etc/nginx/conf.d/default.conf << 'EOF'
            server {
              listen 80;
              server_name _;
              
              location / {
                root /usr/share/nginx/html;
                index index.html;
                try_files $uri $uri/ /index.html;
              }
              
              location /api/ {
                proxy_pass http://blog-api:3000/;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              }
            }
            EOF
            
            nginx -g 'daemon off;'
        ports:
        - containerPort: 80
          name: http
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: blog-frontend
  labels:
    app: blog
    tier: frontend
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: blog
    tier: frontend
```

### Step 4: Ingress

**blog-ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: blog.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-frontend
            port:
              number: 80
```

### Deploy the Blog Platform

```bash
# Create namespace
kubectl create namespace blog-platform

# Deploy in order
kubectl apply -f mysql-pvc.yaml -n blog-platform
kubectl apply -f mysql-secret.yaml -n blog-platform
kubectl apply -f mysql-deployment.yaml -n blog-platform

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l tier=database -n blog-platform --timeout=120s

# Deploy API
kubectl apply -f api-configmap.yaml -n blog-platform
kubectl apply -f api-deployment.yaml -n blog-platform

# Wait for API to be ready
kubectl wait --for=condition=ready pod -l tier=backend -n blog-platform --timeout=120s

# Deploy Frontend
kubectl apply -f frontend-deployment.yaml -n blog-platform

# Deploy Ingress (if you have Ingress controller installed)
kubectl apply -f blog-ingress.yaml -n blog-platform

# Get the service URL
kubectl get svc blog-frontend -n blog-platform
```

### Testing Your Blog

```bash
# Port-forward to access locally
kubectl port-forward svc/blog-frontend 8080:80 -n blog-platform

# Open browser to http://localhost:8080
```

### Challenge Tasks

1. **Add a Delete Feature**: Implement a DELETE endpoint in the API and a delete button in the frontend
2. **Add Categories**: Extend the database schema to support post categories
3. **Implement Search**: Add a search feature to find posts by title or content
4. **Add Pagination**: Display posts in pages (10 per page)
5. **Add Monitoring**: Use the debugging techniques from Chapter 8 to monitor the blog

### Troubleshooting Guide

**Problem**: API can't connect to MySQL
```bash
# Check MySQL is running
kubectl get pods -n blog-platform -l tier=database

# Check API logs
kubectl logs -n blog-platform -l tier=backend

# Test MySQL connection from API pod
kubectl exec -it -n blog-platform deployment/blog-api -- /bin/sh
nc -zv blog-mysql 3306
```

**Problem**: Frontend can't reach API
```bash
# Check API service exists
kubectl get svc blog-api -n blog-platform

# Check NGINX proxy configuration
kubectl exec -it -n blog-platform deployment/blog-frontend -- cat /etc/nginx/conf.d/default.conf

# Test API from frontend pod
kubectl exec -it -n blog-platform deployment/blog-frontend -- wget -O- http://blog-api:3000/health
```

---

## Project 2: E-Commerce Microservices (Intermediate)

### The Story

Your startup just got funding to build an e-commerce platform. The CTO wants a microservices architecture that can scale independently. You're responsible for building the MVP (Minimum Viable Product) that handles products, shopping carts, and orders.

### What You'll Build

A microservices-based e-commerce platform with:
- **Product Service**: Manages product catalog
- **Cart Service**: Handles shopping carts (uses Redis)
- **Order Service**: Processes orders
- **Frontend**: Web UI that connects to all services
- **Database**: PostgreSQL for products and orders
- **Cache**: Redis for cart sessions
- **Service Mesh**: Basic service communication patterns

### Architecture Diagram

```
                        ┌─────────────────┐
                        │   Browser       │
                        └────────┬────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │   Ingress Controller   │
                    └────────┬───────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │   Frontend     │
                    │   (React SPA)  │
                    └────┬─────┬─────┘
                         │     │
        ┌────────────────┘     └────────────────┐
        │                                       │
        ▼                                       ▼
┌───────────────────┐                  ┌────────────────────┐
│  Product Service  │                  │   Cart Service     │
│  Port: 8080       │                  │   Port: 8081       │
└────────┬──────────┘                  └─────────┬──────────┘
         │                                       │
         ▼                                       ▼
┌────────────────┐                      ┌────────────────┐
│  PostgreSQL    │                      │     Redis      │
│  (Products DB) │                      │  (Cart Cache)  │
└────────────────┘                      └────────────────┘
         │
         └──────────────┐
                        ▼
                ┌────────────────┐
                │ Order Service  │
                │ Port: 8082     │
                └────────┬───────┘
                         │
                         ▼
                ┌────────────────┐
                │  PostgreSQL    │
                │  (Orders DB)   │
                └────────────────┘
```

### Learning Objectives

- Build and deploy multiple independent microservices
- Implement service-to-service communication
- Use different databases for different services
- Implement caching with Redis
- Handle distributed transactions
- Monitor multiple services
- Debug cross-service issues

### Architecture Overview

**Product Service** (Python/Flask)
- GET /products - List all products
- GET /products/:id - Get product details
- POST /products - Create product
- PUT /products/:id - Update product

**Cart Service** (Node.js/Express)
- GET /cart/:userId - Get user cart
- POST /cart/:userId/items - Add item to cart
- DELETE /cart/:userId/items/:productId - Remove item
- POST /cart/:userId/checkout - Checkout cart

**Order Service** (Go)
- GET /orders/:userId - Get user orders
- POST /orders - Create order
- GET /orders/:id - Get order details

### Implementation Files

Due to the complexity, I'll provide the key manifests. You'll need to implement the actual service code.

**namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    name: ecommerce
```

**postgres-configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: ecommerce
data:
  POSTGRES_DB: "ecommerce"
  POSTGRES_USER: "ecommerce_user"
```

**postgres-secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: ecommerce
type: Opaque
stringData:
  POSTGRES_PASSWORD: "SecurePass123"
```

**postgres-statefulset.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: ecommerce
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - configMapRef:
            name: postgres-config
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: ecommerce
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
```

**redis-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
          name: redis
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: ecommerce
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
```

**product-service-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
  labels:
    app: product-service
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
        version: v1
    spec:
      containers:
      - name: product-service
        image: python:3.11-slim
        command: ["/bin/sh", "-c"]
        args:
          - |
            pip install flask psycopg2-binary requests
            cat > /app/product_service.py << 'EOF'
            from flask import Flask, jsonify, request
            import psycopg2
            import os
            import time
            
            app = Flask(__name__)
            
            def get_db():
                conn = psycopg2.connect(
                    host=os.environ.get('DB_HOST', 'postgres'),
                    database=os.environ.get('DB_NAME', 'ecommerce'),
                    user=os.environ.get('DB_USER', 'ecommerce_user'),
                    password=os.environ.get('DB_PASSWORD', 'password')
                )
                return conn
            
            def init_db():
                time.sleep(5)
                conn = get_db()
                cur = conn.cursor()
                cur.execute('''
                    CREATE TABLE IF NOT EXISTS products (
                        id SERIAL PRIMARY KEY,
                        name VARCHAR(255) NOT NULL,
                        description TEXT,
                        price DECIMAL(10,2) NOT NULL,
                        stock INT NOT NULL DEFAULT 0,
                        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                    )
                ''')
                
                cur.execute('SELECT COUNT(*) FROM products')
                if cur.fetchone()[0] == 0:
                    sample_products = [
                        ('Kubernetes T-Shirt', 'Comfortable cotton t-shirt with K8s logo', 29.99, 100),
                        ('Docker Mug', 'Keep your coffee warm while containerizing', 14.99, 50),
                        ('DevOps Hoodie', 'Stay warm during those late-night deployments', 49.99, 75),
                        ('Cloud Stickers Pack', 'Decorate your laptop with cloud providers', 9.99, 200),
                        ('Microservices Book', 'Learn to build scalable applications', 39.99, 30)
                    ]
                    cur.executemany(
                        'INSERT INTO products (name, description, price, stock) VALUES (%s, %s, %s, %s)',
                        sample_products
                    )
                
                conn.commit()
                cur.close()
                conn.close()
            
            @app.route('/health', methods=['GET'])
            def health():
                return jsonify({'status': 'healthy', 'service': 'product-service'})
            
            @app.route('/products', methods=['GET'])
            def get_products():
                conn = get_db()
                cur = conn.cursor()
                cur.execute('SELECT id, name, description, price, stock FROM products')
                products = []
                for row in cur.fetchall():
                    products.append({
                        'id': row[0],
                        'name': row[1],
                        'description': row[2],
                        'price': float(row[3]),
                        'stock': row[4]
                    })
                cur.close()
                conn.close()
                return jsonify(products)
            
            @app.route('/products/<int:product_id>', methods=['GET'])
            def get_product(product_id):
                conn = get_db()
                cur = conn.cursor()
                cur.execute('SELECT id, name, description, price, stock FROM products WHERE id = %s', (product_id,))
                row = cur.fetchone()
                cur.close()
                conn.close()
                
                if row:
                    return jsonify({
                        'id': row[0],
                        'name': row[1],
                        'description': row[2],
                        'price': float(row[3]),
                        'stock': row[4]
                    })
                return jsonify({'error': 'Product not found'}), 404
            
            if __name__ == '__main__':
                init_db()
                app.run(host='0.0.0.0', port=8080, debug=True)
            EOF
            python /app/product_service.py
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: DB_HOST
          value: "postgres"
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_DB
        - name: DB_USER
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 5
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: ecommerce
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
    name: http
  selector:
    app: product-service
```

### Deploy E-Commerce Platform

```bash
# Create namespace
kubectl apply -f namespace.yaml

# Deploy databases
kubectl apply -f postgres-configmap.yaml
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl apply -f redis-deployment.yaml

# Wait for databases
kubectl wait --for=condition=ready pod -l app=postgres -n ecommerce --timeout=120s
kubectl wait --for=condition=ready pod -l app=redis -n ecommerce --timeout=120s

# Deploy services
kubectl apply -f product-service-deployment.yaml
# (Add cart-service and order-service deployments)

# Check all services
kubectl get all -n ecommerce
```

### Challenge Tasks

1. **Implement Cart Service**: Build the cart service using Node.js and Redis
2. **Implement Order Service**: Build the order service using Go
3. **Add API Gateway**: Use NGINX Ingress as an API Gateway
4. **Implement Rate Limiting**: Limit API requests per user
5. **Add Health Checks**: Implement liveness and readiness probes for all services
6. **Monitor Services**: Set up basic monitoring with resource metrics
7. **Implement Retries**: Add retry logic for service-to-service communication

---

## Project 3: Full-Stack Production App (Advanced)

### The Story

Your startup has grown. You need a production-ready application with everything: monitoring, logging, auto-scaling, CI/CD, security, and high availability. The board wants zero downtime deployments, and you need to prove your Kubernetes skills can handle it.

### What You'll Build

A complete production application with:
- Multi-tier microservices architecture
- Horizontal Pod Autoscaling
- Rolling updates with zero downtime
- Centralized logging (EFK stack - Elasticsearch, Fluentd, Kibana)
- Metrics monitoring (Prometheus + Grafana)
- Ingress with TLS
- Network policies for security
- Resource quotas and limits
- Pod disruption budgets
- Readiness and liveness probes
- ConfigMaps and Secrets management
- Persistent storage with backups

### Architecture Overview

This is a simplified social media platform called "CloudChat":
- **Auth Service**: User authentication (JWT tokens)
- **User Service**: User profiles
- **Post Service**: Create/read posts
- **Feed Service**: Aggregates posts into feeds
- **Notification Service**: Real-time notifications
- **Frontend**: React SPA
- **API Gateway**: NGINX Ingress with rate limiting
- **Databases**: PostgreSQL (users, posts), Redis (sessions, cache)
- **Message Queue**: RabbitMQ (async notifications)

### Pre-requisites

```bash
# Install Prometheus Operator
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

# Install NGINX Ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Install metrics-server (if not already installed)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Project Structure

```
cloudchat/
├── infrastructure/
│   ├── namespaces/
│   │   └── namespace.yaml
│   ├── databases/
│   │   ├── postgres-statefulset.yaml
│   │   └── redis-deployment.yaml
│   ├── messaging/
│   │   └── rabbitmq-deployment.yaml
│   └── monitoring/
│       ├── servicemonitor.yaml
│       └── grafana-dashboard.yaml
├── services/
│   ├── auth-service/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── pdb.yaml
│   ├── user-service/
│   ├── post-service/
│   ├── feed-service/
│   └── notification-service/
├── frontend/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── hpa.yaml
├── ingress/
│   ├── ingress.yaml
│   └── certificate.yaml
├── security/
│   └── network-policies.yaml
└── config/
    ├── configmaps/
    └── secrets/
```

### Key Implementation: Auth Service with HPA

**auth-service-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: cloudchat
  labels:
    app: auth-service
    tier: backend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
        tier: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - auth-service
              topologyKey: kubernetes.io/hostname
      containers:
      - name: auth-service
        image: auth-service:1.0.0  # Your actual image
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: PORT
          value: "8080"
        - name: DB_HOST
          value: "postgres"
        - name: REDIS_HOST
          value: "redis"
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: auth-secrets
              key: jwt-secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # Graceful shutdown
      terminationGracePeriodSeconds: 30
```

**auth-service-hpa.yaml**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-service-hpa
  namespace: cloudchat
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-service
  minReplicas: 3
  maxReplicas: 10
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 15
      selectPolicy: Max
```

**auth-service-pdb.yaml**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: auth-service-pdb
  namespace: cloudchat
spec:
  minAvailable: 2  # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: auth-service
```

**network-policy.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: auth-service-policy
  namespace: cloudchat
spec:
  podSelector:
    matchLabels:
      app: auth-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53  # DNS
```

**ingress-with-tls.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cloudchat-ingress
  namespace: cloudchat
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - cloudchat.example.com
    secretName: cloudchat-tls
  rules:
  - host: cloudchat.example.com
    http:
      paths:
      - path: /api/auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 8080
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /api/posts
        pathType: Prefix
        backend:
          service:
            name: post-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

### Production Deployment Checklist

Before deploying to production:

- [ ] All services have resource requests and limits
- [ ] Liveness and readiness probes configured
- [ ] HPA configured for scalable services
- [ ] PDB configured to prevent disruption
- [ ] Network policies restrict unnecessary communication
- [ ] Secrets are not hardcoded (use Kubernetes Secrets)
- [ ] TLS certificates configured for Ingress
- [ ] Monitoring and alerting set up
- [ ] Logging centralized
- [ ] Backup strategy for databases
- [ ] Disaster recovery plan documented
- [ ] Load testing performed
- [ ] Security scanning completed
- [ ] Documentation updated

### Deployment Strategy

```bash
# Create namespace
kubectl create namespace cloudchat

# Deploy infrastructure
kubectl apply -f infrastructure/namespaces/
kubectl apply -f infrastructure/databases/
kubectl apply -f infrastructure/messaging/

# Wait for databases
kubectl wait --for=condition=ready pod -l app=postgres -n cloudchat --timeout=120s
kubectl wait --for=condition=ready pod -l app=redis -n cloudchat --timeout=120s

# Deploy config and secrets
kubectl apply -f config/configmaps/
kubectl apply -f config/secrets/

# Deploy services
for service in auth-service user-service post-service feed-service notification-service; do
  kubectl apply -f services/$service/
  kubectl wait --for=condition=available deployment/$service -n cloudchat --timeout=120s
done

# Deploy frontend
kubectl apply -f frontend/

# Apply security policies
kubectl apply -f security/

# Deploy ingress
kubectl apply -f ingress/

# Verify deployment
kubectl get all -n cloudchat
```

### Testing Zero-Downtime Deployment

```bash
# Start continuous requests
while true; do
  curl -s https://cloudchat.example.com/api/health
  sleep 1
done

# In another terminal, update the auth-service
kubectl set image deployment/auth-service auth-service=auth-service:1.0.1 -n cloudchat

# Watch the rollout
kubectl rollout status deployment/auth-service -n cloudchat

# The continuous requests should show zero failures
```

### Monitoring with Prometheus & Grafana

```bash
# Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Default credentials:
# Username: admin
# Password: prom-operator

# Import dashboard for your services
# Use dashboard ID 6417 for Kubernetes cluster monitoring
```

### Challenge Tasks

1. **Implement CI/CD**: Set up GitHub Actions or GitLab CI to automatically deploy on push
2. **Add Canary Deployments**: Deploy new versions to 10% of users first
3. **Implement Blue-Green Deployment**: Have two identical environments
4. **Add Rate Limiting**: Implement per-user rate limiting
5. **Set up Alerts**: Configure Prometheus alerts for high CPU, memory, errors
6. **Implement Distributed Tracing**: Add Jaeger or Zipkin for request tracing
7. **Database Backups**: Implement automated PostgreSQL backups
8. **Multi-Region**: Deploy to multiple regions for high availability
9. **Cost Optimization**: Implement pod autoscaling based on custom metrics
10. **Security Hardening**: Run security scans, implement RBAC, use Pod Security Standards

---

## General Tips for All Projects

### Debugging Strategy

1. **Check Pod Status**
```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

2. **Check Logs**
```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous  # Previous container
```

3. **Test Service Connectivity**
```bash
kubectl run -it --rm debug --image=busybox --restart=Never -n <namespace> -- sh
wget -O- http://service-name:port
```

4. **Check Resource Usage**
```bash
kubectl top pods -n <namespace>
kubectl top nodes
```

### Best Practices Applied

From Chapter 9, remember to:
- Set resource requests and limits
- Use health probes
- Implement graceful shutdown
- Use ConfigMaps for configuration
- Use Secrets for sensitive data
- Label everything consistently
- Use namespaces for isolation
- Implement network policies
- Monitor everything
- Document everything

---

## Submission Guidelines

### What to Submit

1. **Code Repository**
   - All YAML manifests
   - Application code (if applicable)
   - README with setup instructions
   - Architecture diagrams

2. **Documentation**
   - Setup instructions
   - Architecture decisions
   - Challenges faced and how you solved them
   - Screenshots of running application

3. **Demo Video** (Optional but recommended)
   - 5-10 minute walkthrough
   - Show deployment process
   - Demonstrate key features
   - Show monitoring/debugging

### Evaluation Criteria

- **Functionality** (40%): Does it work as specified?
- **Kubernetes Best Practices** (25%): Resource limits, health probes, labels, etc.
- **Code Quality** (15%): Clean, well-organized manifests
- **Documentation** (10%): Clear setup instructions and explanations
- **Innovation** (10%): Extra features or creative solutions

---

## Next Steps

### After Completing Projects

1. **Get Certified**
   - Certified Kubernetes Application Developer (CKAD)
   - Certified Kubernetes Administrator (CKA)

2. **Contribute to Open Source**
   - Kubernetes itself
   - Kubernetes operators
   - Helm charts

3. **Keep Learning**
   - Service meshes (Istio, Linkerd)
   - GitOps (ArgoCD, Flux)
   - Security (Falco, OPA)
   - Serverless on K8s (Knative, OpenFaaS)

4. **Build Your Portfolio**
   - Deploy your projects to a real cluster (AWS EKS, GKE, AKS)
   - Write blog posts about what you learned
   - Share on GitHub and LinkedIn

---

## Conclusion

You started this journey not knowing what a Pod was. Now you can build production-ready microservices on Kubernetes. That's incredible progress.

These capstone projects aren't just exercises - they're real patterns you'll use in your career. Every company using Kubernetes deals with these exact challenges: connecting services, managing databases, scaling applications, ensuring zero downtime.

The skills you've learned in these 10 chapters will serve you for years. Kubernetes is the foundation of modern cloud-native applications, and you now have the knowledge to build on that foundation.

## Remember

- Start simple, then add complexity
- Always test in development first
- Monitor everything
- Document your decisions
- Learn from failures
- Ask for help when stuck
- Share your knowledge with others

Good luck with your projects! You've got this.

---

**Course Complete! 🎉**

*From zero to Kubernetes hero in 10 chapters.*

Share your projects:
- GitHub: Tag with `#kubernetes-for-students`

We can't wait to see what you build!

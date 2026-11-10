# Society Management System – Docker Deployment & DevOps Documentation

## 1. Project Overview

This document records the Docker containerization, deployment, networking, environment configuration, database integration, and troubleshooting performed for the Society Management System.

### Technology Stack

| Component | Technology |
|---|---|
| Frontend | React, Vite |
| Backend | Node.js, Express |
| Database | MongoDB Atlas |
| Frontend Web Server | Nginx Alpine |
| Backend Container | Node.js 22 Alpine |
| Containerization | Docker |
| Orchestration | Docker Compose |
| Container Networking | Docker Bridge Network |
| Cloud Platform | AWS EC2 |

---

## 2. Docker Multi-Stage Build – Frontend

The React frontend was containerized using a multi-stage Docker build.

### Why Multi-Stage Builds?

- Node.js is used only for building the React application.
- Nginx serves the generated static files.
- Node.js and npm are excluded from the final production image.
- Reduces unnecessary dependencies in the production container.

### Frontend Dockerfile

```dockerfile
FROM node:22-alpine AS build

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build

FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Important Concepts

- `FROM node:22-alpine AS build`: Creates the build stage.
- `WORKDIR /app`: Sets the working directory.
- `npm ci`: Installs dependencies from the lock file.
- `npm run build`: Generates the production build in `dist`.
- `FROM nginx:alpine`: Starts a separate production stage.
- `COPY --from=build`: Copies only the generated frontend files.
- `EXPOSE 80`: Documents the Nginx container port.

---

## 3. Nginx Configuration

### Problem: React Router Refresh Returns 404

Initially, navigating to the login page worked, but refreshing `/login` returned a 404 error.

### Root Cause

React applications using client-side routing are Single Page Applications (SPA).

When a user refreshes `/login`, Nginx attempts to locate a physical file or directory named `login`.

Since the route is managed by React Router, the file does not exist.

### Solution: SPA Fallback

Created `client/nginx.conf`:

```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend:3000;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /socket.io/ {
        proxy_pass http://backend:3000;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

### Important Configuration

```nginx
try_files $uri $uri/ /index.html;
```

This ensures that frontend routes are handled by React Router.

The `/api/` location forwards backend requests to the Node.js container.

The `/socket.io/` location supports WebSocket communication.

---

## 4. Frontend Environment Variables

Vite environment variables are embedded into the JavaScript bundle during the build process.

### Environment Variable

For the Docker Compose deployment with Nginx reverse proxy:

```env
VITE_API_URL=/api/v1
```

### Frontend API Request

Example:

```javascript
axios.post(
    `${import.meta.env.VITE_API_URL}/auth/login`,
    data
);
```

The resulting browser request becomes:

```text
http://YOUR_SERVER_IP:8083/api/v1/auth/login
```

### Passing Build Arguments

For deployments where the API URL is configured at build time:

```dockerfile
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
```

Build command:

```bash
docker build \
  --build-arg VITE_API_URL=/api/v1 \
  -t society-frontend:1.0 .
```

**Important:** `VITE_` variables are publicly accessible in browser code. Never store passwords, API secrets, or private credentials in them.

---

## 5. Backend Dockerization

The backend uses Node.js 22 Alpine.

### Backend Dockerfile

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

### Why `npm ci --omit=dev`?

- Installs dependencies from `package-lock.json`.
- Provides reproducible dependency installation.
- Excludes development dependencies.
- Reduces unnecessary packages in production.

### Important Docker Commands

Build:

```bash
docker build -t backend-society:1.0 .
```

Run:

```bash
docker run -d \
  --name society-backend \
  -p 3000:3000 \
  --env-file .env \
  backend-society:1.0
```

---

## 6. Environment Variable Management

Backend configuration is maintained separately in `server/.env`.

### Example Environment Variables

```env
MONGODB_URI=mongodb+srv://USERNAME:PASSWORD@YOUR_CLUSTER.mongodb.net/society_db

JWT_SECRET_STRING=your_jwt_secret
JWT_EXPIRES_IN=7d

SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

CLIENT_URL=http://YOUR_SERVER_IP:8083

AWS_REGION=ap-south-1
AWS_S3_BUCKET_NAME=your_bucket
```

### Best Practices

- Never hardcode credentials in Dockerfiles.
- Never push `.env` files to GitHub.
- Use Docker `--env-file` or Compose `env_file` to inject backend configuration.
- Use AWS Secrets Manager or another secrets manager for production credentials.
- Rotate credentials if accidentally exposed.

### Difference Between ARG, ENV and Runtime Variables

| Mechanism | Purpose |
|---|---|
| `ARG` | Build-time configuration |
| `ENV` | Environment variable inside an image/container |
| `docker run -e` | Runtime environment injection |
| `--env-file` | Load runtime variables from a file |
| Compose `env_file` | Inject variables into Compose services |

---

## 7. MongoDB Atlas Integration

The backend connects to MongoDB Atlas using Mongoose.

### Initial Problems

Two errors occurred during database integration:

**Error 1: Invalid Scheme**

```text
MongoParseError: Invalid scheme
```

Cause: The connection string was incorrectly formatted, containing an extra `MONGODB_URI=` prefix.

Solution: Corrected the environment variable assignment.

**Error 2: Authentication Failed**

```text
MongoServerError: bad auth : authentication failed
```

Cause: MongoDB Atlas rejected the database credentials.

Solution: Verified and updated the database user credentials.

### Successful Connection

The backend logs eventually confirmed:

```text
MONGODB IS UP
server is running
```

The GDPR cleanup worker also executed successfully.

### Security Note

Never print the full MongoDB connection URI in application logs.

---

## 8. Docker Networking

Initially, the frontend and backend communicated through the EC2 public IP and published host ports.

Example:

```text
Frontend: http://YOUR_SERVER_IP:8083
Backend:  http://YOUR_SERVER_IP:3000
```

This worked without a custom Docker network because both ports were published on the Docker host.

Later, the deployment architecture was improved using a custom Docker bridge network.

### Docker Network

```bash
docker network create society-network
```

For manual container networking, both containers must be connected to the same network.

Docker provides internal DNS resolution using container or service names.

Example:

```text
http://backend:3000
```

The frontend's React code does not directly use this hostname because React executes in the user's browser. Nginx uses it for server-side proxying.

---

## 9. Docker Compose Deployment

Docker Compose was introduced to manage frontend and backend containers together.

### docker-compose.yml

```yaml
services:

  frontend:
    build:
      context: ./client
    container_name: society-frontend
    ports:
      - "8083:80"
    depends_on:
      - backend
    restart: unless-stopped
    networks:
      - society-network

  backend:
    build:
      context: ./server
    container_name: society-backend
    env_file:
      - ./server/.env
    expose:
      - "3000"
    restart: unless-stopped
    networks:
      - society-network

networks:
  society-network:
    driver: bridge
```

### Important Docker Compose Concepts

| Configuration | Explanation |
|---|---|
| `build` | Builds the image from the specified directory |
| `ports` | Publishes container ports to the host |
| `expose` | Documents internal service ports |
| `depends_on` | Defines startup ordering |
| `env_file` | Loads backend environment variables |
| `networks` | Connects services to a common network |
| `restart` | Defines container restart policy |

Docker Compose automatically creates the declared network when the application is started.

The default network name is generally based on the Compose project name and declared network name.

### Deployment Commands

Build images:

```bash
docker compose build
```

Start services:

```bash
docker compose up -d
```

Build and start together:

```bash
docker compose up -d --build
```

Check running containers:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

Stop services:

```bash
docker compose down
```

Restart backend:

```bash
docker compose restart backend
```

---

## 10. Troubleshooting Performed

### Issue 1: Container Exited with Code 127

Error:

```text
/bin/sh: [npm,start]: not found
```

Cause: Incorrect Docker CMD formatting.

Incorrect:

```dockerfile
CMD ["[npm,start]"]
```

Correct backend command:

```dockerfile
CMD ["npm", "start"]
```

Correct frontend command:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

### Issue 2: React Login Page Returned 404 on Refresh

Cause: Missing Nginx SPA fallback.

Solution:

```nginx
try_files $uri $uri/ /index.html;
```

### Issue 3: CORS Errors

Cause: Frontend and backend were running on different origins.

Frontend:

```text
http://YOUR_SERVER_IP:8083
```

Backend:

```text
http://YOUR_SERVER_IP:3000
```

Solution: Configure backend CORS correctly and ensure dotenv configuration is loaded before reading environment variables.

### Issue 4: Login API Returned 404

Incorrect endpoint:

```text
/api/auth/login
```

Correct backend endpoint:

```text
/api/v1/auth/login
```

Solution:

```env
VITE_API_URL=/api/v1
```

The Nginx reverse proxy forwards the request to the backend.

### Issue 5: MongoDB Authentication Failure

Cause: Incorrect database credentials.

Solution: Verify the Atlas database user, password, and connection URI.

### Issue 6: Seed Scripts Missing

The backend package contained npm seed commands referencing:

```text
seeders/seedRole.js
seeders/seedUser.js
seeders/seedFlat.js
```

However, those files were not present in the Docker image.

An existing `seed.js` file was found, which creates an admin account.

**Important:** Inspect seed scripts before execution. Avoid deleting existing users or inserting duplicate records.

---

## 11. Useful Docker Commands

| Task | Command |
|---|---|
| List running containers | `docker ps` |
| List all containers | `docker ps -a` |
| View container logs | `docker logs CONTAINER_NAME` |
| Follow logs | `docker logs -f CONTAINER_NAME` |
| Inspect container | `docker inspect CONTAINER_NAME` |
| List images | `docker images` |
| List networks | `docker network ls` |
| Inspect network | `docker network inspect NETWORK_NAME` |
| Enter container | `docker exec -it CONTAINER_NAME sh` |
| Remove container | `docker rm -f CONTAINER_NAME` |
| Test Nginx configuration | `nginx -t` |

---

## 12. Security Improvements

The following improvements should be maintained for production deployment:

1. Keep backend port 3000 private when using the Nginx reverse proxy.
2. Allow only necessary inbound ports in the AWS Security Group.
3. Use HTTPS with a valid SSL certificate.
4. Keep database credentials outside the Docker image.
5. Add `.env` to `.gitignore`.
6. Rotate credentials if exposed.
7. Avoid printing sensitive environment variables in logs.
8. Use MongoDB Atlas IP access restrictions.
9. Run containers with least privilege where practical.
10. Implement health checks and database connection readiness.

---

## 13. Final Deployment Architecture

```text
                  USER BROWSER
                       |
                       |
               AWS EC2 PUBLIC IP
                       |
                       |
                 PORT 8083
                       |
                       |
              FRONTEND CONTAINER
                  NGINX ALPINE
                       |
                /api/v1 requests
                       |
                       |
               DOCKER NETWORK
                society-network
                       |
                       |
               BACKEND CONTAINER
                NODE.JS ALPINE
                  PORT 3000
                       |
                       |
                  MONGOOSE
                       |
                       |
                 MONGODB ATLAS
```

### Final Summary

The Society Management System was containerized using Docker multi-stage builds, configured with Nginx for frontend routing and API reverse proxying, integrated with MongoDB Atlas, and prepared for multi-container deployment using Docker Compose and a custom bridge network.

This setup separates frontend and backend responsibilities, reduces public backend exposure, simplifies environment management, and provides a foundation for future CI/CD automation and production deployment.

---

**Documentation status:** Dockerfiles, Nginx configuration, environment management, MongoDB integration, networking, and Docker Compose deployment documented.

**Next planned improvements:**
- HTTPS and custom domain configuration.
- Docker health checks.
- CI/CD pipeline using Jenkins.
- Container image scanning using Trivy.
- Centralized logging and monitoring.
- Automated deployment to AWS EC2.

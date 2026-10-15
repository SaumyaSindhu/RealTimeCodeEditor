# Real-Time Collaborative Code Editor

A browser-based collaborative code editor built with React, Monaco Editor, Yjs, and Socket.IO. Multiple users can join the same editor session, see the connected users, and edit the document in real time.

## Live application

**[Open the deployed editor](http://docker-aws-ALB-1246814122.eu-north-1.elb.amazonaws.com)**

The application is exposed through an AWS Application Load Balancer in the `eu-north-1` region. Open the link, enter a username, and share the same URL with another user to collaborate.

## Features

- Real-time collaborative editing
- Monaco Editor with JavaScript syntax highlighting
- User presence list
- Yjs document synchronization
- Socket.IO transport for browser-to-server communication
- Express static-file serving
- Dockerized production build
- Health-check endpoint at `/health`

## Technology stack

### Frontend

- React
- Vite
- Monaco Editor
- Yjs
- `y-monaco`
- `y-socket.io`
- Tailwind CSS

### Backend

- Node.js 20
- Express
- Socket.IO
- `y-socket.io`

### Infrastructure

- Docker with a multi-stage image build
- AWS Application Load Balancer (ALB)
- AWS compute host running the Docker container
- AWS region: `eu-north-1` (Stockholm)

## How it works

1. The frontend asks the visitor for a username.
2. The browser creates a Yjs document and binds it to Monaco Editor.
3. The browser connects to the backend using `y-socket.io` over Socket.IO.
4. Yjs distributes document updates between connected clients.
5. Socket.IO awareness events update the list of users currently connected to the session.
6. In production, the backend serves the Vite-generated frontend from its `public` directory and also handles real-time connections.

## Project structure

```text
project/
├── backend/
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── package.json
│   ├── src/
│   │   └── app/
│   │       └── App.jsx
│   └── vite.config.js
├── dockerfile
└── README.md
```

## Run locally without Docker

### Prerequisites

- Node.js 20 or newer
- npm

### Start the backend

```bash
cd backend
npm install
npm start
```

The backend listens on port `3000`.

### Start the frontend

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Vite will print the local development URL in the terminal. During local frontend development, the backend and frontend are separate processes. For the complete production-style setup, use Docker.

## Run with Docker

The [`dockerfile`](./dockerfile) uses a multi-stage build:

1. The `frontend-builder` stage uses Node 20 Alpine to install frontend dependencies and run `npm run build`.
2. The final Node 20 Alpine stage installs backend dependencies.
3. The generated frontend files are copied into `backend/public`.
4. The backend starts with `node server.js`.

Build the image from the project root:

```bash
docker build -f dockerfile -t realtime-editor .
```

Run the container:

```bash
docker run --name realtime-editor -p 3000:3000 realtime-editor
```

Open [http://localhost:3000](http://localhost:3000).

Check the backend health endpoint:

```bash
curl http://localhost:3000/health
```

Expected response:

```json
{
  "message": "ok",
  "success": true
}
```

Stop and remove the container:

```bash
docker stop realtime-editor
docker rm realtime-editor
```

## AWS deployment overview

This application is suitable for deployment on an AWS compute service that can run Docker, such as Amazon EC2, ECS, or App Runner. The supplied live hostname is an AWS Application Load Balancer hostname, so the deployment can be understood as:

```text
Browser
   │
   ▼
AWS Application Load Balancer
   │
   ▼
Docker container
   ├── Express static files
   └── Socket.IO / Yjs real-time server
```

### Important AWS configuration

- The container must listen on port `3000`.
- The target group should forward traffic to container port `3000`.
- The ALB health check should use `GET /health`.
- The security group should allow inbound traffic to the ALB listener and allow the ALB to reach the container target.
- Socket.IO traffic must be allowed through the load balancer. The ALB supports WebSocket upgrades, which are required for efficient real-time collaboration.
- For production HTTPS, attach an AWS Certificate Manager certificate to an HTTPS listener and redirect HTTP traffic to HTTPS.
- If the service is scaled to multiple container instances, configure Socket.IO/Yjs state sharing or session affinity as appropriate. A single in-memory real-time server does not automatically synchronize state between independent container instances.

### Typical EC2 deployment flow

The following is a high-level example for an EC2 host with Docker installed:

```bash
docker build -f dockerfile -t realtime-editor .
docker stop realtime-editor || true
docker rm realtime-editor || true
docker run -d \
  --name realtime-editor \
  --restart unless-stopped \
  -p 3000:3000 \
  realtime-editor
```

For a production workflow, publish the image to Amazon Elastic Container Registry (ECR), pull the tagged image on the deployment host, and replace the running container during deployment. ECR provides private image storage and integrates with AWS IAM for controlled access.

## AWS services involved

- **Application Load Balancer:** Provides a stable public DNS endpoint, routes HTTP traffic to healthy targets, and supports WebSocket connections.
- **Amazon EC2, ECS, or App Runner:** Runs the Dockerized Node.js application.
- **Amazon ECR:** Stores versioned Docker images for repeatable deployments.
- **AWS CloudWatch:** Collects container, host, and load-balancer logs and metrics.
- **AWS Certificate Manager:** Provides TLS certificates for HTTPS.
- **IAM:** Controls access between deployment tooling, ECR, and runtime resources.

## Useful commands

```bash
# Build the frontend only
cd frontend
npm run build

# Lint the frontend
cd frontend
npm run lint

# Run the backend in development mode
cd backend
npm run dev
```

## Notes

- The editor currently uses one shared Yjs document named `monaco`.
- A username is stored in the URL query string as `?username=...`.
- The backend currently listens on a fixed port of `3000`.
- The application does not include authentication or persistent document storage. Do not use it for sensitive code or data without adding access control and a persistence strategy.

## License

The backend package currently declares the ISC license. Review and update licensing terms before distributing the project.

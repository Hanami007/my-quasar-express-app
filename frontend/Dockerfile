#Stage 1: Build Quasar SPA
FROM node:20-alpine AS builder
WORKDIR /app

# Copy package*.json and .npmrc first to leverage cache layer
COPY package*.json ./

# Copy the entire project files (except those in .dockerignore)
# This must be here to allow Quasar scripts to run properly during npm ci
COPY . .

# Install dependencies and run prepare script (which previously caused errors)
# We use npm install instead of npm ci for flexibility; if errors persist, consider switching back to npm ci
RUN npm install --legacy-peer-deps

# Build Quasar SPA (output is in /app/dist/spa)
RUN npx quasar build -m spa
# Install Leaflet and @types/leaflet
RUN npm install leaflet @types/leaflet

# Stage 2: Production (Serve with Nginx)
FROM nginx:1.27-alpine

# Copy build artifacts from stage 1
COPY --from=builder /app/dist/spa /usr/share/nginx/html

# Create Nginx Config to support Quasar History Mode (fix 404 on page refresh)
RUN echo 'server { \
    listen 80; \
    root /usr/share/nginx/html; \
    index index.html; \
    location / { \
        try_files $uri $uri/ /index.html; \
    } \
}' > /etc/nginx/conf.d/default.conf

# Expose port and run Nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
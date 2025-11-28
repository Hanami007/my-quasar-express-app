# Stage 1: Build Quasar SPA
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files และ install dependencies
COPY package*.json ./
RUN npm ci --legacy-peer-deps

# Copy source code ทั้งหมด
COPY . .

# Build Quasar (SPA)
RUN npm run build


# Stage 2: Production (Nginx serve static files)
FROM nginx:1.27-alpine

# Copy build จาก Stage 1
COPY --from=builder /app/dist/spa /usr/share/nginx/html

# Expose port 80
EXPOSE 80

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]


# DevOps Utility Hub - Production Dockerfile
# Multi-stage build with Nginx


# Stage 1: Build & Prepare
FROM node:18-alpine AS builder

LABEL maintainer="Hitesh <hitesh@devops.com>"
LABEL description="DevOps Utility Hub - 12+ Tools"
LABEL version="1.0.0"

WORKDIR /app

# Copy all source files
COPY index.html .
COPY css/ ./css/
COPY js/ ./js/
COPY images/ ./images/ 
COPY README.md .

# Verify files exist
RUN ls -la && echo " Build files ready!"

# Stage 2: Production with Nginx

FROM nginx:1.25-alpine AS production

# Remove default nginx content
RUN rm -rf /usr/share/nginx/html/*

# Copy custom nginx config
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf

# Copy built files from builder stage
COPY --from=builder /app/index.html /usr/share/nginx/html/
COPY --from=builder /app/css/ /usr/share/nginx/html/css/
COPY --from=builder /app/js/ /usr/share/nginx/html/js/
COPY --from=builder /app/README.md /usr/share/nginx/html/

# Copy images if they exist (won't fail if empty)
COPY images/ /usr/share/nginx/html/images/

# Create non-root user for security
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup && \
    chown -R appuser:appgroup /usr/share/nginx/html && \
    chown -R appuser:appgroup /var/cache/nginx && \
    chown -R appuser:appgroup /var/log/nginx && \
    chown -R appuser:appgroup /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && \
    chown -R appuser:appgroup /var/run/nginx.pid

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Expose port
EXPOSE 8080

# Switch to non-root user
USER appuser

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
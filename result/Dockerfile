FROM node:24.11.1-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy application code
COPY --chown=node:node . .

# Switch to non-root user
USER node

EXPOSE 4000
CMD ["node", "server.js"]
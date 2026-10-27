FROM node:20-alpine AS builder

WORKDIR /app

# copiar los package
COPY package*.json ./

# instalar 
RUN npm ci --omit=dev

FROM node:20-alpine

WORKDIR /app

# dependencias del builder
COPY --from=builder /app/node_modules ./node_modules

# copiar el codigo
COPY . .

# usar el usuario node de alpine
USER node

EXPOSE 3000

# healthcheck
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://127.0.0.1:3000/healthz || exit 1

CMD ["node", "server.js"]

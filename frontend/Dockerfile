FROM node:22-alpine AS build
WORKDIR /ui
ENV NODE_ENV=production
COPY package.json package-lock.json* ./
RUN npm ci --include=dev
COPY . .
RUN npm run build \
    && rm -rf node_modules

FROM nginx:1.27-alpine
RUN rm -f /etc/nginx/conf.d/default.conf
COPY --from=build /ui/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
RUN chown -R nginx:nginx /usr/share/nginx/html \
    && chown -R nginx:nginx /var/cache/nginx \
    && mkdir -p /run \
    && chown -R nginx:nginx /run
USER nginx
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -q -O /dev/null http://127.0.0.1/ || exit 1

FROM nginxinc/nginx-unprivileged:alpine

# copio la configuracion del lab
COPY default.conf /etc/nginx/conf.d/default.conf

# copio el html index del lab
COPY index.html /usr/share/nginx/html/index.html

# expongo el 8080 como pide
EXPOSE 8080

# usuario no root
USER 101

# healthcheck
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://127.0.0.1:8080/ || exit 1

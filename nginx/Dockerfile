FROM nginx:1.9

# A default configuration for reverse proxy.
COPY ./default.conf /etc/nginx/conf.d/default.conf

# A response to NOT FOUND page.
COPY ./backend-not-found.html /var/www/html/backend-not-found.html

#  Reverse Proxy and SSL configurations
COPY ./includes/ /etc/nginx/includes/

# Reverse Proxy SSL certificates
COPY ./ssl/ /etc/ssl/certs/nginx/

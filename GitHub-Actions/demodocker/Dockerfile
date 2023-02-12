FROM nginx:1.23-alpine

COPY nginx.conf /etc/nginx/nginx.conf
COPY index.html /var/www/html/index.html
COPY style.css /var/www/html/style.css

CMD ["nginx", "-g", "daemon off;"]
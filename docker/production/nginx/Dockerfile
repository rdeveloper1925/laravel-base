# Static files come from the php-fpm image (includes `npm run build` output).
# Compose passes it as additional context `app` (see docker-compose.yml).
FROM nginx:alpine

COPY ./docker/production/nginx/nginx.conf /etc/nginx/nginx.conf

COPY --from=app /var/www/public /var/www/public

WORKDIR /var/www/public

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

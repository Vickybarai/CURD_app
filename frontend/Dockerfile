FROM node:22-alpine AS build

WORKDIR /opt

COPY . .

RUN npm install && npm run build

FROM httpd:latest 

# RUN apk add apache2

COPY --from=build /opt/dist/ htdocs/ 

# RUN cp -rf dist/* /var/www/localhost/htdocs/

EXPOSE 80

CMD ["httpd", "-D", "FOREGROUND"]

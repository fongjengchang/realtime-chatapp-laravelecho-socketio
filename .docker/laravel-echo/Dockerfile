FROM node:12.13-alpine

LABEL maintainer="Mai Trung Duc (maitrungduc1410@gmail.com)"

WORKDIR /app

RUN npm install -g pm2 laravel-echo-server

COPY echo.json /app/echo.json
COPY laravel-echo-server.json /app/laravel-echo-server.json

# RUN addgroup -g 20 www-data

#RUN adduser -D -u 501 www -G dialout

RUN chown -R node:node /app

EXPOSE 6001

USER node

CMD ["pm2-runtime", "echo.json"]

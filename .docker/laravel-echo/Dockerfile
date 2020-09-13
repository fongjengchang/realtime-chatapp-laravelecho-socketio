FROM node:12.13-alpine

LABEL maintainer="Mai Trung Duc (maitrungduc1410@gmail.com)"

WORKDIR /app

RUN npm install -g pm2 laravel-echo-server

COPY echo.json /app/echo.json
COPY laravel-echo-server.json /app/laravel-echo-server.json

#------IF host UID:GID is 1000:1000
RUN chown -R node:node /app

#------ ELSE
# RUN addgroup -g <HOST_GROUP_ID> www

# RUN adduser -D -u <HOST_USER_ID> www -G www

#RUN chown -R www:www /app

#------ END

EXPOSE 6001

#------ IF host UID:GID is 1000:1000
USER node
#------ ELSE
# USER www
#------ END

CMD ["pm2-runtime", "echo.json"]

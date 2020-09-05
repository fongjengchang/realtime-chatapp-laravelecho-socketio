# Introduction
This branch is used to run with Docker, using non-root user for all containers which is highly recommended in production.
# Notice
Running container using non-root user requires knowledge about Linux user, how container treats a non-root user and mounts volume.

You're recommended to read my blog about this to understand deeply (Vietnamese). [Check it here](https://viblo.asia/p/tai-sao-nen-chay-ung-dung-container-voi-non-root-user-jvEla3VNKkw#_the-con-redis-va-mongodb-7)
# Before begin
In your host machine, make sure you're a non-root user.

Then find out which is your User ID and Group ID by running the following commands:
```
id -u
->>> 1000

id -g
->>> 1000
```
**Notice** those UserID and GroupID above, you'll need it later
# Setup
## Install dependencies
Clone my source code, make sure you're in `docker-non-root` branch.

First generate `.env` file:
```
cp .env .example
```
Then setup database info in `.env` (remember to set user and password). Then set password for Redis in `REDIS_PASSWORD` field and `LARAVEL_ECHO_SERVER_REDIS_PASSWORD`

Next, we need to run `composer install` using intermediate container:
```
docker run --rm -v $(pwd):/app -w /app composer install --ignore-platform-reqs --no-autoloader --no-dev --no-interaction --no-progress --no-suggest --no-scripts --prefer-dist
```
After that we need to `dump` classes for PHP:
```
docker run --rm -v $(pwd):/app -w /app composer dump-autoload --classmap-authoritative --no-dev --optimize
```
Next we need to install `node_modules` and build VueJS codes:
```
docker run --rm -v $(pwd):/app -w /app node npm install --production

docker run --rm -v $(pwd):/app -w /app node npm run prod
```
Because those commands above run by root user, now we need to change owner of files in project back to current user:
```
sudo chown -R $USER:$USER .
```
Now generate key for your project:
```
php artisan key:generate
```
## Build Docker images
Next we need to build Docker images.

First open `.docker/laravel-echo/Dockerfile` and change comment sections which I highlight. If your host user has `UID:GID=1000:1000` then you don't need to do anything here

Next, open `Dockerfile` in root level of project, and check the section where I add user, if your host user is not `1000:1000` then update the value to match yours

Finally, open `docker-compose.yml` and look for services `db` and `redis`, change user to match your host `UID:GID`
## Run the app
Now it's time to launch the app.

Open terminal and run:
```
docker-compose up -d --build
```
After that you need to migrate and seed database:
```
docker-compose exec app php artisan migrate --seed
```

Now you can try the app by accessing `localhost:4000`
# About Task Scheduling (Cronjob)
Because `crond` in Linux required to be run as `root`, then after that we can create cronjob for specific non-root user. That means in this case we need to manually start `crond` by root:
```
docker-compose exec -u root app sh
crond -b
```
Then check if `crond` is running:
```
top

--->>>>
.....
139     1 root     S     1556   0%   3   0% crond -b
```
Now `crond` will look for the cronjob `/etc/crontabs/www` which we defined in Dockerfile and run them using `www` user.

Note that you have to manually start `crond` everytime your app restarts (after running `docker-compose up`)
# Digging deeper
## Using built in non-root user for MySQL and Redis
If you my blog about [running containers as non-root user](https://viblo.asia/p/tai-sao-nen-chay-ung-dung-container-voi-non-root-user-jvEla3VNKkw#_the-con-redis-va-mongodb-7), you may see that the reason we have to run database (service `db`) and `redis` with user `1000:1000` is to make sure their volumes have same permission in host machine and inside containers.

So if you don't map volume for that 2 containers, you can use the built-in non-root users which supplied by their images by default (same like service `adminer`):
```yaml
#MySQL Service
  db:
    ...
    user: mysql
  redis:
    ...
    user:redis
```
> Why do you know those built-in users?. You can find those built-in users by reading their Dockerfiles. Eg: [Adminer](https://github.com/TimWolla/docker-adminer/blob/18344428b4b12907f82ab8c03e865094d1ae0663/4/Dockerfile), [Redis](https://github.com/docker-library/redis/blob/master/5.0/alpine/Dockerfile), [MySQL](https://github.com/docker-library/mysql/blob/master/5.7/Dockerfile)
## Removal of ENTRYPOINT
If you look at Dockerfile, you may notice that there's no `ENTRYPOINT` anymore. In my code in `master` branch I use [docker-entrypoint.sh](https://github.com/maitrungduc1410/realtime-chatapp-laravelecho-socketio/blob/master/.docker/docker-entrypoint.sh) to setup Horizon, cronjob, worker,... But in this branch I put them directly in Dockerfile.

The reason is those commands in ENTRYPOINT they need to be run as `root` and by doing that the `CMD` in Dockerfil used to start container will be run as `root` - our container will run as `root` user.

I tried using `sudo`, `gosu`, `su-exec` but can't make it work, and I realize that to make container run as non-root user we have to use `USER myuser` in Dockerfile.
# Ruby on Rails

### Requirements:
- Rails 5.2.1 (latest)
- ruby 2.5.3
- postgresql
- reactjs
- imagemagick

## Setup Ruby on Rails

1. Create new project

    ```
    rails new project_name -d postgresql --webpack=react
    ```

2. Setup database

    ```
    rails db:setup
    rails db:migrate
    ```

3. Create `index_controller.rb` to make sure Rails is running on localhost


4. Start server and visit [http://localhost:3000](http://localhost:3000)

    ```
    rails server
    ```

## Setup Docker

The docker setup is modified from the [docker-rails](https://github.com/ledermann/docker-rails/) repo.

__Features:__

- Use Alpine (ruby:2.5.3-alpine) as base image - smaller image size
- Use Multi-stage build - The idea is to first build the stuff and then copy only the resulting artefacts into the final image.
- Remove bundler cache - the cache folder, C source file and compiled object file for gems with native extensions.

  ```
  RUN rm -rf /usr/local/bundle/cache/*.gem \
    && find /usr/local/bundle/gems/ -name "*.c" -delete \
    && find /usr/local/bundle/gems/ -name "*.o" -delete
  ```

- Remove parts of the app not needed in resulting image - `node_modules/`, `tmp/cache`, `spec/`, `test/` and the `assets/` folders

  ```
  # Remove folders not needed in resulting image
  RUN rm -rf node_modules tmp/cache app/assets vendor/assets lib/assets spec
  ```

5. Add `Dockerfile`

6. Build the docker image

    ```
    docker build . -t terraform-rails
    ```

    ```
    ...
    ...
    Successfully built 04e42927d47b
    Successfully tagged terraform-rails:latest
    ```

7. Check that the image is built

    ```
    docker images
    ```

    ```
    REPOSITORY          TAG       IMAGE ID        CREATED                SIZE
    terraform-rails     latest    04e42927d47b    About a minute ago     203MB
    ```

## Docker Compose

Compose is a tool for defining and running multi-container Docker applications.

Having dockerized your rails app is not enough, the app still needs to communite with other services, at the
minimal, to a database server. Rails app contain is only one of the services.

By using Compose, you can define multiple container services and how they are linked to each other.

8. Add `docker-compose.yml`

9. Build containers

    ```
    docker-compose build
    ```

10. Check if the image is built

    ```
    docker images
    ```

    ```
    REPOSITORY          TAG        IMAGE ID         CREATED             SIZE
    terraform-rails     latest     724c9b8a43e8     32 seconds ago      203MB
    ```

11. Remember to change the database config

    ```
    # docker-compose.yml
    services:
      db:
        environment:
          - POSTGRES_PASSWORD
      ...

      app: &app_base
        ...
        ...
        environment:
          - POSTGRES_HOST=db
          - POSTGRES_PASSWORD
          - POSTGRES_USER=postgres
    ```

    ```
    # config/database.yml

    default: &default
      ...
      host: <%= ENV['POSTGRES_HOST'] %>
      username: <%= ENV['POSTGRES_USER'] %>
      password: <%= ENV['POSTGRES_PASSWORD'] %>
      ...

    ```

12. Start the containers

    ```
    docker-compose up
    ```

    To stop the containers

    ```
    docker-compose down
    ```

## Nginx as web server

There are many reasons to use nginx as web server between your app and the users.

- Static redirects - you could setup your nginx to redirect all http traffic to the same url with https. This way such trivial requests will never hit your app server.
- Multipart upload - Nginx is better suited to handle multipart uploads. Nginx will combine all the requests and send it as a single file to puma.
- Serving static assets - It is recommended to serve static assets (those in /public/ endpoint in rails) via a webserver without loading your app server.
- There are some basic DDoS protections built-in in nginx.

13. Add `devops/docker/web/Dockerfile` for nginx container

14. Add `devops/docker/web/nginx.conf`

15. Configure `docker-compose.yml` to use nginx

    ```
    # docker-compose.yml
    ...
    ...
      web:
        build:
          context: .
          dockerfile: ./devops/docker/web/Dockerfile
        depends_on:
          - app
        ports:
          - 8080:80
    ...
    ...
    ```

16. Start the containers again

    ```
    docker-compose up
    ```

    Inspect your running containers, you should see 3 containers are running, only the web container are exposing the port 8080

    ```
    docker ps

    CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                  NAMES
    8cccc470542f        terraform-rails_web   "nginx -g 'daemon of…"   2 minutes ago       Up 2 minutes        0.0.0.0:8080->80/tcp   terraform-rails_web_1
    382b29806b87        terraform-rails       "./devops/docker/sta…"   2 minutes ago       Up 2 minutes        3000/tcp               terraform-rails_app_1
    9b7077ef437b        postgres:alpine       "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes        5432/tcp               terraform-rails_db_1
    ```

    Visit [http://localhost:8080](http://localhost:8080) for your app.

## docker-sync

[http://docker-sync.io](http://docker-sync.io)

Developing with docker under OSX/ Windows is a huge pain, since sharing your code into containers will __slow down__ the
code-execution about 60 times.


17. Install `docker-sync`

    See [installation guide](https://github.com/EugenMayer/docker-sync/wiki/docker-sync-on-OSX)


18. Configure `docker-sync` with `docker-sync.yml`

    ```
    # docker-sync.yml

    version: "2"

    options:
      verbose: true
    syncs:
      app-sync: # tip: add -sync and you keep consistent names as a convention
        src: './'

        # use 'native_osx' or 'unison'
        sync_strategy: 'unison'

        sync_excludes: ['.gitignore', '.git/', '.DS_Store']

        # this does not user groupmap but rather configures the server to map
        # optional: usually if you map users you want to set the user id of your application container here
        sync_userid: '1000'

        # optional, a list of regular expressions to exclude from the fswatch - see fswatch docs for details
        # IMPORTANT: this is not supported by native_osx
        watch_excludes: ['.*/.git', '.*/node_modules', '.*/bower_components', 'tmp', '.*/sass-cache', '.*/.sass-cache', '.*/.sass-cache', '.coffee', '.scss', '.sass', '.gitignore']

        # optional: use this to switch to fswatch verbose mode
        watch_args: '-v'
    ```
    See [docker-sync configuration](https://github.com/EugenMayer/docker-sync/wiki/2.-Configuration)

19. Modify `docker-compose.yml` or use `docker-compose-dev.yml` which `docker-sync` will use to override the `docker-compose.yml`

    ```
    # docker-compose-dev.yml
    version: '3.4'

    services:
      app:
        volumes:
          - terraform-rails-app-sync:/app:nocopy
      worker:
        volumes:
          - terraform-rails-app-sync:/app:nocopy

    volumes:
      terraform-rails-app-sync:
        external: true
    ```

    __WARNING:__ Because unison is a two-way sync, do not use the same volume name as your other apps, your codes might get
    replaced with your other apps.

20. Use `docker-sync` commands to replace `docker-compose` to start, stop and cleanup

    To start
    ```
    docker-sync-stack start
    ```
    To stop, just Ctrl-C

    To clean up,
    ```
    docker-sync-stack clean
    ```

## See Also

- [Tips](docs/tips.md)
- [Troubleshooting](docs/troubleshooting.md)

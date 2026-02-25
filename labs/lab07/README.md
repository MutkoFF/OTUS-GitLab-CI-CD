## CD c Docker, Docker Compose, Kubernetes
### Цели:
1) Написать CD собранного в предыдущем ДЗ приложения на сервер. Деплой должен происходить в Docker или Docker Compose;

2) Написать СD приложения, деплой должен происходить в Kubernetes. Ссылка на приложение: https://github.com/matheusfm/httpbin-chart;

  Для начала в настройках Ci/CD я создал новые переменные для указания конечного хоста, на который будет производить деплой, пользователя и SSH ключ.

  ![Картинка](./pictures/Picture01.jpg)

  Далее я добавил stage deploy в gitlab-ci файл
  ```
  stages:
  - build
  - docker_image_create
  - deploy


build:
  image: node:20-alpine
  stage: build

  script:
    - ls -la
    - cd jellyfin-web
    - ls -la
    - npm ci
    - npm run build:production
    - mv dist ../dist

  artifacts:
    paths:
      - dist
    expire_in: 2 hours
    
image_create:
  stage: docker_image_create
  
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination $CI_REGISTRY_IMAGE:latest


deploy:
  stage: deploy
  image: alpine:3

  before_script:
    - apk add --no-cache openssh-client
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519
    - ssh-keyscan -H $SERVER_HOST >> ~/.ssh/known_hosts

  script:
    - ssh $SERVER_USER@$DEPLOY_HOST "
        docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY &&
        docker pull $CI_REGISTRY_IMAGE:latest &&
        docker stop jellyfin-web || true &&
        docker rm jellyfin-web || true &&
        docker run -d -p 80:80 --name jellyfin-web $CI_REGISTRY_IMAGE:latest
      "

  only:
    - main
```

Успешное выполнение всех jobs

![Картинка](./pictures/Picture02.jpg)

Успешный запуск приложения

![Картинка](./pictures/Picture03.jpg)

Далее, я поднял ещё одну машину в YandexCloud, установил на неё кубер, настроил так, чтобы эта control plane мастер нода тоже могла запускать поды, после чего написал вот такой пайплайн
```
stages:
  - deploy

deploy:
  stage: deploy
  image: alpine:latest

  before_script:
    - apk add --no-cache openssh-client
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519
    - ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts

  script:
    - |
      ssh root@$DEPLOY_HOST "
        cd /root/httpbin-chart || git clone https://github.com/matheusfm/httpbin-chart.git /root/httpbin-chart &&
        cd /root/httpbin-chart &&
        git pull &&
        helm upgrade --install httpbin . \
          --namespace httpbin \
          --create-namespace \
          --set service.type=NodePort \
          --set service.nodePort=30080 \
          --atomic \
          --wait
      "

  only:
    - main
```

И, как видимо, под запустился, приложение задеплоилось

![Картинка](./pictures/Picture04.jpeg)

![Картинка](./pictures/Picture05.jpeg)

![Картинка](./pictures/Picture06.jpeg)
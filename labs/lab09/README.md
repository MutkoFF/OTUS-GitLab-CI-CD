## Добавление анализаторов в pipeline
### Цели:
 Для выполнения это ДЗ вы можете взять проект с уже готовым GitLab CI pipeline. Например тот, который использовался во втором модуле курса. Или вы можете работать с любым другим проектом.

1) Добавьте в проект проверки безопасности, которые на ваш взгляд, необходимы этому проекту.

2) Дайте развёрнутое и обоснованное пояснение, почему вы выбрали именно эти проверки.

Я буду добавлять проверку для gitlab-ci, который написал для сборки и развёртывания jellyfin-web в 7 домашнем задании. В итоге я получил вот такой пайплайн

```
stages:
  - security
  - build
  - docker_image_create
  - deploy

dependency_scan:
  stage: security
  image: node:20-alpine
  script:
    - cd jellyfin-web
    - npm ci
    - npm audit --audit-level=high
  allow_failure: false

secret_scan:
  stage: security
  image: zricethezav/gitleaks:latest
  script:
    - gitleaks detect --source . --verbose --redact
  allow_failure: false

dockerfile_lint:
  stage: security
  image: hadolint/hadolint:latest-debian
  script:
    - hadolint Dockerfile
  allow_failure: false

build:
  image: node:20-alpine
  stage: build
  script:
    - ls -la
    - cd jellyfin-web
    - npm ci
    - npm run build:production
    - mv dist ../dist
  artifacts:
    paths:
      - dist
    expire_in: 2 hours
  needs:
    - dependency_scan
    - secret_scan
    - dockerfile_lint

image_create:
  stage: docker_image_create
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor 
        --context $CI_PROJECT_DIR 
        --dockerfile $CI_PROJECT_DIR/Dockerfile 
        --destination $CI_REGISTRY_IMAGE:latest
  needs:
    - build

image_scan:
  stage: docker_image_create
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:latest
  needs:
    - image_create


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
  needs:
    - image_scan
```

Добавлены следующие проверки:

1) Проверка npm-зависимостей (команда npm audit --audit-level=high)

Она проверяет уязвимости в npm-пакетах, транзитивные зависимости, критические и high CVE. Это необходимо т.к. node.js-проекты используют сотни зависимостей, и большинство атак сегодня происходит через supply-chain. Даже если наш код безопасен, уязвимость в стороннем пакете может привести к XSS, Prototype pollution, Remote Code Execution. Пайплайн теперь падает при наличии high/critical уязвимостей, что предотвращает атаку потенциально опасной версии.

2) Сканирование Docker-образа (Trivy, команда trivy image --severity HIGH,CRITICAL)

Она проверяет CVE в базовом образе (например Alpine), уязвимости в системных библиотеках (openssl, musl, glibc), уязвимости зависимостей внутри контейнера. Это необходимо т.к. даже если npm-зависимости чистые, образ может содержать уязвимую базу, устаревшие системные пакеты, известные CVE.
Без этого этапа мы могли бы задеплоить контейнер с критической уязвимостью на прод. Теперь деплой невозможен, если образ содержит HIGH или CRITICAL CVE.

3) Secret Scanning (поиск утёкших секретов)

Добавлен поиск секретов в репозитории. Она проверяет приватные ключи, токены
пароли, API-ключи, .env-файлы. Это необходимо т.к. в проекте используется SSH-деплой и registry-авторизация. Если секрет случайно попадёт в git, его невозможно полностью удалить из истории, он может быть использован злоумышленником, это компрометация инфраструктуры + Secret scanning снижает риск человеческой ошибки.

4) Проверка Dockerfile (Hadolint)

Добавлена статическая проверка Dockerfile. Она проверяет использование latest, запуск контейнера от root, отсутствие pinned-версий, небезопасные инструкции, плохие практики сборки. Это необходимо т.к. ошибки в Dockerfile могут привести к нестабильным сборкам, небезопасному контейнеру, увеличенной поверхности атаки. Это превентивная защита от конфигурационных уязвимостей.

5) Блокировка деплоя при провале security-проверок

Добавлена зависимость стадий так, что:

если падает dependency scan -> нет сборки

если падает image scan -> нет деплоя

если найдены секреты -> нет сборки

если Dockerfile небезопасен -> нет сборки

Это необходимо т.к. без жёсткой зависимости можно было бы проигнорировать предупреждения, вручную задеплоить уязвимый образ. Теперь безопасность встроена в процесс доставки.

### Почему не были добавлены другие проверки

SAST

Для данного проекта (frontend без сложной backend-логики) основной риск — зависимости, а не собственный код. Поэтому приоритет отдан dependency scanning.

DAST

Проект собирается в статический фронтенд-контейнер. Dynamic Application Security Testing актуален для backend API и сложной бизнес-логики.
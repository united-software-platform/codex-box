FROM node:22-slim

ARG PROJECT_DIR=/app
ARG USER_ID=1000
ARG GROUP_ID=1000
# Версия Codex = версия образа. При сборке в CI подставляется конкретный
# номер: он же становится тегом публикации, поэтому тег и содержимое совпадают
ARG CODEX_VERSION=latest

WORKDIR ${PROJECT_DIR}

# git, openssh-client и gh — работа с репозиторием и авторизацией.
# procps даёт ps: Codex вызывает его при старте фонового app-server,
# и без него запуск падает с «failed to invoke ps», а slim-образ ps не несёт.
# wget, curl и python3 — инструменты агента: загрузка файлов, обращение
# к HTTP API и скрипты; wget заодно получает ключ репозитория gh
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        git openssh-client ca-certificates procps wget curl python3 \
    && mkdir -p -m 755 /etc/apt/keyrings /etc/apt/sources.list.d \
    && wget -nv -O /etc/apt/keyrings/githubcli-archive-keyring.gpg \
        https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    && chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
        > /etc/apt/sources.list.d/github-cli.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends gh \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g "@openai/codex@${CODEX_VERSION}" \
    && npm cache clean --force

# Пользователь node базового образа становится пользователем codex с UID/GID хоста:
# созданные агентом файлы не оказываются во владении root. Точки монтирования
# заводятся заранее: иначе Docker создаст недостающие родительские каталоги от root,
# и ~/.config станет недоступен на запись
RUN set -eux; \
    if getent group node > /dev/null; then \
        groupmod --new-name codex node; \
    elif ! getent group codex > /dev/null; then \
        groupadd --gid "${GROUP_ID}" codex; \
    fi; \
    if [ "$(getent group codex | cut -d: -f3)" != "${GROUP_ID}" ]; then \
        groupmod --gid "${GROUP_ID}" codex; \
    fi; \
    if id node > /dev/null 2>&1; then \
        usermod --login codex --home /home/codex --move-home node; \
    elif ! id codex > /dev/null 2>&1; then \
        useradd --uid "${USER_ID}" --gid codex --create-home --shell /bin/bash codex; \
    fi; \
    if [ "$(id -u codex)" != "${USER_ID}" ]; then \
        usermod --uid "${USER_ID}" codex; \
    fi; \
    mkdir -p /home/codex/.codex /home/codex/.ssh /home/codex/.config/gh; \
    chmod 700 /home/codex/.ssh; \
    chown -R codex:codex "${PROJECT_DIR}" /home/codex

USER codex

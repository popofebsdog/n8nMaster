ARG NODE_VERSION=20
ARG N8N_VERSION=snapshot

# 1. Create an image to build n8n (Builder Stage)
FROM --platform=linux/amd64 n8nio/base:${NODE_VERSION} AS builder

# Build the application from source
WORKDIR /src
# 這裡的 COPY . /src 會複製整個 Zeabur 專案（也就是你整個 GitHub 倉庫）的內容到 /src
COPY . /src
# 現在 /src 裡面會有 package.json，所以 pnpm install 會成功
RUN --mount=type=cache,id=pnpm-store,target=/root/.local/share/pnpm/store --mount=type=cache,id=pnpm-metadata,target=/root/.cache/pnpm/metadata DOCKER_BUILD=true pnpm install --frozen-lockfile --ignore-scripts
RUN pnpm build

# Delete all dev dependencies ... (省略)
# Deploy the `n8n` package into /compiled ... (省略)

# 2. Start with a new clean image with just the code that is needed to run n8n (Runtime Stage)
FROM n8nio/base:${NODE_VERSION}
ENV NODE_ENV=production

ARG N8N_RELEASE_TYPE=dev
ENV N8N_RELEASE_TYPE=${N8N_RELEASE_TYPE}

LABEL org.opencontainers.image.title="n8n"
LABEL org.opencontainers.image.description="Workflow Automation Tool"
LABEL org.opencontainers.image.source="https://github.com/n8n-io/n8n"
LABEL org.opencontainers.image.url="https://n8n.io"
LABEL org.opencontainers.image.version=${N8N_VERSION}

WORKDIR /home/node
# 複製編譯後的 n8n 程式碼
COPY --from=builder /compiled /usr/local/lib/node_modules/n8n

# 複製入口點腳本和其他設定檔。這些路徑是相對於你 Git 倉庫根目錄的。
COPY docker/images/n8n/docker-entrypoint.sh /
COPY docker/images/n8n/n8n-task-runners.json /etc/n8n-task-runners.json

# Setup the Task Runner Launcher ... (省略)

# --- 在這裡添加安裝 exiftool-vendored 的指令 ---
# 更改工作目錄到 n8n 的 node_modules 所在的實際路徑
WORKDIR /usr/local/lib/node_modules/n8n

# 安裝 exiftool-vendored
# 使用 npm 而不是 pnpm，因為 pnpm 是用於構建階段的
RUN npm install exiftool-vendored

# 將工作目錄改回 /home/node，這是後續操作預期的工作目錄
WORKDIR /home/node
# --- 添加結束 ---

RUN \
    cd /usr/local/lib/node_modules/n8n && \
    npm rebuild sqlite3 && \
    cd - && \
    ln -s /usr/local/lib/node_modules/n8n/bin/n8n /usr/local/bin/n8n && \
    mkdir .n8n && \
    chown node:node .n8n

ENV SHELL /bin/sh
USER node
ENTRYPOINT ["tini", "--", "/docker-entrypoint.sh"]
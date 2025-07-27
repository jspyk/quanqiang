# 阶段1: Node.js 环境（仅用于提供 node 和 npm）
FROM node:20-slim as nodebuilder

# 阶段2: 构建前端依赖
FROM python:3.11-slim-bullseye as builder
ARG QL_MAINTAINER="whyour"
LABEL maintainer="${QL_MAINTAINER}"
ARG QL_URL=https://github.com/${QL_MAINTAINER}/qinglong.git
ARG QL_BRANCH=v2.18.3

ENV QL_DIR=/ql \
    QL_BRANCH=${QL_BRANCH}

COPY --from=nodebuilder /usr/local/bin/node /usr/local/bin/
COPY --from=nodebuilder /usr/local/lib/node_modules/. /usr/local/lib/node_modules/
RUN set -x && \
    ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
    apt-get update && \
    apt-get install --no-install-recommends -y libatomic1 git && \
    git clone --depth=1 -b ${QL_BRANCH} ${QL_URL} ${QL_DIR}

RUN mkdir /tmp/build && \
    cp ${QL_DIR}/package.json ${QL_DIR}/.npmrc ${QL_DIR}/pnpm-lock.yaml /tmp/build/

RUN npm i -g pnpm@8.3.1 && \
    cd /tmp/build && \
    pnpm install --prod

# 阶段3: 最终镜像
FROM python:3.11-slim-bullseye

# ▼▼▼▼▼▼▼▼▼▼▼▼ 硬编码默认值（核心修改部分） ▼▼▼▼▼▼▼▼▼▼▼▼
ENV DEFAULT_ADMIN_USERNAME="15614620010" \
    DEFAULT_ADMIN_PASSWORD="Z4h0p88." \
    DEFAULT_RCLONE_CONF="[huggingface]\ntype = webdav\nurl = https://kamo.teracloud.jp/dav/\nvendor = other\nuser = zhanghaipengbeifen1\npass = PdK8wF6V9HDHHiM7482nbfCqlUSvBwjL_k-0nd8-zBk" \
    DEFAULT_NOTIFY_CONFIG='{"type": "weWorkBot", "weWorkBotKey": "323eef9d-844a-404f-a0f3-4cff479666f1"}'
# ▲▲▲▲▲▲▲▲▲▲▲▲ 默认值结束 ▲▲▲▲▲▲▲▲▲▲▲▲

# 标准环境变量
ARG QL_MAINTAINER="whyour"
LABEL maintainer="${QL_MAINTAINER}"
ARG QL_URL=https://github.com/${QL_MAINTAINER}/qinglong.git
ARG QL_BRANCH=v2.18.3

ENV PNPM_HOME=/root/.local/share/pnpm \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/root/.local/share/pnpm:/root/.local/share/pnpm/global/5/node_modules:$PNPM_HOME \
    NODE_PATH=/usr/local/bin:/usr/local/pnpm-global/5/node_modules:/usr/local/lib/node_modules:/root/.local/share/pnpm/global/5/node_modules \
    LANG=C.UTF-8 \
    SHELL=/bin/bash \
    PS1="\u@\h:\w \$ " \
    QL_DIR=/ql \
    QL_BRANCH=${QL_BRANCH}

COPY --from=nodebuilder /usr/local/bin/node /usr/local/bin/
COPY --from=nodebuilder /usr/local/lib/node_modules/. /usr/local/lib/node_modules/

RUN set -x && \
    ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
    ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx && \
    apt-get update && \
    apt-get upgrade -y && \
    apt-get install --no-install-recommends -y git curl sudo build-essential default-jdk default-jre g++ gcc libzbar0 fish ffmpeg nmap ca-certificates zsh cron wget tzdata perl openssl openssh-client nginx jq procps netcat sshpass rclone unzip libatomic1 && \
    apt-get clean && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" >/etc/timezone && \
    npm install -g pnpm@8.3.1 pm2 ts-node && \
    rm -rf /root/.pnpm-store /root/.local/share/pnpm/store /root/.cache /root/.npm && \
    chmod u+s /usr/sbin/cron && \
    ulimit -c 0

# 安装 code-server
RUN curl -fsSL https://code-server.dev/install.sh | sh -s -- --version=4.96.4

# 克隆 Qinglong 主仓库
ARG SOURCE_COMMIT
RUN git clone --depth=1 -b ${QL_BRANCH} ${QL_URL} ${QL_DIR} && \
    cd ${QL_DIR} && \
    cp -f .env.example .env && \
    chmod 777 ${QL_DIR}/shell/*.sh ${QL_DIR}/docker/*.sh && \
    git clone --depth=1 -b ${QL_BRANCH} https://github.com/${QL_MAINTAINER}/qinglong-static.git /static && \
    mkdir -p ${QL_DIR}/static && \
    cp -rf /static/* ${QL_DIR}/static && \
    rm -rf /static && \
    rm -f ${QL_DIR}/docker/docker-entrypoint.sh

# 复制关键文件（包括修改后的 entrypoint）
COPY docker-entrypoint.sh ${QL_DIR}/docker/
COPY front.conf ${QL_DIR}/docker/front.conf
COPY services.json /ql/node_modules/nodemailer/lib/well-known/
COPY notify.py /notify.py

# 创建目录并设置权限
RUN mkdir -p /ql/data/{config,log,db,scripts,repo,raw,deps} && \
    chmod -R 777 /ql /var /usr/local /etc/nginx /run /usr /root

# 配置 code-server 用户
RUN useradd -m -s /bin/zsh coder && \
    echo 'coder ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
    mkdir -p /home/coder/.local/share/code-server/User && \
    echo '{ "workbench.colorTheme": "Default Dark Modern", "telemetry.enableTelemetry": true }' > /home/coder/.local/share/code-server/User/settings.json && \
    chown -R coder:coder /home/coder

USER coder
WORKDIR ${QL_DIR}

HEALTHCHECK --interval=5s --timeout=2s --retries=20 \
    CMD curl -sf --noproxy '*' http://127.0.0.1:5400/api/health || exit 1

ENTRYPOINT ["./docker/docker-entrypoint.sh"]
VOLUME /ql/data
EXPOSE 5700

FROM node:10-alpine

RUN npm i docsify-cli -g --registry=https://registry.npm.taobao.org

ONBUILD COPY src /srv/docsify/docs
ONBUILD WORKDIR /srv/docsify

CMD ["/usr/local/bin/docsify", "serve", "docs"]
FROM node:22.22.0-trixie-slim

RUN apt update \
  && apt install -y --no-install-recommends \
    curl \
    tini \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/local/app

RUN npm install -g nodemon

COPY package*.json ./

RUN npm ci \
  && npm cache clean --force \
  && mv /usr/local/app/node_modules /node_modules

COPY . .

ENV PORT=80
EXPOSE 80

ENTRYPOINT ["/usr/bin/tini", "--"]

CMD ["node", "server.js"]

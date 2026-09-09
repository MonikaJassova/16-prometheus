FROM node:24-alpine

RUN mkdir -p /usr/app

COPY package*.json /usr/app/
COPY app /usr/app/

WORKDIR /usr/app

EXPOSE 3000

RUN npm install
CMD ["node", "server.js"]

FROM node:14

WORKDIR /app

COPY package.json .

RUN yarn install

COPY . .

ARG DEFAULT_PORT=80

ENV PORT $DEFAULT_PORT

EXPOSE $PORT

# CMD ["node", "server.js"]

CMD [ "yarn", "start" ]

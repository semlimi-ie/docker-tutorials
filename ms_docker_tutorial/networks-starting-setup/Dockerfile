FROM node

WORKDIR /app

COPY package.json .

RUN yarn install

COPY . .

CMD ["node", "app.js"]d
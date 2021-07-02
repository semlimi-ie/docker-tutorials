FROM node

WORKDIR /app

COPY package.json .

RUN yarn install

COPY . .

EXPOSE 80

ENV MONGODB_USERNAME=root

ENV MONGODB_PASSWORD=secret

CMD ["yarn", "start"]
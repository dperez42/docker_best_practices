FROM node:21.5-alpine as develop

#Labels as key value pair
LABEL Maintainer="dperez"
# Any working directory can be chosen
WORKDIR /back

## Prefer COPY Over ADD
COPY ./src/package*.json ./

RUN npm install
#RUN npm install nodemon

COPY ./src .

## Use Unprivileged Containers
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

USER appuser

EXPOSE 3000

## Prefer Array Over String Syntax
#CMD ["npm", "run", "start.dev"]
CMD ["npm", "start"]
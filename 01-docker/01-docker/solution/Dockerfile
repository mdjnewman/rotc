FROM node:15-alpine
COPY package.json /
RUN npm install express redis
COPY files/ /files/
COPY webui-dist.js /webui.js
CMD ["node", "webui.js"]
EXPOSE 80

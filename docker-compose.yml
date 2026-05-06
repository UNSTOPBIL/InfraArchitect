version: '3.9'
services:
  lb:
    image: haproxy:alpine
    ports: ["80:80"]
  web-1:
    image: nginx:alpine
  web-2:
    image: nginx:alpine
  api:
    image: python:3.10-slim
    depends_on: [db]
  db:
    image: mariadb:latest
    environment:
      - MARIADB_ROOT_PASSWORD=linuxcore_secret

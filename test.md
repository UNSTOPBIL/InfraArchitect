version: '3.9'
services:
  kali-rolling:
    image: kalilinux/kali-rolling
    tty: true
    networks:
      - target-net
  
  vulnerable-webapp:
    image: bkimminich/juice-shop
    ports:
      - "3000:3000"
    networks:
      - target-net

  db-leak:
    image: mongo:latest
    networks:
      - target-net

networks:
  target-net:
    driver: bridge

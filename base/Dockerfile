FROM alpine:latest
RUN apk add --no-cache iptables
COPY init .
RUN chmod u+x init
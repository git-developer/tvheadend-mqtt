FROM alpine:latest
RUN apk add --no-cache tzdata util-linux net-tools curl xmlstarlet jq mosquitto-clients
COPY init /
COPY app /app
CMD ["/init"]

FROM alpine:latest
RUN apk add --no-cache tzdata curl jq mosquitto-clients
COPY init /
COPY app /app
CMD ["/init"]

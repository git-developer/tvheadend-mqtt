FROM alpine
RUN apk add --no-cache tzdata curl mosquitto-clients
COPY init /
COPY app /app
CMD ["/init"]

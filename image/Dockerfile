FROM alpine
RUN apk add --no-cache tzdata curl mosquitto-clients
RUN adduser -D user
USER user
COPY init /
COPY app /app
CMD ["/init"]

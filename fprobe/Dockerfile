FROM alpine:3.19

RUN apk add --no-cache fprobe

ENTRYPOINT ["fprobe", "-i", "wlan0", "-fip", "-l", "2", "localhost:2055"]
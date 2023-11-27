FROM alpine

RUN apk update && apk upgrade && apk add --no-cache fprobe

ENTRYPOINT fprobe -i wlan0 -fip 127.0.0.1:2055
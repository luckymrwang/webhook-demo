FROM golang:1.14

WORKDIR /go/src/webhook-demo

COPY . /go/src/webhook-demo

ENV CGO_ENABLED 0 \
    GO111MODULE=on \
    GOSUMDB=off \
    GOPROXY=https://goproxy.cn

RUN  go build -o /go/bin/webhook-demo cmd/main.go

FROM alpine

LABEL maintainer="spider"

# copy the go binaries from the building stage
COPY --from=0 /go/bin /go/bin

EXPOSE 443

ENTRYPOINT ["/go/bin/webhook-demo"]

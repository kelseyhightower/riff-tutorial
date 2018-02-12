FROM golang:1.9.1
WORKDIR /go/src/github.com/kelseyhightower/riff-tutorial/helloriff
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -o helloriff \
    -tags netgo -installsuffix netgo .

FROM busybox
COPY --from=0 /go/src/github.com/kelseyhightower/riff-tutorial/helloriff .
ENTRYPOINT ["/helloriff"]

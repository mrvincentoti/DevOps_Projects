# Dockerfile
FROM golang:alpine
RUN mkdir /app
ADD . /app
WORKDIR /app
COPY . .

RUN go build -o worker ./worker/main.go
RUN go build -o starter ./starter/main.go

EXPOSE 8080

CMD ["sh", "-c", "/app/worker/main && /app/starter/main"]
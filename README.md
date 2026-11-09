# Go Web Application

This is a simple website written in Golang. It uses the `net/http` package to serve HTTP requests.

### Delivery flow

``` text
Developer push to GitHub (main)
              |
              v
       GitHub Actions
       - Go build and tests
       - golangci-lint
       - Docker build and push
       - Update Helm image tag in Git
              |
              +--------------------> Docker Hub
              |
              v
       GitHub Helm chart
              |
              v
           Argo CD
              |
              v
        K3s on Azure VM
              |
       Deployment / ReplicaSet
              |
             Pod
              |
        Go web application

Browser --> Azure NSG --> NGINX NodePort --> Ingress
                                           |
                                      Service --> Pod

## Running the server

To run the server, execute the following command:

```bash
go run main.go
```

The server will start on port 8080. You can access it by navigating to `http://localhost:8080/courses` in your web browser.

## Looks like this

![Website](static/images/golang-website.png)



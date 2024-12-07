name: CI/CD Pipeline for Node.js App

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Log in to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Docker image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/nodejs-app:latest .
          docker tag ${{ secrets.DOCKER_USERNAME }}/nodejs-app:latest ${{ secrets.DOCKER_USERNAME }}/nodejs-app:${{ github.sha }}

      - name: Push Docker image to DockerHub
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/nodejs-app:latest
          docker push ${{ secrets.DOCKER_USERNAME }}/nodejs-app:${{ github.sha }}

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.2'

      - name: Authenticate with Kubernetes cluster
        run: echo "${{ secrets.KUBECONFIG }}" > kubeconfig && export KUBECONFIG=kubeconfig

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/nodejs-app nodejs-app=${{ secrets.DOCKER_USERNAME }}/nodejs-app:latest
          kubectl rollout status deployment/nodejs-app

  notify:
    needs: build-and-deploy
    runs-on: ubuntu-latest
    steps:
      - name: Notify deployment success or failure
        if: success() || failure()
        uses: slackapi/slack-github-action@v1.23.0
        with:
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          channel-id: ${{ secrets.SLACK_CHANNEL_ID }}
          text: |
            CI/CD Pipeline Result:
            *Job*: Build and Deploy
            *Status*: ${{ job.status }}

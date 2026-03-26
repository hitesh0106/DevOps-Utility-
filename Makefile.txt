
# DevOps Utility Hub - Makefile
# Quick Docker commands


.PHONY: help build run stop clean logs health test push deploy

# Variables
IMAGE_NAME = devops-utility-hub
CONTAINER_NAME = devops-hub
PORT = 80
DOCKER_REPO = yourusername/devops-utility-hub
VERSION = 1.0.0

help: ## Show help
	@echo ""
	@echo "  DevOps Utility Hub - Commands"
	@echo ""
	@echo ""
	@echo "  make build     - Build Docker image"
	@echo "  make run       - Run container"
	@echo "  make stop      - Stop container"
	@echo "  make clean     - Remove container & image"
	@echo "  make logs      - View logs"
	@echo "  make health    - Check health"
	@echo "  make test      - Run tests"
	@echo "  make push      - Push to Docker Hub"
	@echo "  make deploy    - Full deploy"
	@echo ""

build: ## Build Docker image
	@echo " Building Docker image..."
	docker build -t $(IMAGE_NAME):$(VERSION) -t $(IMAGE_NAME):latest .
	@echo " Build complete!"

run: ## Run container
	@echo " Starting container..."
	docker run -d \
		--name $(CONTAINER_NAME) \
		-p $(PORT):8080 \
		--restart always \
		--health-cmd="wget --spider http://localhost:8080/health || exit 1" \
		--health-interval=30s \
		$(IMAGE_NAME):latest
	@echo " Running at http://localhost:$(PORT)"

run-compose: ## Run with docker-compose
	docker-compose up -d --build
	@echo " Running with docker-compose!"
 
stop: ## Stop container
	@echo " Stopping..."
	docker stop $(CONTAINER_NAME) 2>/dev/null || true
	docker rm $(CONTAINER_NAME) 2>/dev/null || true
	@echo " Stopped!"

clean: ## Full cleanup
	@echo " Cleaning..."
	docker stop $(CONTAINER_NAME) 2>/dev/null || true
	docker rm $(CONTAINER_NAME) 2>/dev/null || true
	docker rmi $(IMAGE_NAME):$(VERSION) 2>/dev/null || true
	docker rmi $(IMAGE_NAME):latest 2>/dev/null || true
	@echo " Cleaned!"

logs: ## View logs
	docker logs -f $(CONTAINER_NAME)

health: ## Health check
	@echo  Checking health..."
	@curl -s http://localhost:$(PORT)/health | python -m json.tool 2>/dev/null || \
		curl -s http://localhost:$(PORT)/health
	@echo ""

test: build ## Run tests
	@echo " Running tests..."
	docker run -d --name test-hub -p 8888:8080 $(IMAGE_NAME):latest
	@sleep 3
	@echo "Testing health endpoint..."
	@curl -sf http://localhost:8888/health && echo " Health OK" || echo " Health FAILED"
	@echo "Testing main page..."
	@curl -sf http://localhost:8888/ > /dev/null && echo "  Main page OK" || echo "  Main page FAILED"
	@echo "Testing static assets..."
	@curl -sf http://localhost:8888/css/style.css > /dev/null && echo "  CSS OK" || echo "  CSS FAILED"
	@curl -sf http://localhost:8888/js/script.js > /dev/null && echo "  JS OK" || echo "  JS FAILED"
	@docker stop test-hub && docker rm test-hub
	@echo " All tests passed!"

push: build ## Push to Docker Hub
	docker tag $(IMAGE_NAME):$(VERSION) $(DOCKER_REPO):$(VERSION)
	docker tag $(IMAGE_NAME):latest $(DOCKER_REPO):latest
	docker push $(DOCKER_REPO):$(VERSION)
	docker push $(DOCKER_REPO):latest
	@echo " Pushed to Docker Hub!"

deploy: build run ## Full deploy
	@echo " Deployed successfully!"
	@echo "   URL: http://localhost:$(PORT)"
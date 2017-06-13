VERSION=1.0-3.4.10
PROJECT_ID=google_containers
PROJECT=gcr.io/${PROJECT_ID}

all: build

build:
	docker build --pull -t ${PROJECT}/kubernetes-zookeeper:${VERSION} .

push: build
	gcloud docker -- push ${PROJECT}/kubernetes-zookeeper:${VERSION}

.PHONY: all build push

VERSION=1.0-3.4.9
PROJECT_ID=google_samples
PROJECT=gcr.io/${PROJECT_ID}

all: build

build:
	docker build --pull -t ${PROJECT}/kubernetes-zookeeper:${VERSION} .

push: build
	gcloud docker -- push ${PROJECT}/kubernetes-zookeeper:${VERSION}

.PHONY: all build push

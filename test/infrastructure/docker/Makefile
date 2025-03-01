# Copyright 2018 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# If you update this file, please follow:
# https://suva.sh/posts/well-documented-makefiles/

ROOT = ../../..

.DEFAULT_GOAL:=help

GO_VERSION ?= 1.17.3
GO_CONTAINER_IMAGE ?= docker.io/library/golang:$(GO_VERSION)

# Use GOPROXY environment variable if set
GOPROXY := $(shell go env GOPROXY)
ifeq ($(GOPROXY),)
GOPROXY := https://proxy.golang.org
endif
export GOPROXY

# Active module mode, as we use go modules to manage dependencies
export GO111MODULE=on

# This option is for running docker manifest command
export DOCKER_CLI_EXPERIMENTAL := enabled

# Directories.
ROOT_DIR := $(shell dirname $(realpath $(firstword $(MAKEFILE_LIST))))
TOOLS_DIR := $(ROOT)/hack/tools
TOOLS_BIN_DIR := $(TOOLS_DIR)/bin
BIN_DIR := bin
EXP_DIR := exp

# Set --output-base for conversion-gen if we are not within GOPATH
ifneq ($(abspath $(ROOT_DIR)),$(shell go env GOPATH)/src/sigs.k8s.io/cluster-api/test/infrastructure/docker)
	CONVERSION_GEN_OUTPUT_BASE := --output-base=$(ROOT_DIR)
endif

# Binaries.
CONTROLLER_GEN_BIN := controller-gen
CONTROLLER_GEN_VER := $(shell grep "^CONTROLLER_GEN_VER" $(ROOT)/Makefile | awk '{print $$NF}')
CONTROLLER_GEN := $(abspath $(TOOLS_BIN_DIR)/$(CONTROLLER_GEN_BIN)-$(CONTROLLER_GEN_VER))
CONVERSION_GEN_BIN := conversion-gen
CONVERSION_GEN_VER := $(shell grep "^CONVERSION_GEN_VER" $(ROOT)/Makefile | awk '{print $$NF}')
# We are intentionally using the binary without version suffix, to avoid the version
# in generated files.
CONVERSION_GEN := $(abspath $(TOOLS_BIN_DIR)/$(CONVERSION_GEN_BIN))
GOTESTSUM_BIN := gotestsum
GOTESTSUM_VER := $(shell grep "^GOTESTSUM_VER" $(ROOT)/Makefile | awk '{print $$NF}')
GOTESTSUM := $(abspath $(TOOLS_BIN_DIR)/$(GOTESTSUM_BIN)-$(GOTESTSUM_VER))
KUSTOMIZE_BIN := kustomize
KUSTOMIZE_VER := $(shell grep "^KUSTOMIZE_VER" $(ROOT)/Makefile | awk '{print $$NF}')
KUSTOMIZE := $(abspath $(TOOLS_BIN_DIR)/$(KUSTOMIZE_BIN)-$(KUSTOMIZE_VER))

$(KUSTOMIZE): # Build kustomize from tools folder.
	$(MAKE) -C $(ROOT) kustomize

# Define Docker related variables. Releases should modify and double check these vars.
REGISTRY ?= gcr.io/$(shell gcloud config get-value project)
IMAGE_NAME ?= capd-manager
CONTROLLER_IMG ?= $(REGISTRY)/$(IMAGE_NAME)
ARCH ?= $(shell go env GOARCH)
ALL_ARCH = amd64 arm arm64 ppc64le s390x

STAGING_REGISTRY ?= gcr.io/k8s-staging-cluster-api
STAGING_BUCKET ?= artifacts.k8s-staging-cluster-api.appspot.com

# TAG is set to GIT_TAG in GCB, a git-based tag of the form vYYYYMMDD-hash, e.g., v20210120-v0.3.10-308-gc61521971.
TAG ?= dev

# Allow overriding the imagePullPolicy
PULL_POLICY ?= Always

all: test manager clusterctl

help:  ## Display this help
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  make \033[36m<target>\033[0m\n"} /^[a-zA-Z_-]+:.*?##/ { printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2 } /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) } ' $(MAKEFILE_LIST)

## --------------------------------------
## Testing
## --------------------------------------

ARTIFACTS ?= $(ROOT)/_artifacts

.PHONY: test
test: ## Run tests.
	go test ./... $(TEST_ARGS)

.PHONY: test-verbose
test-verbose: ## Run tests with verbose settings.
	TEST_ARGS="$(TEST_ARGS) -v" $(MAKE) test

.PHONY: test-junit
test-junit: $(GOTESTSUM) ## Run tests with verbose setting and generate a junit report.
	(go test -json ./... $(TEST_ARGS); echo $$? > $(ARTIFACTS)/junit.infra_docker.exitcode) | tee $(ARTIFACTS)/junit.infra_docker.stdout
	$(GOTESTSUM) --junitfile $(ARTIFACTS)/junit.infra_docker.xml --raw-command cat $(ARTIFACTS)/junit.infra_docker.stdout
	exit $$(cat $(ARTIFACTS)/junit.infra_docker.exitcode)

## --------------------------------------
## Binaries
## --------------------------------------

.PHONY: manager
manager: ## Build manager binary
	go build -trimpath -o $(BIN_DIR)/manager sigs.k8s.io/cluster-api/test/infrastructure/docker

$(CONTROLLER_GEN):
	$(MAKE) -C $(ROOT) controller-gen

$(CONVERSION_GEN):
	$(MAKE) -C $(ROOT) conversion-gen

$(GOTESTSUM):
	$(MAKE) -C $(ROOT) gotestsum

## --------------------------------------
## Generate / Manifests
## --------------------------------------

.PHONY: generate
generate: $(CONTROLLER_GEN) ## Generate code
	$(MAKE) generate-manifests
	$(MAKE) generate-go

.PHONY: generate-go
generate-go: $(CONTROLLER_GEN) $(CONVERSION_GEN) ## Runs Go related generate targets
	$(CONTROLLER_GEN) \
		object:headerFile=$(ROOT)/hack/boilerplate/boilerplate.generatego.txt \
		paths=./api/... \
		paths=./$(EXP_DIR)/api/...
	(IFS=','; for i in "./api/v1alpha3,./$(EXP_DIR)/api/v1alpha3"; do find $$i -type f -name 'zz_generated.conversion*' -exec rm -f {} \;; done)
	(IFS=','; for i in "./api/v1alpha4,./$(EXP_DIR)/api/v1alpha4"; do find $$i -type f -name 'zz_generated.conversion*' -exec rm -f {} \;; done)
	$(CONVERSION_GEN) \
		--input-dirs=./api/v1alpha3 \
		--input-dirs=./api/v1alpha4 \
		--input-dirs=./$(EXP_DIR)/api/v1alpha3 \
		--input-dirs=./$(EXP_DIR)/api/v1alpha4 \
		--build-tag=ignore_autogenerated_capd \
		--extra-peer-dirs=sigs.k8s.io/cluster-api/api/v1alpha3 \
		--extra-peer-dirs=sigs.k8s.io/cluster-api/api/v1alpha4 \
		--output-file-base=zz_generated.conversion $(CONVERSION_GEN_OUTPUT_BASE) \
		--go-header-file=$(ROOT)/hack/boilerplate/boilerplate.generatego.txt

.PHONY: generate-manifests
generate-manifests: $(CONTROLLER_GEN) ## Generate manifests e.g. CRD, RBAC etc.
	$(CONTROLLER_GEN) \
		paths=./api/... \
		paths=./$(EXP_DIR)/api/... \
		paths=./$(EXP_DIR)/internal/controllers/... \
		paths=./internal/controllers/... \
		crd:crdVersions=v1 \
		rbac:roleName=manager-role \
		output:crd:dir=./config/crd/bases \
		output:webhook:dir=./config/webhook \
		webhook

## --------------------------------------
## Docker
## --------------------------------------

.PHONY: docker-pull-prerequisites
docker-pull-prerequisites:
	docker pull docker.io/docker/dockerfile:1.1-experimental
	docker pull $(GO_CONTAINER_IMAGE)
	docker pull gcr.io/distroless/static:latest

.PHONY: docker-build
docker-build: docker-pull-prerequisites ## Build the docker image for controller-manager
	DOCKER_BUILDKIT=1 docker build --build-arg builder_image=$(GO_CONTAINER_IMAGE) --build-arg goproxy=$(GOPROXY) --build-arg ARCH=$(ARCH) ../../.. -t $(CONTROLLER_IMG)-$(ARCH):$(TAG) --file Dockerfile
	MANIFEST_IMG=$(CONTROLLER_IMG)-$(ARCH) MANIFEST_TAG=$(TAG) $(MAKE) set-manifest-image
	$(MAKE) set-manifest-pull-policy

.PHONY: docker-push
docker-push: ## Push the docker image
	docker push $(CONTROLLER_IMG)-$(ARCH):$(TAG)

## --------------------------------------
## Docker — All ARCH
## --------------------------------------

.PHONY: docker-build-all ## Build all the architecture docker images
docker-build-all: $(addprefix docker-build-,$(ALL_ARCH))

docker-build-%:
	$(MAKE) ARCH=$* docker-build

.PHONY: docker-push-all ## Push all the architecture docker images
docker-push-all: $(addprefix docker-push-,$(ALL_ARCH))
	$(MAKE) docker-push-manifest

docker-push-%:
	$(MAKE) ARCH=$* docker-push

.PHONY: docker-push-manifest
docker-push-manifest: ## Push the fat manifest docker image.
	## Minimum docker version 18.06.0 is required for creating and pushing manifest images.
	docker manifest create --amend $(CONTROLLER_IMG):$(TAG) $(shell echo $(ALL_ARCH) | sed -e "s~[^ ]*~$(CONTROLLER_IMG)\-&:$(TAG)~g")
	@for arch in $(ALL_ARCH); do docker manifest annotate --arch $${arch} ${CONTROLLER_IMG}:${TAG} ${CONTROLLER_IMG}-$${arch}:${TAG}; done
	docker manifest push --purge $(CONTROLLER_IMG):$(TAG)
	MANIFEST_IMG=$(CONTROLLER_IMG) MANIFEST_TAG=$(TAG) $(MAKE) set-manifest-image
	$(MAKE) set-manifest-pull-policy

.PHONY: set-manifest-image
set-manifest-image:
	$(info Updating kustomize image patch file for manager resource)
	sed -i'' -e 's@image: .*@image: '"${MANIFEST_IMG}:$(MANIFEST_TAG)"'@' ./config/default/manager_image_patch.yaml

.PHONY: set-manifest-pull-policy
set-manifest-pull-policy:
	$(info Updating kustomize pull policy file for manager resource)
	sed -i'' -e 's@imagePullPolicy: .*@imagePullPolicy: '"$(PULL_POLICY)"'@' ./config/default/manager_pull_policy.yaml

## --------------------------------------
## Release
## --------------------------------------

RELEASE_TAG ?= $(shell git describe --abbrev=0 2>/dev/null)
RELEASE_ALIAS_TAG ?= $(PULL_BASE_REF)
RELEASE_DIR := out

$(RELEASE_DIR):
	mkdir -p $(RELEASE_DIR)/

.PHONY: release
release: clean-release ## Builds and push container images using the latest git tag for the commit.
	@if [ -z "${RELEASE_TAG}" ]; then echo "RELEASE_TAG is not set"; exit 1; fi
	@if ! [ -z "$$(git status --porcelain)" ]; then echo "Your local git repository contains uncommitted changes, use git clean before proceeding."; exit 1; fi
	git checkout "${RELEASE_TAG}"
	# Set the manifest image to the staging bucket.
	REGISTRY=$(STAGING_REGISTRY) $(MAKE) manifest-modification
	$(MAKE) release-manifests

.PHONY: manifest-modification
manifest-modification: # Set the manifest images to the staging/production bucket.
	$(MAKE) set-manifest-image MANIFEST_IMG=$(REGISTRY)/$(IMAGE_NAME) MANIFEST_TAG=$(RELEASE_TAG)
	PULL_POLICY=IfNotPresent $(MAKE) set-manifest-pull-policy

.PHONY: release-manifests
release-manifests: $(RELEASE_DIR) $(KUSTOMIZE)## Builds the manifests to publish with a release
	$(KUSTOMIZE) build config/default > $(RELEASE_DIR)/infrastructure-components.yaml

.PHONY: release-staging
release-staging: ## Builds and push container images to the staging bucket.
	REGISTRY=$(STAGING_REGISTRY) $(MAKE) docker-build-all docker-push-all release-alias-tag

.PHONY: release-staging-nightly
release-staging-nightly: ## Tags and push container images to the staging bucket. Example image tag: cluster-api-controller:nightly_main_20210121
	$(eval NEW_RELEASE_ALIAS_TAG := nightly_$(RELEASE_ALIAS_TAG)_$(shell date +'%Y%m%d'))
	$(MAKE) release-alias-tag TAG=$(RELEASE_ALIAS_TAG) RELEASE_ALIAS_TAG=$(NEW_RELEASE_ALIAS_TAG)
	# Set the manifest image to the production bucket.
	$(MAKE) manifest-modification REGISTRY=$(STAGING_REGISTRY) RELEASE_TAG=$(NEW_RELEASE_ALIAS_TAG)
	## Build the manifests
	$(MAKE) release-manifests
	# Example manifest location: artifacts.k8s-staging-cluster-api.appspot.com/components/nightly_main_20210121/infrastructure-components.yaml
	gsutil cp $(RELEASE_DIR)/* gs://$(STAGING_BUCKET)/components/$(NEW_RELEASE_ALIAS_TAG)

.PHONY: release-alias-tag
release-alias-tag: # Adds the tag to the last build tag.
	gcloud container images add-tag $(CONTROLLER_IMG):$(TAG) $(CONTROLLER_IMG):$(RELEASE_ALIAS_TAG)

## --------------------------------------
## Cleanup / Verification
## --------------------------------------

.PHONY: clean
clean: ## Remove all generated files
	$(MAKE) clean-bin

.PHONY: clean-bin
clean-bin: ## Remove all generated binaries
	rm -rf bin
	rm -rf hack/tools/bin

.PHONY: clean-release
clean-release: ## Remove the release folder
	rm -rf $(RELEASE_DIR)

.PHONY: verify
verify:
	./hack/verify-all.sh
	$(MAKE) verify-gen

.PHONY: verify-gen
verify-gen: generate
	@if !(git diff --quiet HEAD); then \
		git diff; \
		echo "generated files are out of date, run make generate"; exit 1; \
	fi

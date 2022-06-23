
ARCH = $(shell go env GOARCH)
OS = $(shell go env GOOS)

SONOBUOY_VERSION ?= 0.56.7
sonobuoy_url = https://github.com/vmware-tanzu/sonobuoy/releases/download/v${SONOBUOY_VERSION}/sonobuoy_${SONOBUOY_VERSION}_$(OS)_$(ARCH).tar.gz

curl = curl -L --silent

bins = bin/sonobuoy

include ../embedded-bins/Makefile.variables

ifeq ($(ARCH),amd64)
etcd_arch = amd64
else
etcd_arch = arm64
endif

.PHONY: all
all: $(bins) .footloose-alpine.stamp

bin:
	mkdir -p $@

bin/sonobuoy: | bin
	$(curl) $(sonobuoy_url) | tar -C bin/ -zxv $(notdir $@)

.footloose-alpine.stamp: footloose-alpine/Dockerfile
	docker build \
	  --build-arg ETCD_ARCH=$(etcd_arch) \
	  --build-arg ETCD_VERSION=$(etcd_version) \
	  --build-arg KUBE_VERSION=$(kubernetes_version) \
	  -t footloose-alpine \
	  -f footloose-alpine/Dockerfile \
	  footloose-alpine
	touch $@

check-network: bin/sonobuoy
	$(realpath bin/sonobuoy) run --wait=1200 --plugin=e2e --plugin-env=e2e.E2E_USE_GO_RUNNER=true \
		--e2e-focus='\[sig-network\].*\[Conformance\]' \
		--e2e-skip='\[Serial\]' --e2e-parallel=y \
		--kubernetes-version=v$(kubernetes_version)

check-conformance: bin/sonobuoy
	$(realpath bin/sonobuoy) run --wait=1200 \
		--mode=certified-conformance \
		--kubernetes-version=v$(kubernetes_version)

get-conformance-results: bin/sonobuoy
	$(realpath bin/sonobuoy) retrieve

TIMEOUT ?= 4m

check-ctr: TIMEOUT=10m
check-byocri: TIMEOUT=5m
# readiness check for metric tests takes between around 5 and 6 minutes.
check-metrics: TIMEOUT=6m
check-metricscraper: TIMEOUT=6m

check-calico: TIMEOUT=6m

# Establishing konnectivity tunnels with the LB in place takes a while, thus a bit longer timeout for the smoke
check-customports: TIMEOUT=6m

# Config change smoke runs actually many cases hence a bit longer timeout
check-configchange: TIMEOUT=8m

# Node role check runs several cases
check-noderole: TIMEOUT=6m

# Backup check runs two scenarios
check-backup: TIMEOUT=6m

.PHONY: $(smoketests)
include Makefile.variables

$(smoketests): .footloose-alpine.stamp
	K0S_IMAGES_BUNDLE="$(realpath ../image-bundle/bundle.tar)" K0S_PATH="$(realpath ../k0s)" go test -count=1 -v -timeout $(TIMEOUT) github.com/k0sproject/k0s/inttest/$(subst check-,,$@)

.PHONY: clean

clean:
	$(realpath bin/sonobuoy) delete
	rm -rf bin sonobuoy/*_sonobuoy_*.tar.gz .*.stamp

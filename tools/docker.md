# docker

## 项目部署
docker 部署一般使用如下文件
````
- docker-entrypoint
- Dockerfile
- Makefile
````

docker-entrypoint 可以让镜像表现的像一个可执行程序一样. 借助了 docker 的 ENTRYPOINT 功能.

Dockerfile 用于构建docker镜像. 需要对docker有一定的了解.

Makefile 用于自动化编译, 此处不再多说, 感兴趣的同学可以了解下 make/makefile.

需要注意的是, 当使用不同的基础镜像时, 其内置工具链等环境很可能时不相同的. 如在如下示例中, 当使用 alpine 作为基础镜像时, 需要将go程序静态编译(alpine 镜像内没有相应的工具链). 具体编译命令见 `Makefile/docker-build`, 解决过程参考: [docker 部署调试记录](https://github.com/everywan/note/blob/master/logs/20190327-docker.md)

----

假设项目名为 identifier, 各文件示例如下

### docker-entrypoint
```Bash
#!/bin/sh

set -e

PROJECT=identifier

exec "$@"
```

### Dockerfile
```Dockerfile
FROM alpine:latest

# 用于标示创建者邮箱, 协同项目不需要
LABEL maintainer="zhensheng.five@gmail.com"

# 修正时区为东8区
RUN apk add --no-cache --virtual .build-deps \
        tzdata \
        && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
        && echo "Asia/Shanghai" > /etc/timezone \
        && apk del .build-deps

ENV TZ "Asia/Shanghai"

# 安装ca证书
RUN apk add --update --no-cache \
    ca-certificates \
    && rm -rf /var/cache/apk/*

COPY identifier /usr/local/bin/identifier
COPY docker-entrypoint /usr/local/bin/

WORKDIR /usr/local/var/identifier

EXPOSE 8080

ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
```

### Makefile
```Makefile
NAME = identifier
PACKAGE = github.com/everywan/identifier
MAIN = $(PACKAGE)/entry

DEFAULT_TAG = identifier:latest
DEFAULT_BUILD_TAG = latest
REMOTE_IMAGE = registry.cn.aliyuncs.com/everywan/identifier

BUILD_FLAGS= -mod vendor -v -o $(NAME) entry/main.go

REMOTE_TAG = $(shell git tag -l --sort=-v:refname|head -1)
ifeq "$(MODE)" "dev"
	REMOTE_TAG = staging
endif

ifeq "$(REMOTE_TAG)" ""
	REMOTE_TAG = latest
endif
REMOTE_IMAGE_TAG = "$(REMOTE_IMAGE):$(REMOTE_TAG)"

ifeq "$(BUILD_TAG)" ""
	BUILD_TAG = $(DEFAULT_BUILD_TAG)
endif

CL_RED  = "\033[0;31m"
CL_BLUE = "\033[0;34m"
CL_GREEN = "\033[0;32m"
CL_ORANGE = "\033[0;33m"
CL_NONE = "\033[0m"

define color_out
	@echo $(1)$(2)$(CL_NONE)
endef

docker-build:export GO111MODULE=on
docker-build:export CGO_ENABLED=0
docker-build:
	@go mod vendor
	$(call color_out,$(CL_BLUE),"Building binary in docker ...")
#	@docker run --rm -v "$(PWD)":/go/src/$(PACKAGE) \
#		-w /go/src/$(PACKAGE) \
#		golang:$(BUILD_TAG) \
#		go build --ldflags '-extldflags "-static"' -v -o $(NAME) $(MAIN)
	@go build --ldflags '-extldflags "-static"' -v -o $(NAME) $(MAIN)
	$(call color_out,$(CL_GREEN),"Building binary ok")

docker: docker-build
	$(call color_out,$(CL_BLUE),"Building docker image ...")
	@docker build -t $(DEFAULT_TAG) .
	$(call color_out,$(CL_GREEN),"Building docker image ok")

push: docker
	@docker tag $(DEFAULT_TAG) $(REMOTE_IMAGE_TAG)
	$(call color_out,$(CL_BLUE),"Pushing image $(REMOTE_IMAGE_TAG) ...")
	@docker push $(REMOTE_IMAGE_TAG)
	$(call color_out,$(CL_ORANGE),"Done")

build:
	@go mod vendor
	@go build $(BUILD_FLAGS)

linux:
	@go mod vendor
	@GOOS=linux GOARCH=amd64 go build $(BUILD_FLAGS)

# proto:
	# If build proto failed, make sure you have protoc installed and:
	# go get -u github.com/google/protobuf
	# go get -u github.com/golang/protobuf/protoc-gen-go
	# go install github.com/mwitkow/go-proto-validators/protoc-gen-govalidators
	# mkdir -p $GOPATH/src/github.com/googleapis && git clone git@github.com:googleapis/googleapis.git $GOPATH/src/github.com/googleapis/
# @protoc \
#		--proto_path=${GOPATH}/src \
#		--proto_path=${GOPATH}/src/github.com/google/protobuf/src \
#		--proto_path=${GOPATH}/src/github.com/googleapis/googleapis \
#		--proto_path=. \ --include_imports \
#		--include_source_info \
#		--go_out=plugins=grpc:$(PWD)/pb \
#		--govalidators_out=$(PWD)/pb \
#	$(call color_out,$(CL_ORANGE),"Done")

#.PHONY: all
all:
	build
```

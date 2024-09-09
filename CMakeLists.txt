# Copyright (c) 2024 dingodb.com, Inc. All Rights Reserved
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

cmake_minimum_required(VERSION 3.23.1 FATAL_ERROR)

project(EtcdClient)

set(GO_VERSION 1.15.15)
set(PLATFORM "")


if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
    set(PLATFORM "amd64")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
    set(PLATFORM "arm64")
endif()

set(WORK_DIR ${CMAKE_BINARY_DIR})

add_custom_target(
    install-go
    COMMAND ${CMAKE_COMMAND} -E make_directory ${WORK_DIR}/tmp
    COMMAND ${CMAKE_COMMAND} -E chdir ${WORK_DIR}/tmp wget -N https://golang.google.cn/dl/go${GO_VERSION}.linux-${PLATFORM}.tar.gz
    COMMAND ${CMAKE_COMMAND} -E chdir ${WORK_DIR}/tmp tar zxvf go${GO_VERSION}.linux-${PLATFORM}.tar.gz
)

add_custom_target(
    install-etcdclient
    COMMAND ${CMAKE_COMMAND} -E make_directory ${WORK_DIR}/tmp/gosrc/src/go.etcd.io
    COMMAND ${CMAKE_COMMAND} -E chdir ${WORK_DIR}/tmp/gosrc/src/go.etcd.io git clone --branch dingo-v3.4.18  --depth=1 https://github.com/chuandew/etcd.git
)

set(VENDORPATH ${WORK_DIR}/tmp/gosrc/src/go.etcd.io/etcd/vendor)
add_custom_target(
    libetcdclient
    DEPENDS install-go install-etcdclient
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/etcdclient.go ${WORK_DIR}/tmp/gosrc/src
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/objectManager.go ${WORK_DIR}/tmp/gosrc/src
    COMMAND ${CMAKE_COMMAND} -E rename ${VENDORPATH}/google.golang.org ${WORK_DIR}/tmp/gosrc/src/google.golang.org
    COMMAND ${CMAKE_COMMAND} -E rename ${VENDORPATH}/golang.org ${WORK_DIR}/tmp/gosrc/src/golang.org
    COMMAND ${CMAKE_COMMAND} -E rename ${VENDORPATH}/github.com ${WORK_DIR}/tmp/gosrc/src/github.com
    COMMAND ${CMAKE_COMMAND} -E env GOROOT=${WORK_DIR}/tmp/go  GOPATH=${WORK_DIR}/tmp/gosrc
        ${WORK_DIR}/tmp/go/bin/go build -buildmode=c-shared -o ${WORK_DIR}/libetcdclient.so ${WORK_DIR}/tmp/gosrc/src/etcdclient.go ${WORK_DIR}/tmp/gosrc/src/objectManager.go
)

add_custom_target(build_etcd_client ALL DEPENDS install-go install-etcdclient libetcdclient)

# Copyright 2024 The Kubernetes Authors. All rights reserved.
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
# With BCI as base image, with the SLES 15 SP6 tag, which has a newer version of glibc
FROM registry.suse.com/bci/bci-base:15.6 AS builder
 
RUN zypper addrepo -p 105 http://download.opensuse.org/distribution/leap/15.6/repo/oss/ download.opensuse.org-oss && \
    zypper --gpg-auto-import-keys refresh

COPY patches /patches
COPY --chmod=0755 build.sh /

RUN /build.sh


FROM registry.suse.com/bci/bci-base:15.6 AS packages

RUN zypper addrepo \
    -p 105 http://download.opensuse.org/distribution/leap/15.6/repo/oss/ download.opensuse.org-oss && \
    zypper --gpg-auto-import-keys refresh

# Install binaries for copying to the final image
RUN zypper install -y --allow-vendor-change \
    libcap-progs \
    nano \
    catatonit

# Download packages for installing in the final image
RUN zypper install -y --allow-vendor-change --download-only \
    liblmdb-0_9_30 \
    libxml2-tools \
    libmaxminddb0 \
    libcap-progs \    
    crypto-policies-scripts \
    libgrpc++1 \
    libatomic1 \
    grpc-devel \
    wget \
    which \
    git \
    libyajl2 \
    openssl-3 \
    libicu73_2 \
    liblua5_4-5 \
    make \
    tar \
    gzip \
    procps \
    catatonit \
    # The following packages are dependencies 
    # needed for the smaller bci-micro image
    liblua5_3 \
    python3 

# Setup group and user we will run nginx as
RUN groupadd -rg 101 www-data && \
    useradd -u 101 -M -d /usr/local/nginx -s /sbin/nologin -G www-data -g www-data www-data


FROM registry.suse.com/bci/bci-micro:15.6

ENV PATH=$PATH:/usr/local/luajit/bin:/usr/local/nginx/sbin:/usr/local/nginx/bin

ENV LUA_PATH="/usr/local/share/luajit-2.1.0-beta3/?.lua;/usr/local/share/lua/5.1/?.lua;/usr/local/lib/lua/?.lua;;"
ENV LUA_CPATH="/usr/local/lib/lua/?/?.so;/usr/local/lib/lua/?.so;;"

# Copy built artifacts from builder stage
COPY --from=builder /usr/local /usr/local
COPY --from=builder /opt /opt
COPY --from=builder /etc/nginx /etc/nginx
# All of these are dependencies for the nginx binary
COPY --from=builder /usr/lib64/libopentelemetry* /usr/lib64/
COPY --from=builder /usr/lib64/libcrypt* /usr/lib64/
COPY --from=builder /usr/lib64/libpcre* /usr/lib64/
COPY --from=builder /usr/lib64/libssl* /usr/lib64/
COPY --from=builder /usr/lib64/libz* /usr/lib64/
COPY --from=builder /usr/lib64/libjitter* /usr/lib64/


# Copy binaries from packages stage
COPY --from=packages /usr/bin/find /usr/bin/find
COPY --from=packages /usr/sbin/setcap /usr/sbin/setcap
COPY --from=packages /usr/bin/catatonit /usr/bin/catatonit
COPY --from=packages /usr/bin/nano /usr/bin/nano
COPY --from=packages /usr/lib64/libmagic* /usr/lib64/

# Get the rpm command and its dependencies
COPY --from=packages /usr/bin/rpm /usr/bin/rpm
COPY --from=packages /usr/lib/rpm /usr/lib/rpm
COPY --from=packages /usr/lib64/librpm* /usr/lib64/
COPY --from=packages /usr/lib64/libpopt* /usr/lib64/

# Copy and install RPMs from packages stage
COPY --from=packages /var/cache/zypp/packages /tmp/rpms
RUN find /tmp/rpms -name "*.rpm" -exec rpm -ivh --nodeps {} \; || true && \
    find /tmp/rpms -name "*.rpm" -exec rpm -ivh {} \; && \
    rm -rf /tmp/rpms

RUN ldDirs=" \
      /usr/local/lib \
      /usr/local/lib64 \
  "; \
  for dir in ${ldDirs}; do \
    echo "${dir}" >>/etc/ld.so.conf.d/local.conf; \
  done
RUN /sbin/ldconfig

COPY --from=packages /etc/group /etc/group
COPY --from=packages /etc/passwd /etc/passwd

RUN ln -s /usr/local/nginx/sbin/nginx /sbin/nginx
RUN writeDirs=" \
      /var/log/nginx \
      /var/lib/nginx/body \
      /var/lib/nginx/fastcgi \
      /var/lib/nginx/proxy \
      /var/lib/nginx/scgi \
      /var/lib/nginx/uwsgi \
      /var/log/audit \
  "; \
  for dir in ${writeDirs}; do \
    mkdir -p ${dir}; \
    chown -R www-data.www-data ${dir}; \
  done

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]

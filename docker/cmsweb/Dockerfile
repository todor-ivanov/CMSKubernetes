FROM cern/cc7-base:20201101-1.x86_64
MAINTAINER Valentin Kuznetsov vkuznet@gmail.com

ENV WDIR=/data/cmsweb
RUN mkdir -p $WDIR
ADD http://repository.egi.eu/sw/production/cas/1/current/repo-files/EGI-trustanchors.repo /etc/yum.repos.d/egi.repo
ADD http://linuxsoft.cern.ch/wlcg/wlcg-centos7.repo /etc/yum.repos.d/wlcg-centos7.repo
ADD hadoop.repo /etc/yum.repos.d/hadoop.repo
ADD epel.repo /etc/yum.repos.d/epel.repo
ADD slc7-cernonly.repo /etc/yum.repos.d/slc7-cernonly.repo
ADD RPM-GPG-KEY-wlcg /etc/pki/rpm-gpg/RPM-GPG-KEY-wlcg
ADD logstash.repo /etc/yum.repos.d/logstash.repo

# add logstash GPG key
RUN rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch && yum update -y && yum clean all

# add necessary RPMs for cmsweb deployment
RUN yum install -y cern-get-certificate fetch-crl \
    git-core zip unzip which file bzip2 e2fsprogs e2fsprogs-libs compat-libstdc++-33 \
    CERN-CA-certs ca-certificates dummy-ca-certs ca-policy-lcg ca-policy-egi-core \
    ca_EG-GRID ca_CERN-GridCA ca_CERN-LCG-IOTA-CA ca_CERN-Root-2 \
    wlcg-voms-cms krb5-workstation krb5-libs pam_krb5 myproxy voms-clients-cpp voms-clients-java \
    sudo openssl openssl-libs openssh openssh-clients python-backports-ssl_match_hostname \
    cmake voms voms-devel globus-gsi-credential-devel globus-gsi-cert-utils-devel \
    globus-common-devel globus-gsi-sysconfig globus-gsi-sysconfig-devel globus-gsi-callback-devel \
    globus-gsi-openssl-error globus-gsi-proxy-ssl globus-openssl-module \
    shibboleth log4shib xmltooling-schemas opensaml-schemas \
    perl-Thread-Queue zsh tk freetype perl-ExtUtils-Embed fontconfig \
    perl-Test-Harness perl-Data-Dumper perl-Digest-MD5 perl-Switch perl-Env \
    libX11-devel libX11 libXmu libSM libICE libXcursor libXext libXrandr libXft \
    mesa-libGLU mesa-libGL libXi libXinerama libXft-devel libXrender libXpm \
    libXpm-devel libXext-devel mesa-libGLU-devel \
    libaio libaio-devel net-tools lsof bind-utils initscripts patch \
    voms-devel globus-gsi-credential-devel globus-gsi-cert-utils-devel \
    globus-common-devel globus-gsi-sysconfig-devel globus-gsi-callback-devel \
    oracle-instantclient-tnsnames.ora filebeat \
    HEP_OSlibs python-pip hadoop-bin hadoop-tools \
    spark spark-bin cern-hadoop-client cern-hadoop-xrootd-connector \
    && yum clean all && rm -rf /var/cache/yum

#RUN pip install --upgrade pip setuptools
RUN cp /usr/hdp/spark/kubernetes/dockerfiles/spark/entrypoint.sh /usr/bin/entrypoint.sh && touch /etc/hadoop/conf/topology.table.file && ln -s /bin/bash /usr/bin/bashs

# cd to workdir
WORKDIR $WDIR

# Install latest kubectl
RUN curl -k -O -L https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && mv kubectl /usr/bin && chmod +x /usr/bin/kubectl && curl -k -L -O https://dl.google.com/go/go1.12.1.linux-amd64.tar.gz && tar xfz go1.12.1.linux-amd64.tar.gz && rm go1.12.1.linux-amd64.tar.gz
ENV GOROOT=$WDIR/go
ENV PATH="${GOROOT}/bin:${WDIR}:${PATH}"

# get go dependencies which we'll use on all cmsweb nodes
ENV GOPATH=$WDIR/gopath
RUN mkdir -p $GOPATH
ENV PATH="${GOROOT}/bin:${WDIR}:${PATH}"
RUN go get github.com/dmwm/cmsauth && go get github.com/vkuznet/x509proxy && go get github.com/buger/jsonparser && go get github.com/Lusitaniae/apache_exporter && go get github.com/prometheus/client_golang/prometheus && go get github.com/prometheus/common/log && go get -d github.com/vkuznet/cmsweb-exporters && go get github.com/shirou/gopsutil && go get github.com/ryotarai/prometheus-query && mkdir ${WDIR}/bin

# download node exporter
WORKDIR ${WDIR}
RUN curl -k -L -O https://github.com/prometheus/node_exporter/releases/download/v0.15.2/node_exporter-0.15.2.linux-amd64.tar.gz && tar xfz node_exporter-0.15.2.linux-amd64.tar.gz && cp node_exporter*/node_exporter $WDIR/bin && rm -r node_exporter-0.15.2.linux-amd64/ && rm -r node_exporter-0.15.2.linux-amd64.tar.gz

# build exporters
WORKDIR $GOPATH/src/github.com/vkuznet/cmsweb-exporters
RUN go build das2go_exporter.go && cp das2go_exporter $WDIR/bin && go build reqmgr_exporter.go && cp reqmgr_exporter $WDIR/bin && go build process_exporter.go && cp process_exporter $WDIR/bin && cp process_monitor.sh $WDIR/bin && go build wmcore_exporter.go && cp wmcore_exporter $WDIR/bin && go build cpy_exporter.go && cp cpy_exporter $WDIR/bin && go build cmsweb-ping.go && cp cmsweb-ping $WDIR/bin

ENV PATH="${GOPATH}/src/github.com/vkuznet/cmsweb-exporters:${GOPATH}/src/github.com/vkuznet/cmsweb-exporters/bin:${PATH}"

# setup final environment
ENV PATH="${WDIR}/bin:${GOPATH}/bin:${PATH}"

# build mongo exporter
WORKDIR ${WDIR}
RUN curl -k -L -O "https://github.com/dcu/mongodb_exporter/archive/v1.0.0.tar.gz" && tar xfz v1.0.0.tar.gz && mkdir -p $GOPATH/src/github.com/dcu/ && cp -r mongodb_exporter-1.0.0 $GOPATH/src/github.com/dcu/mongodb_exporter && rm -f v1.0.0.tar.gz
WORKDIR $GOPATH/src/github.com/dcu/mongodb_exporter
RUN go get github.com/Masterminds/glide && make build && cp mongodb_exporter $WDIR/bin

WORKDIR ${WDIR}
ADD print_hmac $WDIR/bin/print_hmac
ADD proxy.sh $WDIR/bin/proxy.sh
ADD hadoop-env.sh $WDIR/bin/hadoop-env.sh
ADD cmsweb-logs.sh $WDIR/bin/cmsweb-logs.sh

# add fetch-crl to fetch all sertificates
RUN echo "32 */6 * * * root ! /usr/sbin/fetch-crl -q -r 360" > /etc/cron.d/fetch-crl-docker
# example how to run cmsweb-logs rsync
# RUN echo "*/10 * * * * /data/cmsweb/bin/cmsweb-logs.sh /etc/secrets/id_dsa_public /data/srv/logs cmsweb@vocms055:/build/k8s-logs" > /etc/cron.d/cmsweb-logs

# start the setup
WORKDIR ${WDIR}

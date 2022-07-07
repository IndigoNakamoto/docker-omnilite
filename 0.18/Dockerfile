FROM debian:stable-slim

LABEL maintainer="Indigo (@indigo_nakamoto)"

RUN useradd -r omnilite \
  && apt-get update -y \
  && apt-get install -y curl gnupg gosu \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

ARG TARGETPLATFORM
ENV OMNILITE_VERSION=0.18.1
ENV LITECOIN_DATA=/home/litecoin/.litecoin
ENV PATH=/opt/litecoin-${OMNILITE_VERSION}/bin:$PATH

RUN set -ex \
#  && if [ "${TARGETPLATFORM}" = "linux/amd64" ]; then export TARGETPLATFORM=x86_64-linux-gnu; fi \
  && if [ "${TARGETPLATFORM}" = "linux/arm64" ]; then export TARGETPLATFORM=aarch64-linux-gnu; fi \
  && if [ "${TARGETPLATFORM}" = "linux/arm/v7" ]; then export TARGETPLATFORM=arm-linux-gnueabihf; fi \
  && for key in \
#    01EA5486DE18A882D4C2684590C8019E36C2E964 \
  ; do \
      gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" || \
      gpg --batch --keyserver pgp.mit.edu --recv-keys "$key" || \
      gpg --batch --keyserver keyserver.pgp.com --recv-keys "$key" || \
      gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key" || \
      gpg --batch --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$key" ; \
    done \
#  && curl -SLO https://github.com/OmniLite/omnilite/releases/download/v${OMNILITE_VERSION}/SHA256SUMS.asc \
  && curl -SLO https://github.com/OmniLite/omnilite/releases/download/v${OMNILITE_VERSION}-${TARGETPLATFORM}.tar.gz \
  && gpg --verify SHA256SUMS.asc \
  && grep " litecoin-${LITECOIN_VERSION}-${TARGETPLATFORM}}.tar.gz$" SHA256SUMS.asc | sha256sum -c - \
  && tar -xzf *.tar.gz -C /opt \
  && rm *.tar.gz *.asc

COPY docker-entrypoint.sh /entrypoint.sh

VOLUME ["/home/litecoin/.litecoin"]

EXPOSE 9332 9333 19332 19333 19443 19444 39333 39332

ENTRYPOINT ["/entrypoint.sh"]

RUN litecoind -version | grep "OmniLite Version v${LITECOIN_VERSION}"

CMD ["litecoind"]
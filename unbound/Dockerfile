FROM debian:buster-slim AS build

RUN sed -ni.bak 'p; s/^deb /deb-src /p' /etc/apt/sources.list
RUN apt-get update \
	&& apt-get build-dep -y unbound \
	&& apt-get install -y curl libmnl-dev libsodium-dev

WORKDIR /usr/src/unbound

ARG UNBOUND_URL=https://nlnetlabs.nl/downloads/unbound/unbound-1.12.0.tar.gz
ARG UNBOUND_SHA512=90d99bc65e9ba62e50a7809dbf1e98889d0fc9fd50cf3cc99b726c67bcaeda0c2bc176d09f84771adb9796833b595591462f96e949d6969a47d6898d8fae3479

RUN curl -fsSL --retry 3 "${UNBOUND_URL}" -o unbound.tar.gz \
	&& echo "${UNBOUND_SHA512}  unbound.tar.gz" | sha512sum -c - \
	&& tar xzf unbound.tar.gz --strip 1

RUN ./configure \
	--disable-rpath \
	--enable-cachedb \
	--enable-dnscrypt \
	--enable-dnstap \
	--enable-ipset \
	--enable-pie \
	--enable-subnet \
	--enable-tfo-client \
	--enable-tfo-server \
	--libdir=/usr/lib \
	--with-chroot-dir="" \
	--with-libevent \
	--with-pidfile=/run/unbound.pid \
	--with-pthreads \
	--with-dnstap-socket-path=/run/dnstap.sock \
	--with-rootkey-file=/var/lib/unbound/root.key 

RUN make && make install DESTDIR=/build

FROM debian:buster-slim

COPY --from=build /build /

ADD ./docker-entrypoint.sh /docker-entrypoint.sh

RUN set -ex \
	#
	&& apt-get update  \
	&& apt-get install -y dns-root-data libcap2-bin libevent-2.1-6 libmnl0 libprotobuf-c1 libsodium23 libssl1.1 tini unbound-anchor \
	#
	&& chmod +x /docker-entrypoint.sh \
	#
	&& useradd -Mr unbound \
	&& mkdir -p /var/lib/unbound \
	&& cp /usr/share/dns/root.key /var/lib/unbound/root.key \
	&& chown -R unbound:unbound /var/lib/unbound \
	#
	&& setcap 'cap_setgid+ep cap_net_bind_service+ep' /usr/local/sbin/unbound

USER unbound

EXPOSE 53

ENTRYPOINT [ "/usr/bin/tini", "--", "/docker-entrypoint.sh" ]

CMD [ "/usr/local/sbin/unbound", "-c", "/usr/local/etc/unbound/unbound.conf", "-d" ]
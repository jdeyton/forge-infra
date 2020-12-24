FROM forge-base:latest
RUN apk add --no-cache openntpd
RUN mkdir -m 755 -p /var/run/ntp/tmp && \
    chown root:root /var/run/ntp && \
    chown ntp:root /var/run/ntp/tmp && \
    sed -i 's#^\(ntp:.*:\)/var/empty#\1/var/run/ntp#' /etc/passwd
EXPOSE 123/udp
CMD ./init && ntpd -ds
COPY iptables.rules .
COPY ntpd.conf /etc/ntpd.conf
# In the Dockerfile

```
# - Systemd hacks to work nice with a docker container
#   https://github.com/solita/docker-systemd/blob/master/Dockerfile
#   https://www.freedesktop.org/software/systemd/man/systemd.html
RUN systemctl set-default multi-user.target \
    && systemctl disable systemd-remount-fs keyboard-setup \
    && systemctl disable NetworkManager ufw openvpn rsync rsyslog wpa_supplicant console-setup irqbalance ondemand \
    && systemctl enable console-getty \
    && mkdir -p /etc/systemd/system-generators \
    && touch /etc/systemd/system-generators/systemd-gpt-auto-generator \
    && chmod a+x /etc/systemd/system-generators/systemd-gpt-auto-generator
RUN echo '#!/bin/bash\n\
chgrp 1001 /var/run/docker.sock' >/usr/local/lib/docker-for-user.sh \
&& chmod 0755 /usr/local/lib/docker-for-user.sh
RUN echo '[Service]\n\
Type=oneshot\n\
RemainAfterExit=yes\n\
ExecStart=/usr/local/lib/docker-for-user.sh\n\
[Install]\n\
WantedBy=multi-user.target' >/etc/systemd/system/docker-for-user.service && systemctl enable docker-for-user

# - Set up DNS resolving rules for systemd-resolved

# Use normal DNS from 8.8.8.8. We route corporate related domains to 10.2.3.4.

# The /etc/resolv.conf is configured with the --dns option at docker
# start time.
RUN echo '\n\
[Resolve]\n\
DNS=8.8.8.8\n\
DNSOverTLS=no\n\
Cache=no\n\
ReadEtcHosts=no' >/etc/systemd/resolved.conf
RUN echo '#!/bin/bash\n\
resolvectl llmnr eth0 no\n\
resolvectl dnsovertls eth0 no\n\
resolvectl dns eth0 10.2.3.4\n\
resolvectl domain eth0 ~company.com ~company.org ~10.in-addr.arpa' >/usr/local/lib/set-up-resolved.sh \
&& chmod 0755 /usr/local/lib/set-up-resolved.sh
RUN echo '[Service]\n\
Type=oneshot\n\
RemainAfterExit=yes\n\
ExecStart=/usr/local/lib/set-up-resolved.sh\n\
[Unit]\n\
After=systemd-resolved.service\n\
Requires=systemd-resolved.service\n\
[Install]\n\
WantedBy=multi-user.target' >/etc/systemd/system/set-up-resolved.service && systemctl enable set-up-resolved

# This hack fixes /docker-desktop/proc unmounting, which by default
# results in setting /proc read-only on the host itself...
RUN echo '[Mount]\nLazyUnmount=yes' >'/etc/systemd/system/docker\x2ddesktop-proc.mount'

WORKDIR /
RUN echo '#!/bin/bash\n\
[[ "$1" ]] && exec "$@"\n\
exec /sbin/init' >/init.sh ; chmod 0700 /init.sh
STOPSIGNAL SIGRTMIN+3
ENTRYPOINT ["/init.sh"]
```

# How to start

```
docker create --security-opt=seccomp=unconfined --cap-add=NET_ADMIN --shm-size=4g --privileged \
  --tmpfs /run --tmpfs /run/lock -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  -v /:/docker-desktop -v /var/run/docker.sock:/var/run/docker.sock -v /usr/local/bin/docker:/usr/bin/docker \
  --dns=127.0.0.53 \
  --name xxx -h xxx -v myhome-home:/home -it \
  image
```

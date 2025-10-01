# Build Ubuntu 24.04 (noble) template (requires dab >= 3.7.0)
BASEDIR:=$(shell dab basedir)
export DAB_DEBUG=1

all: info/init_ok
	# 1) Bootstrap minimal base system
	dab bootstrap --minimal

	# 2) Minimize APT/DPKG footprint and noise
	echo 'APT::Install-Recommends "false";' > ${BASEDIR}/etc/apt/apt.conf.d/00norecommends
	echo 'Acquire::Languages "none";'      > ${BASEDIR}/etc/apt/apt.conf.d/01nolang
	echo 'Dpkg::Use-Pty "false";'          > ${BASEDIR}/etc/apt/apt.conf.d/99nopy
	echo 'Acquire::Retries "3";'           > ${BASEDIR}/etc/apt/apt.conf.d/80retries
	mkdir -p ${BASEDIR}/etc/dpkg/dpkg.cfg.d
	echo 'path-exclude=/usr/share/man/*'           >  ${BASEDIR}/etc/dpkg/dpkg.cfg.d/01nodoc
	echo 'path-exclude=/usr/share/doc/*'           >> ${BASEDIR}/etc/dpkg/dpkg.cfg.d/01nodoc
	echo 'path-include=/usr/share/doc/*/copyright' >> ${BASEDIR}/etc/dpkg/dpkg.cfg.d/01nodoc

	# 3) Journald settings: small, volatile, ratelimited
	mkdir -p ${BASEDIR}/etc/systemd/journald.conf.d
	echo -e "[Journal]\nStorage=volatile\nRuntimeMaxUse=64M" > ${BASEDIR}/etc/systemd/journald.conf.d/10-volatile.conf
	echo -e "[RateLimit]\nInterval=30s\nBurst=200" > ${BASEDIR}/etc/systemd/journald.conf.d/20-ratelimit.conf

	# 4) Locale & timezone (UTC only, containers use host clock)
	echo -e "LANG=C.UTF-8\nLC_ALL=C.UTF-8" > ${BASEDIR}/etc/default/locale
	echo "UTC" > ${BASEDIR}/etc/timezone
	dab exec ln -sf /usr/share/zoneinfo/Etc/UTC /etc/localtime
	dab exec systemctl mask systemd-timesyncd.service systemd-timesyncd.socket

	# 5) Purge unnecessary packages first (reduce image size & noise)
	# stop/mask noisy services before purge
	dab exec systemctl disable --now snapd.seeded.service snapd.socket snapd.service 2>/dev/null || true
	dab exec systemctl disable --now apt-daily.service apt-daily.timer apt-daily-upgrade.service apt-daily-upgrade.timer 2>/dev/null || true

	# purge typical bloat and unused packages for LXC
	dab exec dpkg --purge \
	  postfix man-db cron anacron \
	  snapd cloud-init ubuntu-advantage-tools ubuntu-pro-client ubuntu-pro-client-l10n \
	  apport python3-apport whoopsie \
	  network-manager avahi-daemon \
	  plymouth plymouth-theme* \
	  update-notifier update-manager landscape-common \
	  friendly-recovery installation-report info doc-debian \
	  irqbalance kbd console-setup os-prober grub-common \
	  mdadm lvm2 multipath-tools \
	  byobu ubuntu-release-upgrader-core popularity-contest \
	  2>/dev/null || true

	# 6) Install only the needed packages (minimal but functional)
	#dab exec apt-get update
	dab exec apt-get install -y \
	  curl wget logrotate ca-certificates openssh-server

	# 7) SSH hardening (root key-only login, no password auth, stricter limits)
	sed -ri 's/^[# ]*PermitRootLogin.*/PermitRootLogin prohibit-password/' ${BASEDIR}/etc/ssh/sshd_config || true
	grep -q '^PermitRootLogin ' ${BASEDIR}/etc/ssh/sshd_config || echo 'PermitRootLogin prohibit-password' >> ${BASEDIR}/etc/ssh/sshd_config
	sed -ri 's/^[# ]*PasswordAuthentication.*/PasswordAuthentication no/' ${BASEDIR}/etc/ssh/sshd_config || true
	sed -ri 's/^[# ]*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' ${BASEDIR}/etc/ssh/sshd_config || true
	sed -ri 's/^[# ]*UseDNS.*/UseDNS no/' ${BASEDIR}/etc/ssh/sshd_config || true
	sed -ri 's/^[# ]*X11Forwarding.*/X11Forwarding no/' ${BASEDIR}/etc/ssh/sshd_config || true
	sed -ri 's/^[# ]*MaxAuthTries.*/MaxAuthTries 3/' ${BASEDIR}/etc/ssh/sshd_config || true

	# 8) Service tuning: disable useless or harmful in LXC
	dab exec systemctl enable --now logrotate.timer
	dab exec systemctl mask systemd-networkd-wait-online.service
	dab exec systemctl mask systemd-resolved.service 2>/dev/null || true
	dab exec systemctl disable --now e2scrub_all.timer e2scrub_reap.service 2>/dev/null || true
	dab exec systemctl disable --now motd-news.service motd-news.timer 2>/dev/null || true

	# 9) Reset hostkeys and machine-id so they regenerate at first boot
	dab exec rm -f /etc/ssh/ssh_host_*
	dab exec sh -lc ':> /etc/machine-id; rm -f /var/lib/dbus/machine-id'

	# 10) Cleanup before finalize (reduce size)
	dab exec apt-get -y autoremove --purge
	dab exec apt-get clean
	dab exec rm -rf /var/lib/apt/lists/* /var/cache/apt/* /var/cache/debconf/*-old
	dab exec rm -rf /usr/share/locale/* /usr/share/doc/* /usr/share/man/* || true

	# 11) Finalize appliance (zstd requires Proxmox >= 7.0)
	dab finalize --compressor zstd-max

info/init_ok: dab.conf
	dab init
	touch $@

.PHONY: clean
clean:
	dab clean
	rm -f *~

.PHONY: dist-clean
dist-clean:
	dab dist-clean
	rm -f *~

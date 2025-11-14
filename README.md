# NRPE 4.1.0 on Alpine Linux – Complete Installation Guide

This repository provides a clean, production-ready tutorial for compiling and configuring **NRPE 4.1.0** on **Alpine Linux**, including recommended plugins and a lightweight `check_mem` implementation that works without Perl.

All sensitive data, IPs, hostnames and internal references were removed. This material is safe for public use.

---

## 📌 Overview

NRPE allows Nagios/OMD/Checkmk monitoring servers to execute commands on remote Alpine Linux hosts securely.

This guide covers:

* Building NRPE 4.1.0 from source
* Installing official monitoring plugins
* Creating NRPE user and directories
* Deploying a lightweight `check_mem` plugin
* Creating commands inside `nrpe.cfg`
* Running and testing NRPE

---

## 📦 Requirements

```
Alpine Linux 3.17 or newer
Root access
Nagios / OMD / Checkmk server (remote)
```

---

## 🛠️ Install Required Packages

```sh
apk update
apk add build-base openssl-dev openssl wget tar monitoring-plugins
```

> Perl is optional. This tutorial includes a `check_mem` plugin that does **not** require Perl.

---

## 👤 Create `nagios` User and Group

NRPE daemons should never run as root.

```sh
addgroup -S nagios
adduser  -S -G nagios -h /var/lib/nagios -s /sbin/nologin nagios
```

---

## 🔽 Download & Compile NRPE 4.1.0

```sh
cd /root
wget https://github.com/NagiosEnterprises/nrpe/archive/refs/tags/nrpe-4.1.0.tar.gz
tar xf nrpe-4.1.0.tar.gz
cd nrpe-nrpe-4.1.0/

./configure --disable-ssl2 --disable-ssl3
make all
make install
make install-config
make install-daemon
make install-plugin
```

NRPE will be installed to:

```
/usr/local/nagios/bin/nrpe
/usr/local/nagios/libexec/
```

---

## 📂 Permissions & Plugin Directory

```sh
mkdir -p /usr/local/nagios/libexec
cp -a /usr/lib/monitoring-plugins/* /usr/local/nagios/libexec/
chown -R nagios:nagios /usr/local/nagios
```

---

## 📝 Create NRPE Configuration

```sh
mkdir -p /etc/nagios
cp /root/nrpe-nrpe-4.1.0/sample-config/nrpe.cfg /etc/nagios/nrpe.cfg
```

Modify essential fields:

```sh
sed -i 's|^pid_file=.*|pid_file=/run/nrpe.pid|' /etc/nagios/nrpe.cfg
```

Allow your monitoring server IP:

```sh
sed -i '/^allowed_hosts=/d' /etc/nagios/nrpe.cfg
echo 'allowed_hosts=<MONITORING_SERVER_IP>,127.0.0.1,::1' >> /etc/nagios/nrpe.cfg
```

Replace `<MONITORING_SERVER_IP>` with your Nagios/OMD/Checkmk server.

---

## 🧩 Add NRPE Commands

Append to `/etc/nagios/nrpe.cfg`:

```cfg
command[check_load]=/usr/local/nagios/libexec/check_load -w 1.5,1.5,1.5 -c 2.0,2.0,2.0
command[check_disk]=/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /
command[check_swap]=/usr/local/nagios/libexec/check_swap -w 90% -c 80%
command[check_total_procs]=/usr/local/nagios/libexec/check_procs -w 150 -c 200
command[check_zombie_procs]=/usr/local/nagios/libexec/check_procs -w 5 -c 10 -s Z
command[check_mem]=/usr/local/nagios/libexec/check_mem.pl -u -w 90 -c 95
```

---

## 🧠 Lightweight `check_mem` (Shell Version)

This version works on Alpine **without Perl**, using only `/proc/meminfo`.

Create the file:

```sh
cat > /usr/local/nagios/libexec/check_mem.pl <<'EOF'
#!/bin/sh
warn=90
crit=95
while getopts "w:c:u" opt; do
  case "$opt" in
    w) warn="$OPTARG" ;;
    c) crit="$OPTARG" ;;
  esac
done
awk '/^MemTotal:/ {mt=$2} /^MemAvailable:/ {ma=$2} END {
  used=mt-ma
  usedp=(used/mt)*100
  printf("%.0f %.0f %d %d\n", used/1024, usedp, mt/1024, ma/1024)
}' /proc/meminfo | {
  read used_mb used_pct total_mb avail_mb
  if [ "$used_pct" -ge "$crit" ]; then echo "CRITICAL - ${used_pct}% used (${used_mb}MB)"; exit 2; fi
  if [ "$used_pct" -ge "$warn" ]; then echo "WARNING - ${used_pct}% used (${used_mb}MB)"; exit 1; fi
  echo "OK - ${used_pct}% used (${used_mb}MB)"; exit 0
}
EOF
```

Give permissions:

```sh
chmod +x /usr/local/nagios/libexec/check_mem.pl
chown nagios:nagios /usr/local/nagios/libexec/check_mem.pl
```

---

## ▶️ Start NRPE

```sh
pkill -f nrpe
/usr/local/nagios/bin/nrpe --config=/etc/nagios/nrpe.cfg --daemon
```

Check:

```sh
ps aux | grep nrpe
netstat -plnt | grep 5666
```

---

## 🧪 Local Tests

```sh
/usr/local/nagios/libexec/check_load
/usr/local/nagios/libexec/check_mem.pl -u -w 90 -c 95
```

---

## 🧪 Remote Tests (Run from Nagios/OMD/Checkmk)

```sh
check_nrpe -H <SERVER_IP> -c check_load
check_nrpe -H <SERVER_IP> -c check_mem
check_nrpe -H <SERVER_IP> -c check_disk
```

---

## 📚 License

MIT – Free to use, modify, and improve.

---

## 🤝 Contributing

Feel free to open PRs with improvements, bugfixes, plugins, or extra examples.

---

## ⭐ If this helped you

Leave a star on the repository when published! 🌟

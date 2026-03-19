# Скрипт `push_install` — пошаговое описание команд

```bash
dnf -y update
```
- Обновляет все пакеты в системе до последних версий (`-y` автоматически отвечает “yes”).

```bash
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```
- Временно переводит SELinux в `Permissive` и постоянно отключает его в конфиге.

```bash
dnf -y install redis nodejs npm wget curl
```
- Ставит Redis, Node.js, npm и утилиты `wget`/`curl`, необходимые для push‑сервера.

```bash
mkdir -p /etc/redis /var/log/redis /var/run/redis
chown -R redis:redis /etc/redis /var/log/redis /var/run/redis
```
- Создаёт каталоги конфигурации, логов и runtime для Redis и назначает владельца `redis`.

```bash
cat > /etc/redis/redis.conf <<'EOF'
...
EOF
```
- Перезаписывает `redis.conf` готовым конфигом: сокет, память, политика вытеснения, RDB/AOF и пр.

```bash
grep -q '^vm.overcommit_memory = 1$' /etc/sysctl.conf || echo 'vm.overcommit_memory = 1' >> /etc/sysctl.conf
sysctl -w vm.overcommit_memory=1
```
- Включает `vm.overcommit_memory=1` (рекомендация Redis) и применяет параметр сразу.

```bash
mkdir -p /etc/systemd/system/redis.service.d
cat > /etc/systemd/system/redis.service.d/custom.conf <<'EOF'
[Service]
Group=www-data
RuntimeDirectory=redis
RuntimeDirectoryMode=0775
PIDFile=/var/run/redis/redis-server.pid
EOF
```
- Добавляет override для сервиса Redis: группа `www-data`, свои каталоги и PID‑файл.

```bash
groupadd -f www-data
systemctl daemon-reload
systemctl enable --now redis
systemctl restart redis
```
- Создаёт группу `www-data` (если нет), перегружает unit’ы и включает Redis в автозагрузку.

```bash
id bitrix >/dev/null 2>&1 || useradd -r -g www-data -s /sbin/nologin bitrix
```
- Создаёт системного пользователя `bitrix` в группе `www-data`, если его ещё нет.

```bash
cd /opt
wget -q https://repo.bitrix.info/vm/push-server-0.4.0.tgz
npm install --production ./push-server-0.4.0.tgz
rm -f ./push-server-0.4.0.tgz
```
- Скачивает архив push‑сервера, устанавливает его через npm в режиме `production` и удаляет архив.

```bash
ln -sf /opt/node_modules/push-server/etc/push-server /etc/push-server
ln -sf /opt/node_modules/push-server /opt/push-server
```
- Создаёт удобные symlink’и для конфигов и каталога push‑сервера.

```bash
cp /opt/node_modules/push-server/etc/init.d/push-server-multi /usr/local/bin/push-server-multi
chmod +x /usr/local/bin/push-server-multi
```
- Копирует управляющий скрипт `push-server-multi` и делает его исполняемым.

```bash
mkdir -p /etc/sysconfig
cp /opt/node_modules/push-server/etc/sysconfig/push-server-multi /etc/sysconfig/push-server-multi
cp /opt/node_modules/push-server/etc/push-server/push-server.service /etc/systemd/system/push-server.service
```
- Копирует базовый sysconfig и unit‑файл сервиса push‑сервера.

```bash
cryptokey=$(openssl rand -hex 16)
```
- Генерирует случайный 16‑байтный hex‑ключ для `SECURITY_KEY`.

```bash
cat >> /etc/sysconfig/push-server-multi <<EOF
GROUP=www-data
SECURITY_KEY="${cryptokey}"
RUN_DIR=/tmp/push-server
REDIS_SOCK=/var/run/redis/redis.sock
WS_HOST=127.0.0.1
EOF
```
- Добавляет в sysconfig push‑сервера группу, секретный ключ, пути и адрес WebSocket‑хоста.

```bash
/usr/local/bin/push-server-multi configs pub
/usr/local/bin/push-server-multi configs sub
```
- Генерирует конфиги publishers/subscribers для push‑сервера по текущим настройкам.

```bash
cat > /etc/tmpfiles.d/push-server.conf <<'EOF'
d /tmp/push-server 0770 bitrix www-data -
EOF
systemd-tmpfiles --create
```
- Создаёт правило `tmpfiles.d` и директорию `/tmp/push-server` с нужными правами и владельцами.

```bash
mkdir -p /var/log/push-server
chown -R bitrix:www-data /var/log/push-server
```
- Создаёт каталог логов push‑сервера и назначает владельца `bitrix:www-data`.

```bash
sed -i \
  -e 's|^User=.*|User=bitrix|' \
  -e 's|^Group=.*|Group=www-data|' \
  -e 's|^ExecStart=.*|ExecStart=/usr/local/bin/push-server-multi systemd_start|' \
  -e 's|^ExecStop=.*|ExecStop=/usr/local/bin/push-server-multi stop|' \
  /etc/systemd/system/push-server.service
```
- Правит unit‑файл сервиса, чтобы:
  - запускаться от `bitrix:www-data`,
  - использовать `push-server-multi` для старта/остановки.

```bash
systemctl daemon-reload
systemctl enable --now push-server
systemctl restart push-server
```
- Перечитывает unit’ы `systemd`, включает push‑сервер в автозагрузку и перезапускает его.

```

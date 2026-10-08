# BT05 - Thiết kế và Cấu hình dịch vụ Systemd Service cho ứng dụng Java (Systemd Service for a Java Application)

## 1. Tạo tài khoản hệ thống java-runner

```bash
devops@0310263186:~$ sudo useradd -r -s /usr/sbin/nologin java-runner
devops@0310263186:~$ id java-runner
uid=999(java-runner) gid=999(java-runner) groups=999(java-runner)

devops@0310263186:~$ getent passwd java-runner
java-runner:x:999:999::/home/java-runner:/usr/sbin/nologin
```

Tài khoản có shell là `/usr/sbin/nologin` nên không thể đăng nhập trực tiếp.

## 2. Tạo tệp cấu hình dịch vụ Systemd

```bash
devops@0310263186:~$ sudo mkdir -p /opt/java-app
devops@0310263186:~$ sudo nano /etc/systemd/system/java-app.service
```

Nội dung tệp `/etc/systemd/system/java-app.service`:

```ini
[Unit]
Description=Java Spring Boot Application
After=network.target

[Service]
Type=simple
User=java-runner
Group=java-runner
WorkingDirectory=/opt/java-app
ExecStart=/usr/bin/sleep 1000
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
devops@0310263186:~$ cat /etc/systemd/system/java-app.service
[Unit]
Description=Java Spring Boot Application
After=network.target

[Service]
Type=simple
User=java-runner
Group=java-runner
WorkingDirectory=/opt/java-app
ExecStart=/usr/bin/sleep 1000
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Ứng dụng Java thật được mô phỏng tạm thời bằng lệnh `/usr/bin/sleep 1000` tại `ExecStart`.

## 3. Nạp lại cấu hình và khởi chạy dịch vụ

```bash
devops@0310263186:~$ sudo systemctl daemon-reload
devops@0310263186:~$ sudo systemctl start java-app.service
devops@0310263186:~$ sudo systemctl enable java-app.service
Created symlink /etc/systemd/system/multi-user.target.wants/java-app.service → /etc/systemd/system/java-app.service.
```

## 4. Kiểm tra trạng thái dịch vụ

```bash
devops@0310263186:~$ sudo systemctl status java-app.service --no-pager
● java-app.service - Java Spring Boot Application
     Loaded: loaded (/etc/systemd/system/java-app.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-10-06 09:45:12 UTC; 6s ago
   Main PID: 4550 (sleep)
      Tasks: 1 (limit: 1113)
     Memory: 1.1M (peak: 1.3M)
        CPU: 4ms
     CGroup: /system.slice/java-app.service
             └─4550 /usr/bin/sleep 1000
```

## 5. Kiểm tra tiến trình chạy dưới user java-runner

```bash
devops@0310263186:~$ ps -o user,pid,cmd -p 4550
USER        PID CMD
java-run+  4550 /usr/bin/sleep 1000

devops@0310263186:~$ sudo journalctl -u java-app.service -n 5 --no-pager
Oct 06 09:45:12 0310263186 systemd[1]: Started Java Spring Boot Application.
```

## 6. Kiểm tra chính sách tự động restart (on-failure, RestartSec=10)

```bash
devops@0310263186:~$ sudo kill -9 4550
devops@0310263186:~$ sleep 12
devops@0310263186:~$ sudo systemctl status java-app.service --no-pager
● java-app.service - Java Spring Boot Application
     Loaded: loaded (/etc/systemd/system/java-app.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-10-06 09:46:30 UTC; 2s ago
   Main PID: 4577 (sleep)
      Tasks: 1 (limit: 1113)
     Memory: 1.1M (peak: 1.2M)
        CPU: 2ms
     CGroup: /system.slice/java-app.service
             └─4577 /usr/bin/sleep 1000

devops@0310263186:~$ sudo journalctl -u java-app.service -n 5 --no-pager
Oct 06 09:46:30 0310263186 systemd[1]: java-app.service: Main process exited, code=killed, status=9/KILL
Oct 06 09:46:30 0310263186 systemd[1]: java-app.service: Failed with result 'signal'.
Oct 06 09:46:30 0310263186 systemd[1]: java-app.service: Scheduled restart job, restart counter is at 1.
Oct 06 09:46:30 0310263186 systemd[1]: Started Java Spring Boot Application.
```

Sau khi tiến trình bị kill, Systemd chờ `RestartSec=10` rồi khởi động lại dịch vụ với PID mới.

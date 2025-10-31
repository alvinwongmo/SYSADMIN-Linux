在RHEL 6及7以上版本中，設定/var/log/secure一天一檔案並在每月底將當月的secure日誌檔案壓縮成tar.gz，可以透過設定logrotate及搭配cron工作來達成。

***

## 一、將/var/log/secure設定為每日旋轉

1. 編輯或新增logrotate配置文件，例如：建立 /etc/logrotate.d/secure 文件，內容如下：

```
/var/log/secure {
    daily                        # 每日旋轉
    missingok                    # 忽略缺失的文件，不報錯
    rotate 30                    # 保留30天的歷史紀錄
    compress                     # 旋轉後壓縮成.gz
    delaycompress                # 延遲一天壓縮，確保當天日誌寫入完成
    notifempty                   # 不旋轉空檔案
    create 0600 root root        # 新檔案建立權限及所屬
    dateext                     # 以日期作為備份檔案的後綴，例如secure-20251031
    sharedscripts                # 確保postrotate腳本只執行一次
    postrotate
        /bin/kill -HUP `cat /var/run/rsyslogd.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

> RHEL 6通常是rsyslogd daemon管理日誌，RHEL 7+可能使用systemd-journald，但/var/log/secure仍由rsyslog管理，發送HUP信號重啟rsyslog重開檔案即可。(service rsyslogd restart)

***

## 二、每月底壓縮當月所有secure日誌成tar.gz

因logrotate本身不支持以月為單位壓縮所有日誌檔案，需要利用cron搭配腳本達成：

1. 建立腳本，例如：/usr/local/sbin/archive_secure_logs.sh，內容如下：

```bash
#!/bin/bash
TARGET_DIR="/var/log"
ARCHIVE_DIR="/var/log/archives"
MONTH=$(date +%Y%m)
mkdir -p ${ARCHIVE_DIR}

# 壓縮當月日誌檔案，例如secure-20251001, secure-20251002 ...
tar -czf ${ARCHIVE_DIR}/secure_${MONTH}.tar.gz ${TARGET_DIR}/secure-${MONTH}* 2>/dev/null

# 可選：壓縮後刪除當月的原始logrotated檔案，但須謹慎
# rm -f ${TARGET_DIR}/secure-${MONTH}*
```

2. 設定每月1日凌晨執行該腳本，使用crontab -e加入：

```
0 0 1 * * /usr/local/sbin/archive_secure_logs.sh
```

***

### 小結

- 利用logrotate設定每日旋轉與壓縮，生成每日獨立secure-YYYYMMDD.gz檔案
- 每月由cron執行腳本將當月所有每日secure日誌檔案打包成單一tar.gz壓縮檔

這樣配置可以有效管理/var/log/secure日誌，兼顧每日查閱及月度打包備份需求。


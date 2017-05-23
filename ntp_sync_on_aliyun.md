阿里云服务器时间矫正
==========

#### 步骤命令
```bash
date

# stop the ntpd service
sudo systemctl stop ntpd

# synchronize the time
sudo ntpdate 10.143.33.50

# restart the ntp service
sudo systemctl start ntpd

# check again
date
```

#### 参考资料
- [内网和公共NTP服务器](https://help.aliyun.com/knowledge_detail/40583.html)

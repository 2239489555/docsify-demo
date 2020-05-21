### 安装最小的CentOS7使用yum报错



```shell
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
There are no enabled repos.
 Run "yum repolist all" to see the repos you have.
 To enable Red Hat Subscription Management repositories:
     subscription-manager repos --enable <repo>
 To enable custom repositories:
     yum-config-manager --enable <repo>

```

### 解决方式

```shell
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

### 原因

```text
缺少文件，从阿里云下载更新缺失的文件
```

### 安装Ansible

```shell
rpm -Uvh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm

yum install ansible -y
```


# Google Authenticator PAM module

Example PAM module demonstrating two-factor authentication.

[![Build Status](https://travis-ci.org/google/google-authenticator-libpam.svg?branch=master)](https://travis-ci.org/google/google-authenticator-libpam)

## Build & install
```shell
./bootstrap.sh
./configure
make
sudo make install
```

If you don't have access to "sudo", you have to manually become "root" prior
to calling "make install".

## Setting up the PAM module for your system

For highest security, make sure that both password and OTP are being requested
even if password and/or OTP are incorrect. This means that *at least* the first
of `pam_unix.so` (or whatever other module is used to verify passwords) and
`pam_google_authenticator.so` should be set as `required`, not `requisite`. It
probably can't hurt to have both be `required`, but it could depend on the rest
of your PAM config.

If you use HOTP (counter based as opposed to time based) then add the option
`no_increment_hotp` to make sure the counter isn't incremented for failed
attempts.

Add this line to your PAM configuration file:

`  auth required pam_google_authenticator.so no_increment_hotp`

## Setting up a user

Run the `google-authenticator` binary to create a new secret key in your home
directory. These settings will be stored in `~/.google_authenticator`.

If your system supports the "libqrencode" library, you will be shown a QRCode
that you can scan using the Android "Google Authenticator" application.

If your system does not have this library, you can either follow the URL that
`google-authenticator` outputs, or you have to manually enter the alphanumeric
secret key into the Android "Google Authenticator" application.

In either case, after you have added the key, click-and-hold until the context
menu shows. Then check that the key's verification value matches (this feature
might not be available in all builds of the Android application).

Each time you log into your system, you will now be prompted for your TOTP code
(time based one-time-password) or HOTP (counter-based), depending on options
given to `google-authenticator`, after having entered your normal user id and
your normal UNIX account password.

During the initial roll-out process, you might find that not all users have
created a secret key yet. If you would still like them to be able to log
in, you can pass the "nullok" option on the module's command line:

`  auth required pam_google_authenticator.so nullok`

## Encrypted home directories

If your system encrypts home directories until after your users entered their
password, you either have to re-arrange the entries in the PAM configuration
file to decrypt the home directory prior to asking for the OTP code, or
you have to store the secret file in a non-standard location:

`  auth required pam_google_authenticator.so secret=/var/unencrypted-home/${USER}/.google_authenticator`

would be a possible choice. Make sure to set appropriate permissions. You also
have to tell your users to manually move their .google_authenticator file to
this location.

In addition to "${USER}", the `secret=` option also recognizes both "~" and
`${HOME}` as short-hands for the user's home directory.

When using the `secret=` option, you might want to also set the `user=`
option. The latter forces the PAM module to switch to a dedicated hard-coded
user id prior to doing any file operations. When using the `user=` option, you
must not include "~" or "${HOME}" in the filename.

The `user=` option can also be useful if you want to authenticate users who do
not have traditional UNIX accounts on your system.

## Module options

### secret=/path/to/secret/file

See "encrypted home directories", above.

### authtok_prompt=prompt

Overrides default token prompt. If you want to include spaces in the prompt,
wrap the whole argument in square brackets:

`  auth required pam_google_authenticator.so [authtok_prompt=Your secret token: ]`

### user=some-user

Force the PAM module to switch to a hard-coded user id prior to doing any file
operations. Commonly used with `secret=`.

### no_strict_owner

DANGEROUS OPTION!

By default the PAM module requires that the secrets file must be owned the user
logging in (or if `user=` is specified, owned by that user). This option
disables that check.

This option can be used to allow daemons not running as root to still handle
configuration files not owned by that user, for example owned by the users
themselves.

### allowed_perm=0nnn

DANGEROUS OPTION!

By default, the PAM module requires the secrets file to be readable only by the
owner of the file (mode 0600 by default). In situations where the module is used
in a non-default configuration, an administrator may need more lenient file
permissions, or a specific setting for their use case.

### debug

Enable more verbose log messages in syslog.

### try_first_pass / use_first_pass / forward_pass

Some PAM clients cannot prompt the user for more than just the password. To
work around this problem, this PAM module supports stacking. If you pass the
`forward_pass` option, the `pam_google_authenticator` module queries the user
for both the system password and the verification code in a single prompt.
It then forwards the system password to the next PAM module, which will have
to be configured with the `use_first_pass` option.

In turn, `pam_google_authenticator` module also supports both the standard
`use_first_pass` and `try_first_pass` options. But most users would not need
to set those on the `pam_google_authenticator`.

### noskewadj

If you discover that your TOTP code never works, this is most commonly the
result of the clock on your server being different from the one on your Android
device. The PAM module makes an attempt to compensate for time skew. You can
teach it about the amount of skew that you are experiencing, by trying to log
it three times in a row. Make sure you always wait 30s (but not longer), so
that you get three distinct TOTP codes.

Some administrators prefer that time skew isn't adjusted automatically, as
doing so results in a slightly less secure system configuration. If you want
to disable it, you can do so on the module command line:

`  auth required pam_google_authenticator.so noskewadj`

### no_increment_hotp

Don't increment the counter for failed HOTP attempts. This is important if log
attempts with failed passwords still get an OTP prompt.

### nullok

Allow users to log in without OTP, if they haven't set up OTP yet.

### echo_verification_code

By default, the PAM module does not echo the verification code when it is
entered by the user. In some situations, the administrator might prefer a
different behavior. Pass the `echo_verification_code` option to the module
in order to enable echoing.

If you would like verification codes that are counter based instead of
timebased, use the `google-authenticator` binary to generate a secret key in
your home directory with the proper option.  In this mode, clock skew is
irrelevant and the window size option now applies to how many codes beyond the
current one that would be accepted, to reduce synchronization problems.

### 中文版本
Google Authentication 项目 包含了多个手机平台的一次性验证码生成器的实现，以及一个可插拔的验证认证模块（PAM）。这些实现支持基于 HMAC 的一次性验证码（HOTP）算法（RFC 4226）和基于时间的一次性验证码（TOTP）算法（RFC 6238）
下面将在 CentOS 上安装并使用 Google Authenticator 做登录的身份验证，当前系统的版本为
CentOS Linux release 7.2.1511 (Core)

### 安装 Google Authenticator PAM module

1.确保 ntpd 已安装并正常运行运行
yum install -y ntpdate
systemctl start ntpd
systemctl enable ntpd
ntpdate 是用来自动同步时间的程序，这里启动它并设置它开机自动启动。

2.安装一些接下去会用到的组件
yum install -y git make gcc libtool pam-devel

3.编译安装 Google Authenticator PAM module
git clone https://github.com/dingbb/google-authenticator-libpam
cd google-authenticator-libpam
./bootstrap.sh
./configure
make
make install
ln -s /usr/local/lib/security/pam_google_authenticator.so /usr/lib64/security/

### 配置 SSH 服务
打开 /etc/ssh/sshd_config 文件

vim /etc/ssh/sshd_config
修改下面字段的配置

ChallengeResponseAuthentication yes
PasswordAuthentication no
PubkeyAuthentication yes
UsePAM yes
然后重启一下 sshd 服务，使配置生效

systemctl restart sshd
这里将 PubkeyAuthentication 配置成了 yes 表示支持公钥验证登录，即使某个账号启用了 Google Authenticator 验证，只要登录者机器的公钥在这个账号的授权下，就可以不输入密码和 Google Authenticator 的认证码直接登录。

### 配置 PAM 
打开 /etc/pam.d/sshd 文件

vim /etc/pam.d/sshd
这里分四种情况来配置

验证密码和认证码，没有启用 Google Authenticator 服务的账号只验证密码（推荐）

auth substack password-auth
#...
auth required pam_google_authenticator.so nullok
password-auth 与 pam_google_authenticator 的先后顺序决定了先输入密码还是先输入认证码。

验证密码和认证码，没有启用 Google Authenticator 服务的账号无法使用密码登录

auth substack password-auth
#...
auth required pam_google_authenticator.so
只验证认证码，不验证密码，没有启用 Google Authenticator 服务的账号不用输入密码直接可以成功登录

#auth substack password-auth
#...
auth required pam_google_authenticator.so nullok
注释掉 auth substack password-auth 配置就不会再验证账号密码了。

只验证认证码，不验证密码，没有启用 Google Authenticator 服务的账号无法使用密码登录

#auth substack password-auth
#...
auth required pam_google_authenticator.so

### 启用 Google Authenticator
切换至想要使用 Google Authenticator 来做登录验证的账号，执行下面命令
google-authenticator
然后会出现一个二维码 #二维码跟手机客户端软件做认证用
或者手动输入后面的密钥（secret key）来代替扫描二维码，之后的操作会用到这个二维码/密钥（secret key）。这里还有一个认证码（verifiction code），暂时不知道有什么用，以及 5 个紧急救助码（emergency scratch code），紧急救助码就是当你无法获取认证码时（比如手机丢了），可以当做认证码来用，每用一个少一个，但其实可以手动添加的，建议如果 root 账户使用 Google Authenticator 的话一定要把紧急救助码另外保存一份。


---
layout: post
title: "WSL Ubuntu 20.04 LTS에서 letsencrypt-auto 오류"
img: "letsencryption.png"
date: 2020-10-09 12:00:00 +0900
tags: [letsencrypt, HTTPS, 인증서, ubuntu, wsl] 
related: letsencrypt 
categories: tools
---

[letsencrypt 인증서 사용하기]({{site.baseurl}} {% link _posts/2018-10-10-letsencryption.markdown.md %})에서 "letsencrypt-auto" 스크립트를 이용하여 인증서만 발급하는 과정을 확인할 수 있다.  
아파치나 nginx 웹서버를 사용하면 "certbot-auto"를 이용하여 인증서 발급과정을 자동화 할 수 있는데 지금은 임시로 인증서만 발급이 필요한 상황이다. 

윈도우즈를 재설치 하면서 WSL을 Ubuntu 20.04로 설치한 후 다시 동일한 과정으로 인증서를 발급하려 하는데 기존 Ubuntu 18.04에서는 문제 없이 동작하던 스크립트가 파이썬 관련 에러가 발생하여 확인한 내용을 기술한다. 

발생원인을 정확히 파악하지 않고 스크립트 오류 부분만 발생하지 않도록 수정해 보기로 한다. 

### 파이썬 virtualenv 실행 관련 오류 

아래와 같이 letsencrypt-auto를 실행하면 ..  
제일 마지막에 보면 "python-virtualenv" 패키지를 찾을 수 없다는 오류가 발생한다. 

```bash
butteryoon@YOON-IP700:~/letsencrypt$ ./letsencrypt-auto
Requesting to rerun ./letsencrypt-auto with root privileges...
Bootstrapping dependencies for Debian-based OSes... (you can skip this with --no-bootstrap)
Get:1 http://security.ubuntu.com/ubuntu focal-security InRelease [107 kB]
Ign:2 http://ppa.launchpad.net/brightbox/ruby-ng/ubuntu focal InRelease
Hit:3 http://archive.ubuntu.com/ubuntu focal InRelease
Hit:4 https://download.jitsi.org stable/ InRelease
Get:5 http://archive.ubuntu.com/ubuntu focal-updates InRelease [111 kB]
Err:6 http://ppa.launchpad.net/brightbox/ruby-ng/ubuntu focal Release
  404  Not Found [IP: 91.189.95.83 80]
Get:7 http://archive.ubuntu.com/ubuntu focal-backports InRelease [98.3 kB]
Reading package lists... Done
E: The repository 'http://ppa.launchpad.net/brightbox/ruby-ng/ubuntu focal Release' does not have a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
apt-get update hit problems but continuing anyway...
Reading package lists... Done
Building dependency tree
Reading state information... Done
Note, selecting 'python-is-python2' instead of 'python'
Note, selecting 'python-dev-is-python2' instead of 'python-dev'
Package python-virtualenv is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

E: Package 'python-virtualenv' has no installation candidate
```

python-virtualenv 패키지는 아래와 같이 "no packages found" 상태라 그냥 virtualenv를 설치한다.  
설치 후 다시 실행하면 동일한 오류가 발생한다. 

```bash
butteryoon@YOON-IP700:/etc/letsencrypt$ apt-cache show python-virtualenv
N: Can't select versions from package 'python-virtualenv' as it is purely virtual
N: No packages found
``` 


letsencrypt-auto 스크립트를 열어 virtualenv 설정부분을 보니 뭔가 이상하다.  
무조건 347라인의 python-virtualenv가 설정되는 거 같아 346 ~ 348 라인은 주석 처리하고 다시 진행한다. 

```bash
diff --git a/letsencrypt-auto b/letsencrypt-auto
index 2a0cda9b3..4834ceb03 100755
--- a/letsencrypt-auto
+++ b/letsencrypt-auto
@@ -343,9 +343,9 @@ BootstrapDebCommon() {
     fi
   fi

-  if apt-cache show python-virtualenv > /dev/null 2>&1; then
-    virtualenv="$virtualenv python-virtualenv"
-  fi
+#  if apt-cache show python-virtualenv > /dev/null 2>&1; then
+#    virtualenv="$virtualenv python-virtualenv"
+#  fi

   augeas_pkg="libaugeas0 augeas-lenses"
```

이번에는 아래와 같이 virtualenv 파라미터 중 "--no-site-packages" 에서 오류가 난다 

```bash
butteryoon@YOON-IP700:~/letsencrypt$ sudo ./letsencrypt-auto certonly --manual --no-bootstrap
./letsencrypt-auto has insecure permissions!
To learn how to fix them, visit https://community.letsencrypt.org/t/certbot-auto-deployment-best-practices/91979/
Creating virtual environment...
usage: virtualenv [--version] [--with-traceback] [-v | -q] [--app-data APP_DATA] [--clear-app-data] [--discovery {builtin}] [-p py] [--creator {builtin,cpython2-posix}] [--seeder {app-data,pip}] [--no-seed] [--activators comma_sep_list]
 . . . 생략 . . .
virtualenv: error: unrecognized arguments: --no-site-packages
Traceback (most recent call last):
  File "<stdin>", line 27, in <module>
  File "<stdin>", line 19, in create_venv
  File "/usr/lib/python2.7/subprocess.py", line 190, in check_call
    raise CalledProcessError(retcode, cmd)
subprocess.CalledProcessError: Command '['virtualenv', '--no-site-packages', '--python', '/usr/bin/python2.7', '/opt/eff.org/certbot/venv']' returned non-zero exit status 2
```

검색해보니 virtualenv 파라미터에서 "--no-site-packages" 파라미터는 필요 없는거 같아 삭제한다. 

* [How To Install Python 2 with Virtualenv  on Ubuntu 20.04](https://computingforgeeks.com/how-to-install-python2-with-virtualenv-on-ubuntu/) 

```bash
@@ -1011,7 +1011,7 @@ def create_venv(venv_path, pyver, verbose):
         # Use virtualenv binary
         environ = os.environ.copy()
         environ['VIRTUALENV_NO_DOWNLOAD'] = '1'
-        command = ['virtualenv', '--no-site-packages', '--python', sys.executable, venv_path]
+        command = ['virtualenv', '--python', sys.executable, venv_path]
         subprocess.check_call(command, stdout=stdout, env=environ)
     else:
         # Use embedded venv module in Python 3
```

이렇게 두 부분을 수정 한 후 기존과 동일하게 인증서 발급 진행 완료 .. 

Ubuntu 20.04 Python 환경이 3.x 기준으로 변경이 되어서 뭔가 기존 스크립트가 맞지 않는거 같은데 Python 3.x 환경에서 다시 한번 시도해 봐야 겠다. 


## 참고  

- [How To Install Python 2 with Virtualenv  on Ubuntu 20.04](https://computingforgeeks.com/how-to-install-python2-with-virtualenv-on-ubuntu/)   
- [python3 pip install](https://linuxhint.com/install_python_pip_tool_ubuntu/)   
- [Setup Let’s Encrypt Wildcard SSL on Ubuntu 20.04-18.04](https://websiteforstudents.com/setup-lets-encrypt-wildcard-on-ubuntu-20-04-18-04/)   
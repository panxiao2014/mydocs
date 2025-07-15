今天在Ubuntu上执行git命令，总是会失败并提示以下错误：

>fatal: unable to access 'https://github.com/xxx/xxx.git':
>
>gnutls_handshake() failed: The TLS connection was non-properly terminated.
>

搜索网页和LLM，提供了一些解决方法，但是都不奏效。突然想起昨天自己在机器上倒腾ssl相关的东西。观察以下文件：

/etc/ssl/certs/ca-certificates.crt

发现文件在昨天被修改过了，但是我自己搞不清楚是怎么修改的。于是采用以下方法更新了该文件：

>apt-get --reinstall install ca-certificates
>
>update-ca-certificates --fresh
>
问题解决。

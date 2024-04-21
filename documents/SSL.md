# Security Sockets Layers(SSL)
SSL英文全称是“Secure Sockets Layer”，中文含义为“安全套接层”，及其继任者传输层安全（Transport Layer Security，TLS）是为网络通信提供安全及数据完整性的一种安全协议。 TLS与SSL在传输层对网络连接进行加密，保障数据传输安全。

# 基本概念

Certificate Authority: 一个第三方机构，负责颁发证书(CA)。比如微软，google或Mozilla。  

SSL系统：有些假的第三方机构也可以颁发假的证书。SSL系统负责认证证书的真伪。当Chrome等浏览器打开伪证书认证的网站时，会提示不安全。  

Certificate: 如果新创建网站，可以去VeriSign等公司购买证书。办好的证书是一个私钥(.key)和一个证书(.cer)文件。  

PEM文件：很多.key和.cer文件都是pem文件格式的。因此看到一个.pem文件时，它可能是私钥，也可能是证书，也可能是两者都具备。  

# 私钥和公钥的运作原理





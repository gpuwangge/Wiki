# Security Sockets Layers(SSL)
SSL英文全称是“Secure Sockets Layer”，中文含义为“安全套接层”，及其继任者传输层安全（Transport Layer Security，TLS）是为网络通信提供安全及数据完整性的一种安全协议。 TLS与SSL在传输层对网络连接进行加密，保障数据传输安全。

# 基本概念

Certificate Authority: 一个第三方机构，负责颁发证书(CA)。比如微软，google或Mozilla。  

SSL系统：有些假的第三方机构也可以颁发假的证书。SSL系统负责认证证书的真伪。当Chrome等浏览器打开伪证书认证的网站时，会提示不安全。  

Certificate: 如果新创建网站，可以去VeriSign等公司购买证书。办好的证书是一个私钥(.key)和一个证书(.cer)文件。每个证书都有一个数字签名。使用SSL系统的网站URL开头看起来是这样：https//，而不是http//  

PEM文件：很多.key和.cer文件都是pem文件格式的。因此看到一个.pem文件时，它可能是私钥，也可能是证书，也可能是两者都具备。用户通过持有.pem文件来验证服务器的身份。.pem文件本身的来源必须可靠。  




# 非对称密钥系统
## 公钥
一把可以公开的钥匙。可以用来加密，也可以解密私钥加密过的文件，也可以用来签名。 

## 私钥
一把不应该公开的钥匙。可以用来加密，也可以解密公钥加密过的文件，也可以用来签名。  

## 签名
把目标文件用Hash函数生成摘要(Digest)，然后用私钥对摘要加密，就可以生成签名(Signature)。  

## 数字签名演示场景
小A有重要的文件要传递给小B。  
黑客小C奉命取得这个文件。  
小B先准备好一对公钥(b)和私钥(b)。其中私钥(b)自己藏好，并把公钥(b)发给小A。  
小C可能会拦截得到公钥(b)，但公钥(b)即使泄露给小A以外的人也无所谓。  
假若小A使用公钥(b)加密文件(b)并传给小B，这份文件只能用小B的私钥(b)解密。因此，即使传文件过程中被黑客小C获得，小C也无法获知其文件内容。  
但是由于小C可以获得公钥(b)，因此小C可以拦截加密文件(b)，然后用公钥(b)做一份假的文件传给小B，这样小B存在被黑客小C愚弄的结果。（中间人攻击）  
小A和小B为了传递正确的文件，他们决定使用签名。  
第一阶段，小A也准备一对公钥(a)和私钥(a)。同样他把公钥(a)分享给小B。  
第二阶段，小A使用私钥(a)对文件产生签名(a)。小A把加密文件(b)和签名(a)一起发送给小B。  
当小B收到两份文件后，先使用公钥(a)解密签名，得到文件摘要。再用私钥(b)解密文件本体。文件本体必须跟文件摘要hash对接上。如果对不上，则说明此文件被黑客篡改。  
这样，小C是无法进行中间人攻击的，因为他没有私钥(a)，因此无法生成签名(a)。  


# 对称密钥系统
跟非对称密钥的区别是加密和解密时候使用相同的密钥，或是可以简单互相推算的密钥。  
对称密钥的缺点是管理困难。因为钥匙一旦发出去，就可能被黑客获得。  
除非，对称密钥的分发过程本身由另一个非堆成密钥系统保证其安全性。  
因为无法确保密钥的安全性，因此对称密钥不适合签名的场景。  

## 浏览器-服务器演示场景
这里的服务器(网站)就相当于小A，浏览器(用户)就相当于小B。服务器需要把正确的网站内容发给浏览器，而浏览器需要把这个内容呈现给用户，并保证这个内容是不被黑客篡改过的。  
服务器(网站)需要准备公钥和私钥。当用户访问网站的时候，浏览器需要获得这个公钥。并且浏览器需要查阅证书以保证公钥是正确的。  
用户不需要准备私钥和公钥，而可以使用一个随机的对称密钥。  
第一阶段，浏览器使用公钥加密对称密钥，并发给服务器。  
因为对称密钥只能由服务器的私钥解密，因此黑客无法获得对称密钥。  
第二阶段，服务器再用对称密钥加密网站内容并发给浏览器。  
最后浏览器使用对称密钥解密网站内容并呈现给用户。  
因为黑客无法获得对称密钥，因此无法伪造网站内容。  








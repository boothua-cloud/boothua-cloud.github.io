---
title: Python实现SSL自签名证书
abbrlink: f91fa873
tags:
  - Python
  - SSL
categories:
  - Python
  - SSL
index_img: /img/pages/ssl.jpg
date: 2023-08-13 17:10:45
---
### 前言
由于项目中需要用到自签名证书，每次都需要使用subprocess执行openssl来生成，感觉不是很方便，所以用python实现了基于cryptography库的x509来生成自签名ssl证书

### 功能介绍
支持自定义ip,域名，common_name，country_name，state_or_province_name，locality_name，organization_name以及有效期

下面是代码实现：
```python
import ipaddress
import os
from datetime import datetime, timedelta
from typing import List, Optional

from cryptography import x509
from cryptography.hazmat._oid import NameOID
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa


class Certificate:
    country_name: str
    common_name: str
    state_or_province_name: str
    locality_name: str
    organization_name: str
    days: int
    ip: List[str]
    dns: List[str]

    def __init__(self, common_name: Optional[str], country_name: Optional[str] = 'CN',
                 state_or_province_name: Optional[str] = 'GuiZhou', locality_name: Optional[str] = 'GuiYang',
                 organization_name: Optional[str] = 'HPC', days: Optional[int] = 365, ip: List[str] = None,
                 dns: List[str] = None):
        self.country_name = country_name
        self.common_name = common_name
        self.state_or_province_name = state_or_province_name
        self.locality_name = locality_name
        self.organization_name = organization_name
        self.days = days
        self.ip = ip or ['127.0.0.1']
        self.dns = dns or ['localhost']

    def generate_ssl_pair(self):
        """
        生成ssl证书密钥对
        """
        pri_key = rsa.generate_private_key(public_exponent=65537, key_size=2048, backend=default_backend())

        name_attrs = {
            NameOID.COUNTRY_NAME: self.country_name,
            NameOID.COMMON_NAME: self.common_name,
            NameOID.STATE_OR_PROVINCE_NAME: self.state_or_province_name,
            NameOID.LOCALITY_NAME: self.locality_name,
            NameOID.ORGANIZATION_NAME: self.organization_name,
        }
        subject = issuer = x509.Name([x509.NameAttribute(attr, value) for attr, value in name_attrs.items()])
        alt_names = []
        for dns in self.dns:
            alt_names.append(x509.DNSName(dns))
        for ip in self.ip:
            alt_names.append(x509.IPAddress(ipaddress.ip_address(ip)))
        # cert使用私钥签名（.sign(私钥，摘要生成算法，填充方式)），使用x509.CertificateBuilder()方法生成证书，证书属性使用下列函数叠加补充
        now = datetime.utcnow()
        not_valid_before = now - timedelta(seconds=1)
        not_valid_after = now + timedelta(days=self.days)

        certs = (
            x509.CertificateBuilder()
            .subject_name(subject)
            .issuer_name(issuer)
            .public_key(pri_key.public_key())
            .add_extension(x509.SubjectAlternativeName(alt_names), False)
            .serial_number(x509.random_serial_number())
            .not_valid_before(not_valid_before)
            .not_valid_after(not_valid_after)
            .sign(pri_key, hashes.SHA256(), default_backend())
        )
        # 最终生成的证书与密钥对为类对象，要保存在文件中还需要进一步转换成字节格式
        # 将证书类对象转换成PEM格式的字节串
        cert_text = certs.public_bytes(serialization.Encoding.PEM)

        # 将私钥类对象转换成PEM格式的字节串，encryption_algorithm=serialization.NoEncryption()这里是私钥不加密的意思
        private_text = pri_key.private_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PrivateFormat.TraditionalOpenSSL,
            encryption_algorithm=serialization.NoEncryption()
        )
        # 输出到文件
        local_cert = os.path.join('./certs', 'local')
        with open(local_cert, 'wb') as f:
            f.write(cert_text)
        with open(local_cert, 'wb') as f:
            f.write(private_text)
```
### 总结
1. 到此通过python生成ssl自签名证书就已经实现了，
2. 如果需要在证书中添加其他身份验证的信息修改name_attrs，在里面加上相关身份认证即可，比如说user_id,具体请信息请看NameOID
3. 其他细节还需要参考cryptography库
---
layout: post
title: "openssl 撤销证书"
description: ""
date: 2023-10-06
tags: [openssl]
---

# 问题

lua-ngix-module 的测试用例中用到了撤销证书 t/cert/test.crl。在更新 t/cert/test.key 和 t/cert/test.crt 后忘记更新撤销证书列表导致用例失败。
那么如何生成撤销证书列表呢？

# 生成撤销证书列表

生成撤销证书列表需要用到 CA 证书，但是因为这里是自签名证书，因此就 CA 就算自签名证书本身。
下面的命令列表中给出了一些使用 CA 证书的例子，这些例子存放在注释中。

```shell
cd t/cert

#Make a directory for a CRL
mkdir -p pki/pulp/content/crl

#Create an index file with the following command
touch pki/pulp/content/crl/index.txt

#Create a file for the CRL number. This file should contain the text 00 only.
echo 00 > pki/pulp/content/crl/pulp_crl_number

#In pulp/, create and write the following contents into a crl_openssl.conf file.
mkdir -p pulp
cat <<EOF > pulp/crl_openssl.conf
# OpenSSL configuration for CRL generation
#
####################################################################
[ ca ]
default_ca	= CA_default		# The default ca section

####################################################################
[ CA_default ]
database = pki/pulp/content/crl/index.txt
crlnumber = pki/pulp/content/crl/pulp_crl_number


default_days	= 3650			# how long to certify for
default_crl_days= 3000			# how long before next CRL
default_md	= default		# use public key default MD
preserve	= no			# keep passed DN ordering

####################################################################
[ crl_ext ]
# CRL extensions.
# Only issuerAltName and authorityKeyIdentifier make any sense in a CRL.
# issuerAltName=issuer:copy
authorityKeyIdentifier=keyid:always,issuer:always
EOF

#Create the CRL file with the following command
#openssl ca -gencrl -keyfile /home/example/ca.key -cert /home/example/ca.crt -out /etc/pki/pulp/content/crl/pulp_crl.pem -config /etc/pulp/crl_openssl.conf
openssl ca -gencrl -keyfile test.key -cert test.crt -out pki/pulp/content/crl/pulp_crl.pem -config pulp/crl_openssl.conf

#Revoke a certificate with the following command
#openssl ca -revoke <Content certificate> -keyfile /home/example/ca.key -cert /home/example/ca.crt -config /etc/pulp/crl_openssl.conf
openssl ca -revoke test.crt -keyfile test.key -cert test.crt -config pulp/crl_openssl.conf

# Regenerate the CRL list with the following command
#openssl ca -gencrl -keyfile /home/example/ca.key -cert /home/example/ca.crt -out /etc/pki/pulp/content/crl/pulp_crl.pem -config /etc/pulp/crl_openssl.conf
openssl ca -gencrl -keyfile test.key -cert test.crt -out test.crl -config pulp/crl_openssl.conf

#view the content of the crl
openssl crl -in test.crl  -CAfile test.crt -text
```



# 参考文档
https://access.redhat.com/documentation/zh-cn/red_hat_update_infrastructure/2.1/html/administration_guide/chap-red_hat_update_infrastructure-administration_guide-certification_revocation_list_crl 

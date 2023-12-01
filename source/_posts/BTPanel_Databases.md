---
title: 无需登录，获取宝塔面板保存的数据库密码
date: 2023-12-1 15:26:33
categories: 
- Develop
tags:
- Decrypt
---
拿到装有宝塔面板的服务器后，在不登录面板的情况下不能直接查看数据库信息

为了解决这个问题，就制作了一个脚本去进行配置信息的解密

```python
import os

#使用前pip3 install PyCryptodome

#下载/www/server/panel/data/div.pl文件，divtext为文件内容
divtext = "3UOiALw6JWPa2F02dtN3+ynzUs9oXWcT5llyhivOBaI="

#下载/www/server/panel/data/default.db，此文件为sqlite数据库文件，打开数据库中databases表，encryptedpass为表中password的值
encryptedpass = "BT-0x:tXeUcwJk+Lu5t+BTgGgLAIJbFXYxU84sZA8ak1yjvec="

def db_decrypt(data):
    try:
        key = __get_db_sgin()
        iv = __get_db_iv()
        str_arr = data.split('\n')
        res_str = ''
        for data in str_arr:
            if not data: continue
            res_str += __aes_decrypt(data, key, iv)
    except:
        res_str = data
    return res_str

def __get_db_sgin():
    keystr = '3gP7+k_7lSNg3$+Fj!PKW+6$KYgHtw#R'
    key = ''
    for i in range(31):
        if i & 1 == 0:
            key += keystr[i]
    return key

def __get_db_iv():
    div = __aes_decrypt_module(divtext)
    return div

def __aes_decrypt_module(data):
    key = 'Z2B87NEAS2BkxTrh'
    iv = 'WwadH66EGWpeeTT6'
    return __aes_decrypt(data, key, iv)

def __aes_decrypt(data, key, iv):
    from Crypto.Cipher import AES
    import base64
    encodebytes = base64.decodebytes(data.encode('utf-8'))
    aes = AES.new(key.encode('utf-8'), AES.MODE_CBC, iv.encode('utf-8'))
    de_text = aes.decrypt(encodebytes)
    unpad = lambda s: s[0:-s[-1]]
    de_text = unpad(de_text)
    return de_text.decode('utf-8')


temptext = db_decrypt(encryptedpass[6:])
print(temptext)
```

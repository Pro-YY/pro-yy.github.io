---
layout: post
title: "[Solved] Shadowsocks: undefined symbol"
date: 2020-09-23 15:03:22 +0800
comments: true
categories: network
---

### Error: undefined symbol EVP_CIPHER_CTX_cleanup

After installing shadowsocks on Ubuntu cloud server:

```
sudo apt install python3-pip
sudo pip3 install shadowsocks
```

It should be easy for us to start server like:

```
ssserver -k hello -p 12345
```

However, for Ubuntu 18.04 or 20.04, Shadowsocks server command `ssserver` would lead to the following error:

```
2020-09-23 11:33:21 INFO     loading libcrypto from libcrypto.so.1.1
Traceback (most recent call last):
  File "/usr/local/bin/ssserver", line 8, in <module>
    sys.exit(main())
  File "/usr/local/lib/python3.8/dist-packages/shadowsocks/server.py", line 34, in main
    config = shell.get_config(False)
  File "/usr/local/lib/python3.8/dist-packages/shadowsocks/shell.py", line 262, in get_config
    check_config(config, is_local)
  File "/usr/local/lib/python3.8/dist-packages/shadowsocks/shell.py", line 124, in check_config
    encrypt.try_cipher(config['password'], config['method'])
  File "/usr/local/lib/python3.8/dist-packages/shadowsocks/encrypt.py", line 44, in try_cipher
    Encryptor(key, method)
  File "/usr/local/lib/python3.8/dist-packages/shadowsocks/encrypt.py", line 82, in __init__
    self.cipher = self.get_cipher(key, method, 1,
  File "/usr/local/lib/python3.8/dist-packages/shadowsocks/encrypt.py", line 109, in get_cipher
    return m[2](method, key, iv, op)
  File "/usr/local/lib/python3.8/dist-packages/shadowsocks/crypto/openssl.py", line 76, in __init__
    load_openssl()
  File "/usr/local/lib/python3.8/dist-packages/shadowsocks/crypto/openssl.py", line 52, in load_openssl
    libcrypto.EVP_CIPHER_CTX_cleanup.argtypes = (c_void_p,)
  File "/usr/lib/python3.8/ctypes/__init__.py", line 386, in __getattr__
    func = self.__getitem__(name)
  File "/usr/lib/python3.8/ctypes/__init__.py", line 391, in __getitem__
    func = self._FuncPtr((name_or_ordinal, self))
AttributeError: /lib/x86_64-linux-gnu/libcrypto.so.1.1: undefined symbol: EVP_CIPHER_CTX_cleanup
```

It seems that there's no symbol `EVP_CIPHER_CTX_cleanup` within libcrypto.so, which shadowsocks needs.


check the symbols in _libcrypto.so.1.1_:

```
brooke@VM-101-145-ubuntu:~$ nm -gD /usr/lib/x86_64-linux-gnu/libcrypto.so.1.1 | grep EVP_CIPHER_CTX
000000000016dab0 T EVP_CIPHER_CTX_block_size
000000000016deb0 T EVP_CIPHER_CTX_buf_noconst
000000000016dae0 T EVP_CIPHER_CTX_cipher
000000000016e510 T EVP_CIPHER_CTX_clear_flags
000000000016d350 T EVP_CIPHER_CTX_copy
000000000016cd10 T EVP_CIPHER_CTX_ctrl
000000000016daf0 T EVP_CIPHER_CTX_encrypting
000000000016c160 T EVP_CIPHER_CTX_free
000000000016db10 T EVP_CIPHER_CTX_get_app_data
000000000016db30 T EVP_CIPHER_CTX_get_cipher_data
000000000016de90 T EVP_CIPHER_CTX_iv
000000000016db60 T EVP_CIPHER_CTX_iv_length
000000000016dea0 T EVP_CIPHER_CTX_iv_noconst
000000000016def0 T EVP_CIPHER_CTX_key_length
000000000016c140 T EVP_CIPHER_CTX_new
000000000016e060 T EVP_CIPHER_CTX_nid
000000000016dec0 T EVP_CIPHER_CTX_num
000000000016de80 T EVP_CIPHER_CTX_original_iv
000000000016d310 T EVP_CIPHER_CTX_rand_key
000000000016c090 T EVP_CIPHER_CTX_reset
000000000016db20 T EVP_CIPHER_CTX_set_app_data
000000000016db40 T EVP_CIPHER_CTX_set_cipher_data
000000000016e500 T EVP_CIPHER_CTX_set_flags
000000000016d2a0 T EVP_CIPHER_CTX_set_key_length
000000000016ded0 T EVP_CIPHER_CTX_set_num
000000000016cce0 T EVP_CIPHER_CTX_set_padding
000000000016e520 T EVP_CIPHER_CTX_test_flags
```

No `cleanup`, but `reset`. And it was changed in OpenSSL 1.1.0, introduced [here](https://www.openssl.org/docs/man1.1.0/man3/EVP_CIPHER_CTX_reset.html).

> EVP_CIPHER_CTX was made opaque in OpenSSL 1.1.0. As a result, EVP_CIPHER_CTX_reset() appeared and EVP_CIPHER_CTX_cleanup() disappeared. EVP_CIPHER_CTX_init() remains as an alias for EVP_CIPHER_CTX_reset(). 

### Solution
Solved by replacing all (total 2) the `EVP_CIPHER_CTX_cleanup()` functions to `EVP_CIPHER_CTX_reset()` in the _openssl.py_ file. With the following command:

edit _/usr/local/lib/python3.8/dist-packages/shadowsocks/crypto/openssl.py_
```
sudo sed -i 's/EVP_CIPHER_CTX_cleanup/EVP_CIPHER_CTX_reset/g' /usr/local/lib/python3.8/dist-packages/shadowsocks/crypto/openssl.py
```

And restart the Shadowsocks server. Done!


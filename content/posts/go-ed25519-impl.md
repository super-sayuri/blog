---
title: "go里面使用ed25519签名的注意事项"
date: 2022-05-18T17:00:20+08:00
category: "技术"
tags: ["go", "加密", "签名", "ed25519"]
---

直接使用ssh-keygen创建出来的ed25519密钥会报错：
```
 asn1: structure error: tags don't match ...
```

这是因为创建出来的密钥是ssh链接加密而不是真正的密钥对，如果要使用真正的密钥对需要以下指令：

```bash
# 生成私钥
openssl genpkey -algorithm ed25519 -outform PEM -out $PRIVATE_KEY_PATH
# 生成公钥
openssl pkey -in $PRIVATE_KEY_PATH -pubout -out $PUBLIC_KEY_PATH
```

在go中的调用如下：

```go
  // private
	fcont, err = ioutil.ReadFile(privateKeyFilePath)
	if err != nil {
		return err
	}
	block, _ = pem.Decode(fcont)
	key, err = x509.ParsePKCS8PrivateKey(block.Bytes)
	if err != nil {
		return err
	}
	_privateKey, ok = key.(ed25519.PrivateKey)
	if !ok {
		return errors.New("cannot get private key")
	}

  // public
  fcont, err = ioutil.ReadFile(publicKeyFilePath)
	if err != nil {
		return err
	}
	block, _ = pem.Decode(fcont)
	key, err = x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		return err
	}
	_publicKey, ok = key.(ed25519.PublicKey)
	if !ok {
		return errors.New("cannot get private key")
	}
```

随后可以使用ed25519包下的 _Sign_ 和 _Verify_ 两个方法进行签名和验证。

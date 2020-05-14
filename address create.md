# 以太坊地址生成过程
文章目录
* 1 以太坊地址生成过程
* 2 以太坊地址生成实例
  * 2.1 生成随机数作为私钥
  * 2.2 私钥生成公钥
  * 2.2.1 使用 bx 工具生成公钥
  * 2.2.2 使用 secp256k1-py 包生成公钥
  * 2.3 计算公钥哈希值
  * 2.4 得到地址
* 3 以太坊地址生成 Python3 实现
* 4 参考资料
## 1 以太坊地址生成过程

以太坊地址生成过程如下：

* 生成 256 位随机数作为私钥。
* 将私钥转化为 secp256k1 非压缩格式的公钥，即 512 位的公钥。
* 使用散列算法 Keccak256 计算公钥的哈希值，转化为十六进制字符串。
* 取十六进制字符串的后 40 个字母，开头加上 0x 作为地址。
## 2 以太坊地址生成实例
生成以太坊地址过程实例数据：

* 私钥：1f2b77e3a4b50120692912c94b204540ad44404386b10c615786a7efaa065d20
* 公钥：04dfa13518ff965498743f3a01439dd86bc34ff9969c7a3f0430bbf8865734252953c9884af787b2cadd45f92dff2b81e21cfdf98873e492e5fdc07e9eb67ca74d
* 地址：0xabcd68033A72978C1084E2d44D1Fa06DdC4A2d57
## 2.1 生成随机数作为私钥
生成 256 位随机数：
```
>>> import random
>>> r = random.randint(0, 2**256)
>>> r
14098500174935566811277058424286341448580475958153633347646702637404947635488
>>> r.to_bytes(32, byteorder='big').hex()
'1f2b77e3a4b50120692912c94b204540ad44404386b10c615786a7efaa065d20'
```
## 2.2 私钥生成公钥
以太坊使用的椭圆曲线算法为 secp256k1，从私钥生成对应的公钥有两种方法：比特币工具 bx 和 secp256k1-py 包。
### 2.2.1 使用 bx 工具生成公钥
Mac 用户可以使用 brew 安装 bx 工具：
```
$ brew install bx
```
以 1f2b77e3a4b50120692912c94b204540ad44404386b10c615786a7efaa065d20 作为私钥，然后使用 bx 工具将私钥转化为 secp256k1 的非压缩格式公钥：
```
$ bx ec-to-public 1f2b77e3a4b50120692912c94b204540ad44404386b10c615786a7efaa065d20 -u
04dfa13518ff965498743f3a01439dd86bc34ff9969c7a3f0430bbf8865734252953c9884af787b2cadd45f92dff2b81e21cfdf98873e492e5fdc07e9eb67ca74d
```
### 2.2.2 使用 secp256k1-py 包生成公钥
使用 pip 安装：
```
$ pip install secp256k1
```
之后将私钥转化为公钥：
```
>>> import secp256k1
>>> private_key = '1f2b77e3a4b50120692912c94b204540ad44404386b10c615786a7efaa065d20'
>>> private_key = bytes.fromhex(private_key)
>>> privkey = secp256k1.PrivateKey(private_key)
>>> privkey.pubkey.serialize(compressed=False).hex()
'04dfa13518ff965498743f3a01439dd86bc34ff9969c7a3f0430bbf8865734252953c9884af787b2cadd45f92dff2b81e21cfdf98873e492e5fdc07e9eb67ca74d'
```
## 2.3 计算公钥哈希值
要使用 keccak256 哈希算法，可以使用 PyCryptodome 工具，使用 pip 进行安装：
```
$ pip install pycryptodome
```
公钥开头去除 04，将剩余部分转化为字节串并使用 keccak256 算法进行哈希:
```
>>> from Crypto.Hash import keccak
>>> keccak_hash = keccak.new(digest_bits=256)
>>> public_key = '04dfa13518ff965498743f3a01439dd86bc34ff9969c7a3f0430bbf8865734252953c9884af787b2cadd45f92dff2b81e21cfdf98873e492e5fdc07e9eb67ca74d'[2:]
>>> public_key = bytes.fromhex(public_key)
>>> keccak_hash.update(public_key)
<Crypto.Hash.keccak.Keccak_Hash object at 0x102960588>
>>> keccak_hash.hexdigest()
'39c0eb3b26d4838930b1f34babcd68033a72978c1084e2d44d1fa06ddc4a2d57'
```
## 2.4 得到地址
取哈希值十六进制字符串后 40 个字母，开头加上 0x 生成最终的以太坊地址：
```
>>> '0x' + '39c0eb3b26d4838930b1f34babcd68033a72978c1084e2d44d1fa06ddc4a2d57'[-40:]
'0xabcd68033a72978c1084e2d44d1fa06ddc4a2d57'
```
# 3 以太坊地址生成 Python3 实现
使用 Python3 实现以太坊地址生成：
```
import secp256k1
from Crypto.Hash import keccak

def get_eth_addr(private_key_str=None):
    if private_key_str is None:
        private_key = secp256k1.PrivateKey()
        private_key_str = private_key.serialize()
    else:
        private_key_bytes = bytes.fromhex(private_key_str)
        private_key = secp256k1.PrivateKey(private_key_bytes)
    public_key_bytes = private_key.pubkey.serialize(compressed=False)
    public_key_str = public_key_bytes.hex()
    keccak_hash = keccak.new(digest_bits=256)
    keccak_hash.update(public_key_bytes[1:])
    h = keccak_hash.hexdigest()
    address = '0x' + h[-40:]
    return {
        "private_key": private_key_str,
        "public_key": public_key_str,
        "address": address
    }
```
# 4 参考资料
* 以太坊在线地址生成工具：可以作为以太坊靓号地址生成工具，代码开源：https://github.com/bokub/vanity-eth。
* https://pycryptodome.readthedocs.io/en/latest/src/hash/keccak.html：Keccak python 工具。
* https://github.com/ctz/keccak：Python2 环境下使用的 Keccak，此为源代码，需要自己 clone 在本地使用。
* https://github.com/ludbb/secp256k1-py：secp256k1 的 Python 库。
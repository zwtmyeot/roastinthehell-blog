---
title: "jwt-go使用"
date: 2020-03-22T00:59:16+08:00
draft: false
tags: [
    "go"
]
---

解释一下什么是JWT，并记录一下jwt-go的使用<!--more-->

## JWT简介

JWT由三部分组成： 头部 (header)、负载 (payload)、签名 (signature) 组成，形如：

```txt
header.payload.signature
```

头部是一个JSON对象，包含了表示这是个JWT数据和所使用的的散列方法：

```txt
{
    "typ": "JWT",
    "alg": "HS256"
}
```

负载也是一个JSON对象，包含了你想放进去的信息，例如用户名、用户类型、此JWT颁发时间等字段：

```txt
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

签名是由前面的头部+负载经过散列算法得到

以下是生成JWT的伪代码：

```code
data = base64urlEncode( header ) + “.” + base64urlEncode( payload )
signature = hash( data, secret )
JWT = data + "." +  = base64urlEncode( signature )
```

首先 `base64` 是一个编码算法，编码不等于加密。编码是把字符串编码成HTTP传输安全的字符。

然后要明确散列（哈希）也不等加密，散列算法是把以长串的字符串，通过一定的算法，生成一个固定长度的字符串。这里使用的 `HS256` 算法就是会散列生成一段256字节长度的字符串。

优秀的散列算法可以保证较低的碰撞率，即避免两个不同的字符串散列后生成同一个散列结果。但是破解者可以制作一个彩虹表，记录每一个字符串散列后的结果，这样来倒推出原始字符串。所以我们就要加盐，上面散列函数第二个参数： `secret` 就是盐。把盐字符串插入到要散列的字符串中，就像撒盐一样。因为每个人的盐都是自己随机选择的，所以可以抵御彩虹表的破解。

加密是要求能把加密结果解密，获得原始信息，而正因为散列不可逆的特性，所以散列并不是加密算法。我们就是利用了散列算法这个特性来做签名。

当服务器收到一个JWT的时：

1. 直接对头部、负荷和签名部分进行base64解码，获得散列算法、用户信息、签名字符串

2. 组合头部和负荷部分，加盐然后散列，将结果和JWT第三部分签名字符串对比，如果没错那就可以证明这个JWT是自己颁发的。

使用JWT好处就是只需要颁布时对账号密码进行验证，之后收到JWT以后，只需要进行验证签名就行，不用再验证账号密码。所以JWT的核心用法是签名，不需要把密码字段保存到JWT里面，也不需要比对密码。

但是JWT也有一些缺点：

1. 在颁布时会填入一个过期时间，在颁布以后，这个JWT会一直有效，你无法废除特定的JWT

2. 如果负荷部分记录的信息过多，会导致JWT字符串数据较大，影响传输效率

JWT和Session验证各有优劣，需要根据使用场景选择合适的方法。

## 安装依赖

```sh
go get -u github.com/dgrijalva/jwt-go
```

## 生成和解析

要注意的是盐应该是一个 `[]byte`类型

```go
package main

import (
    "fmt"
    jwt "github.com/dgrijalva/jwt-go"
    "time"
)

// 盐，需要是字节数组，不能是string类型
var sault = []byte("fjkldsureow[];[p]v342146v8cx6rw")

// 作为负荷部分的结构体
type Clains struct {
    Username string `json:"username"`
    Usertype string `json:"usertype"`
    // 内置的一些字段，有过期时间，签发时间等
    jwt.StandardClaims
}

func main() {
    token, err := GenerateToken("Adam", "employee")
    if err != nil {
        fmt.Println("JWT generato error: ", err)
    }
    claim, err := ParseToken(token)
    if err != nil {
        fmt.Println("JWT parser error: ", err)
    }

    if claim != nil {
        fmt.Println(claim.Username, claim.Usertype)
        // Adam employee
    }

}

// 生成JWT
func GenerateToken(username, usertype string) (string, error) {
    // 过期时间
    nowTime := time.Now()
    expireTime := nowTime.Add(3 * time.Hour)

    // 声明一个负荷
    clains := Clains{
        Username: username,
        Usertype: usertype,
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: expireTime.Unix(),
            Issuer:    "me",
        },
    }

    // 返回一个Token类型变量，里面包含了加密方法，负荷结构体变量等
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, clains)
    // 调用签名函数，生成jwt字符串
    tokenString, err := token.SignedString(sault)
    return tokenString, err
}

// 解析JWT
func ParseToken(tokenString string) (*Clains, error) {
    // 解析JWT字符串为Token类型
    // 第三个参数是一个回调函数，可以在回调函数里自行验证一些数据
    // 例如验证散列算法是否是自己预先设定的、验证字负荷中某些字段是否存在
    // 如果验证不通过第一个返回值就返回nil
    token, err := jwt.ParseWithClaims(tokenString, &Clains{}, func(token *jwt.Token) (interface{}, error) {
        return sault, nil
    })

    // 从Token类型中获取负荷结构体
    if clains, ok := token.Claims.(*Clains); ok && token.Valid {
        return clains, nil
    } else {
        return nil, err
    }
}
```

## 参考

- [[译] 简单 5 步，理解 JWT](https://juejin.im/entry/5bb211b7e51d450e5c478e0b)

- [Gin实践 连载五 使用JWT进行身份校验](https://segmentfault.com/a/1190000013297828)
---
title: Bcrypt 加密算法详解
date: 2023-02-12 18:00:00
tags: [Spring, Java, 密码学]
---
最近在回顾中台系统的用户权限设计，在用户密码落库这块使用了 Bcrypt 算法来保证安全性，这篇文章是对该算法的一个详细介绍。

## 什么是 Bcrypt 加密算法
Bcrypt 是一种密码哈希函数，用于加密用户密码。它是一种强大且安全的算法，用于存储用户密码，并在用户登录时进行验证。
Bcrypt 算法使用了 Blowfish 加密算法，它在对用户密码进行加密时需要一个随机的盐值，这样就可以避免暴力破解。此外，Bcrypt 算法还具有慢哈希的特性，即在加密过程中使用的复杂度可以通过 cost 参数来控制，从而使用户的密码更加安全。

## 加密的过程
### 生成 Salt
首先，Bcrypt 会生成一个随机的 salt，这个 salt 会被用于密码加密过程。
### 对密码和 salt 进行加密
然后，密码和 salt 会被结合在一起，并使用密码哈希函数进行加密。
### 增加复杂度
在加密的过程中，Bcrypt 会使用指定的 cost 参数进行加密，这个参数代表加密的次数。通过增加 cost 参数，可以使加密过程更加复杂，从而提高密码的安全性。
### 生成密文
Bcrypt 加密后的密文是一个长度为 60~100 个字符的字符串。它包含了以下几部分信息：
1. 版本标识：Bcrypt 的密文开头一般以 "$2y$" 或 "$2a$" 开头，这代表了 Bcrypt 的版本信息。
2. cost 参数：密文的下一个字符是数字，代表了 Bcrypt 加密过程中使用的 cost 参数。
3. salt：接下来的 22 个字符代表了随机生成的 salt。使用 Base64 编码。
4. 哈希值：最后的字符串是加密后的哈希值，代表了密码的密文形式。使用 Base 64 编码。
一个加密后的密文可能如下所示：`"$2a$10$S8U6dsEjvAUtf7XuUzK8.eDmI.xMh7OJj0s8Q7WuOvZDz9XBk1PM2"`。

## 与 SHA-256 算法的对比
Bcrypt 和 SHA-256 都是常用的哈希算法，但它们的安全性是有所不同的。Bcrypt 有如下的优点：
1. 慢哈希：Bcrypt 算法具有慢哈希的特点，即在加密过程中使用的计算复杂度可以通过 cost 参数来控制，从而使密码更加安全。而 SHA-256 并不具有这种特点，速度较快。
2. 盐值：Bcrypt 在加密过程中使用了随机的盐值，这样就可以避免暴力破解。盐值存储在加密后的密文中，每个用户都有一个唯一的盐值。而 SHA-256 不支持使用盐值，因此它在数据库密码加密方面比 Bcrypt 更不安全
3. 强度：Bcrypt 算法具有较高的强度，比 SHA-256 算法更难被破解。

由于 Bcrypt 算法具有以上特点，因此它可以有效防止数据库被拖库，并保证密码不被泄露。即使数据库被黑客获取，他们也无法获得用户的原始密码，因为密码已经被加密。因此，使用 Bcrypt 算法存储密码是一种安全的方法。

## 实际的应用
Spring Security 就内置了 Bcrypt 算法的编码器 `BcryptPasswordEncoder`。
### 使用
在项目中添加 `org.springframework.security:spring-security-crypto` 的依赖后在配置项中声明一个 Bean，代码如下：
```Java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BcryptPasswordEncoder();
}
```
然后我们就可以在其他地方使用这个 `PasswordEncoder` 了，对应了两个方法 `encode`() 和 `matches`()。

### 实现源码
其实现了 `PasswordEncoder` 接口，接口的定义如下：
```Java
public interface PasswordEncoder {
	String encode(CharSequence rawPassword);
	boolean matches(CharSequence rawPassword, String encodedPassword);
}
```

编码的具体实现如下：
```Java
public String encode(CharSequence rawPassword) {
    String salt;
    if (strength > 0) {
        if (random != null) {
            salt = Bcrypt.gensalt(strength, random);
        }
        else {
            salt = Bcrypt.gensalt(strength);
        }
    }
    else {
        salt = Bcrypt.gensalt();
    }
    return Bcrypt.hashpw(rawPassword.toString(), salt);
}

public static String hashpw(String password, String salt) throws IllegalArgumentException {
    Bcrypt B;
    String real_salt;
    byte passwordb[], saltb[], hashed[];
    char minor = (char) 0;
    int rounds, off = 0;
    StringBuilder rs = new StringBuilder();
    if (salt == null) {
        throw new IllegalArgumentException("salt cannot be null");
    }
    int saltLength = salt.length();
    if (saltLength < 28) {
        throw new IllegalArgumentException("Invalid salt");
    }
    if (salt.charAt(0) != '$' || salt.charAt(1) != '2') {
        throw new IllegalArgumentException("Invalid salt version");
    }
    if (salt.charAt(2) == '$') {
        off = 3;
    } else {
        minor = salt.charAt(2);
        if (minor != 'a' || salt.charAt(3) != '$') {
            throw new IllegalArgumentException("Invalid salt revision");
        }
        off = 4;
    }
    if (saltLength - off < 25) {
        throw new IllegalArgumentException("Invalid salt");
    }
    // Extract number of rounds
    if (salt.charAt(off + 2) > '$') {
        throw new IllegalArgumentException("Missing salt rounds");
    }
    rounds = Integer.parseint(salt.substring(off, off + 2));
    real_salt = salt.substring(off + 3, off + 25);
    try {
        passwordb = (password + (minor >= 'a' ? "00" : "")).getBytes("UTF-8");
    }
    catch (UnsupportedEncodingException uee) {
        throw new AssertionError("UTF-8 is not supported");
    }
    saltb = decode_base64(real_salt, BCRYPT_SALT_LEN);
    B = new Bcrypt();
    hashed = B.crypt_raw(passwordb, saltb, rounds);
    rs.append("$2");
    if (minor >= 'a') {
        rs.append(minor);
    }
    rs.append("$");
    if (rounds < 10) {
        rs.append("0");
    }
    rs.append(rounds);
    rs.append("$");
    encode_base64(saltb, saltb.length, rs);
    encode_base64(hashed, bf_crypt_ciphertext.length * 4 - 1, rs);
    return rs.toString();
}
```

验证密码的实现如下：
```Java
public Boolean matches(CharSequence rawPassword, String encodedPassword) {
    if (encodedPassword == null || encodedPassword.length() == 0) {
        logger.warn("Empty encoded password");
        return false;
    }
    if (!BCRYPT_PATTERN.matcher(encodedPassword).matches()) {
        logger.warn("Encoded password does not look like Bcrypt");
        return false;
    }
    return Bcrypt.checkpw(rawPassword.toString(), encodedPassword);
}

public static Boolean checkpw(String plaintext, String hashed) {
    return equalsNoEarlyReturn(hashed, hashpw(plaintext, hashed));
}

static Boolean equalsNoEarlyReturn(String a, String b) {
    char[] caa = a.toCharArray();
    char[] cab = b.toCharArray();
    if (caa.length != cab.length) {
        return false;
    }
    byte ret = 0;
    for (int i = 0; i < caa.length; i++) {
        ret |= caa[i] ^ cab[i];
    }
    return ret == 0;
}
```

整个的代码是比较简单的，就不详细说明了。
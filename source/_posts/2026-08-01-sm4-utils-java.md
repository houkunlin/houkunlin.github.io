---
title: 零外部依赖的纯算法实现 SM4 国密加解密工具（Java实现）
date: 2026-08-01 16:51:14
updated: 2026-08-01 16:51:14
tags:
---

有时候需要对在项目中临时使用 SM4 加解密计算一些东西，但是又不想启动程序运行工具，或者不想打开在线工具网页，或者只能在命令行终端等场景使用。

工具使用说明：

```
SM4 国密算法工具 — 密钥生成 / 加解密

用法: java main.SM4Utils [选项]

选项:
  -m, --mode <ECB|CBC>           加密模式（默认 ECB）
  -k, --key <data>               SM4 密钥（编码格式由 --key-encoding 指定，可选，未指定则自动生成；须 16 字节）
  -i, --iv <data>                初始化向量 IV（CBC 模式使用，编码格式与密钥一致，未指定则自动生成；须 16 字节）
  -e, --encrypt <text>           待加密的明文字符串
  -d, --decrypt <data>           待解密的密文（编码格式由 --cipher-encoding 指定，兼容 SM4ENC(...) 格式）
  -ke, --key-encoding <fmt>      密钥 / IV 编码格式：HEX | BASE64 | BASE64URL | BASE32 | UTF8 | BINARY（默认 HEX；UTF8 仅适用于密钥 / IV）
  -ce, --cipher-encoding <fmt>   密文输入 / 输出编码格式：HEX | BASE64 | BASE64URL | BASE32 | BINARY（默认 HEX，不支持 UTF8）
  -h, --help                     显示此帮助信息

使用示例:
  java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
  java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...
  java main.SM4Utils -ke BASE64 -k ASNFZ4mrze8= -ce BASE64 -e HelloWorld
  java main.SM4Utils -ke UTF8 -k 1234567890abcdef -e HelloWorld
  java main.SM4Utils -e HelloWorld
  java main.SM4Utils -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -i 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
  java main.SM4Utils -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...
```

以下是 `SM4Utils.java` 文件内容：

```java
package main;

/**
 * <p>纯 Java 实现的 SM4 国密算法工具（无外部依赖）。</p>
 *
 * <p>支持 ECB / CBC 模式加解密，密钥、IV 及密文均支持 HEX / BASE64 / BASE64URL / BASE32 / UTF8 / BINARY
 * 多种编码格式（默认 HEX），可直接通过 {@code java main.SM4Utils} 命令行调用。</p>
 *
 * <p>命令行参数：</p>
 * <ul>
 *     <li>{@code -m <ECB|CBC>} / {@code --mode <ECB|CBC>} — 加密模式（默认 ECB）</li>
 *     <li>{@code -k <data>} / {@code --key <data>} — SM4 密钥（编码格式由 {@code --key-encoding} 指定，可选，未指定则自动生成）</li>
 *     <li>{@code -i <data>} / {@code --iv <data>} — 初始化向量 IV（CBC 模式使用，编码格式与密钥一致，未指定则自动生成）</li>
 *     <li>{@code -e <text>} / {@code --encrypt <text>} — 待加密的明文字符串</li>
 *     <li>{@code -d <data>} / {@code --decrypt <data>} — 待解密的密文（编码格式由 {@code --cipher-encoding} 指定，兼容 SM4ENC(...) 格式）</li>
 *     <li>{@code -ke <HEX|BASE64|BASE64URL|BASE32|UTF8|BINARY>} / {@code --key-encoding} — 密钥 / IV 编码格式（默认 HEX；UTF8 表示直接按 UTF-8 字节序列使用，仅适用于密钥 / IV）</li>
 *     <li>{@code -ce <HEX|BASE64|BASE64URL|BASE32|BINARY>} / {@code --cipher-encoding} — 密文输入 / 输出编码格式（默认 HEX，不支持 UTF8）</li>
 *     <li>{@code -h} / {@code --help} — 显示帮助信息</li>
 * </ul>
 *
 * <p>使用示例：</p>
 * <pre>{@code
 * java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
 * java main.SM4Utils -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -i 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
 * java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...
 * java main.SM4Utils -ke BASE64 -k ASNFZ4mrze8= -ce BASE64 -e HelloWorld
 * java main.SM4Utils -ke UTF8 -k 1234567890abcdef -e HelloWorld
 * java main.SM4Utils -e HelloWorld
 * }</pre>
 *
 * @author HouKunLin
 */
public class SM4Utils {

    // ======================== 编码格式 ========================

    /**
     * 密钥 / IV / 密文支持的编码格式
     */
    public enum Encoding {
        /**
         * 十六进制（每字节 2 个字符）
         */
        HEX,
        /**
         * Base64（RFC 4648，带 {@code =} 填充）
         */
        BASE64,
        /**
         * Base64 URL 安全变体（RFC 4648 §5，{@code -_} 替换 {@code +/}，带 {@code =} 填充）
         */
        BASE64URL,
        /**
         * Base32（RFC 4648 §6，带 {@code =} 填充）
         */
        BASE32,
        /**
         * UTF-8 明文（密钥 / IV 专用，直接按 UTF-8 字节序列使用）
         */
        UTF8,
        /**
         * 二进制 0/1 字符串（每字节 8 个字符）
         */
        BINARY;

        /**
         * 解析编码格式（不区分大小写）
         *
         * @param value 编码格式字符串
         * @return 对应的 {@link Encoding}
         * @throws IllegalArgumentException 不支持的编码格式
         */
        public static Encoding parse(String value) {
            try {
                return Encoding.valueOf(value.toUpperCase());
            } catch (IllegalArgumentException e) {
                throw new IllegalArgumentException("不支持的编码格式: " + value + "（仅支持 HEX / BASE64 / BASE64URL / BASE32 / UTF8 / BINARY）");
            }
        }
    }

    // ======================== S 盒（16x16 = 256 字节） ========================

    private static final int[] S_BOX = {
            0xd6, 0x90, 0xe9, 0xfe, 0xcc, 0xe1, 0x3d, 0xb7, 0x16, 0xb6, 0x14, 0xc2, 0x28, 0xfb, 0x2c, 0x05,
            0x2b, 0x67, 0x9a, 0x76, 0x2a, 0xbe, 0x04, 0xc3, 0xaa, 0x44, 0x13, 0x26, 0x49, 0x86, 0x06, 0x99,
            0x9c, 0x42, 0x50, 0xf4, 0x91, 0xef, 0x98, 0x7a, 0x33, 0x54, 0x0b, 0x43, 0xed, 0xcf, 0xac, 0x62,
            0xe4, 0xb3, 0x1c, 0xa9, 0xc9, 0x08, 0xe8, 0x95, 0x80, 0xdf, 0x94, 0xfa, 0x75, 0x8f, 0x3f, 0xa6,
            0x47, 0x07, 0xa7, 0xfc, 0xf3, 0x73, 0x17, 0xba, 0x83, 0x59, 0x3c, 0x19, 0xe6, 0x85, 0x4f, 0xa8,
            0x68, 0x6b, 0x81, 0xb2, 0x71, 0x64, 0xda, 0x8b, 0xf8, 0xeb, 0x0f, 0x4b, 0x70, 0x56, 0x9d, 0x35,
            0x1e, 0x24, 0x0e, 0x5e, 0x63, 0x58, 0xd1, 0xa2, 0x25, 0x22, 0x7c, 0x3b, 0x01, 0x21, 0x78, 0x87,
            0xd4, 0x00, 0x46, 0x57, 0x9f, 0xd3, 0x27, 0x52, 0x4c, 0x36, 0x02, 0xe7, 0xa0, 0xc4, 0xc8, 0x9e,
            0xea, 0xbf, 0x8a, 0xd2, 0x40, 0xc7, 0x38, 0xb5, 0xa3, 0xf7, 0xf2, 0xce, 0xf9, 0x61, 0x15, 0xa1,
            0xe0, 0xae, 0x5d, 0xa4, 0x9b, 0x34, 0x1a, 0x55, 0xad, 0x93, 0x32, 0x30, 0xf5, 0x8c, 0xb1, 0xe3,
            0x1d, 0xf6, 0xe2, 0x2e, 0x82, 0x66, 0xca, 0x60, 0xc0, 0x29, 0x23, 0xab, 0x0d, 0x53, 0x4e, 0x6f,
            0xd5, 0xdb, 0x37, 0x45, 0xde, 0xfd, 0x8e, 0x2f, 0x03, 0xff, 0x6a, 0x72, 0x6d, 0x6c, 0x5b, 0x51,
            0x8d, 0x1b, 0xaf, 0x92, 0xbb, 0xdd, 0xbc, 0x7f, 0x11, 0xd9, 0x5c, 0x41, 0x1f, 0x10, 0x5a, 0xd8,
            0x0a, 0xc1, 0x31, 0x88, 0xa5, 0xcd, 0x7b, 0xbd, 0x2d, 0x74, 0xd0, 0x12, 0xb8, 0xe5, 0xb4, 0xb0,
            0x89, 0x69, 0x97, 0x4a, 0x0c, 0x96, 0x77, 0x7e, 0x65, 0xb9, 0xf1, 0x09, 0xc5, 0x6e, 0xc6, 0x84,
            0x18, 0xf0, 0x7d, 0xec, 0x3a, 0xdc, 0x4d, 0x20, 0x79, 0xee, 0x5f, 0x3e, 0xd7, 0xcb, 0x39, 0x48,
    };

    // ======================== 系统参数 FK ========================

    private static final int FK0 = 0xA3B1BAC6;
    private static final int FK1 = 0x56AA3350;
    private static final int FK2 = 0x677D9197;
    private static final int FK3 = 0xB27022DC;

    // ======================== 固定常量 CK（32 个 32 位字） ========================

    private static final int[] CK = generateCK();

    private static int[] generateCK() {
        int[] ck = new int[32];
        for (int i = 0; i < 32; i++) {
            int b0 = ((4 * i) * 7) & 0xFF;
            int b1 = ((4 * i + 1) * 7) & 0xFF;
            int b2 = ((4 * i + 2) * 7) & 0xFF;
            int b3 = ((4 * i + 3) * 7) & 0xFF;
            ck[i] = (b0 << 24) | (b1 << 16) | (b2 << 8) | b3;
        }
        return ck;
    }

    // ======================== PKCS7 填充 ========================

    /**
     * PKCS7 填充，每 16 字节一组，不足时补足，满 16 时额外补 16 字节 {@code 0x10}
     */
    private static byte[] pkcs7Pad(byte[] data) {
        int padLen = 16 - (data.length % 16);
        byte[] padded = new byte[data.length + padLen];
        System.arraycopy(data, 0, padded, 0, data.length);
        for (int i = data.length; i < padded.length; i++) {
            padded[i] = (byte) padLen;
        }
        return padded;
    }

    /**
     * 去除 PKCS7 填充
     */
    private static byte[] pkcs7Unpad(byte[] data) {
        int padLen = data[data.length - 1] & 0xFF;
        if (padLen < 1 || padLen > 16) {
            throw new IllegalArgumentException("无效的 PKCS7 填充");
        }
        byte[] original = new byte[data.length - padLen];
        System.arraycopy(data, 0, original, 0, original.length);
        return original;
    }

    // ======================== 核心算法 ========================

    /**
     * 32 位整数的循环左移
     *
     * @param a 待移位整数
     * @param n 左移位数（0-31）
     * @return 循环左移结果
     */
    private static int rotl(int a, int n) {
        return (a << n) | (a >>> (32 - n));
    }

    /**
     * S 盒字节替换：对 32 位字的每个字节查 S 盒
     *
     * @param x 32 位输入
     * @return S 盒替换后的 32 位结果
     */
    private static int sBoxSub(int x) {
        return (S_BOX[(x >>> 24) & 0xFF] << 24)
                | (S_BOX[(x >>> 16) & 0xFF] << 16)
                | (S_BOX[(x >>> 8) & 0xFF] << 8)
                | S_BOX[x & 0xFF];
    }

    /**
     * 非线性变换 τ：4 个 S 盒并行替换
     */
    private static int tau(int a) {
        return sBoxSub(a);
    }

    /**
     * 线性变换 L（加密/解密中使用）
     * <p>L(B) = B ⊕ (B <<< 2) ⊕ (B <<< 10) ⊕ (B <<< 18) ⊕ (B <<< 24)</p>
     */
    private static int lTransform(int b) {
        return b ^ rotl(b, 2) ^ rotl(b, 10) ^ rotl(b, 18) ^ rotl(b, 24);
    }

    /**
     * 线性变换 L'（密钥扩展中使用）
     * <p>L'(B) = B ⊕ (B <<< 13) ⊕ (B <<< 23)</p>
     */
    private static int lPrimeTransform(int b) {
        return b ^ rotl(b, 13) ^ rotl(b, 23);
    }

    /**
     * 合成变换 T = L(τ(a))，用于加解密轮函数
     */
    private static int tTransform(int a) {
        return lTransform(tau(a));
    }

    /**
     * 合成变换 T' = L'(τ(a))，用于密钥扩展
     */
    private static int tPrimeTransform(int a) {
        return lPrimeTransform(tau(a));
    }

    /**
     * 轮函数 F
     * <p>X0 ⊕ T(X1 ⊕ X2 ⊕ X3 ⊕ rk)</p>
     */
    private static int roundFunction(int x0, int x1, int x2, int x3, int rk) {
        return x0 ^ tTransform(x1 ^ x2 ^ x3 ^ rk);
    }

    /**
     * 密钥扩展：从 128 位密钥生成 32 个 32 位轮密钥
     *
     * @param keyBytes 16 字节密钥
     * @return 32 个轮密钥
     */
    private static int[] keyExpansion(byte[] keyBytes) {
        int[] mk = bytesToWords(keyBytes);
        int k0 = mk[0] ^ FK0;
        int k1 = mk[1] ^ FK1;
        int k2 = mk[2] ^ FK2;
        int k3 = mk[3] ^ FK3;

        int[] rk = new int[32];
        int[] k = new int[36];
        k[0] = k0;
        k[1] = k1;
        k[2] = k2;
        k[3] = k3;

        for (int i = 0; i < 32; i++) {
            k[i + 4] = k[i] ^ tPrimeTransform(k[i + 1] ^ k[i + 2] ^ k[i + 3] ^ CK[i]);
            rk[i] = k[i + 4];
        }
        return rk;
    }

    /**
     * ECB 模式加密一个 16 字节数据块
     *
     * @param plaintext 16 字节明文块
     * @param rk        32 个轮密钥
     * @return 16 字节密文块
     */
    private static byte[] encryptBlock(byte[] plaintext, int[] rk) {
        int[] x = bytesToWords(plaintext);
        for (int i = 0; i < 32; i++) {
            int x4 = roundFunction(x[0], x[1], x[2], x[3], rk[i]);
            x[0] = x[1];
            x[1] = x[2];
            x[2] = x[3];
            x[3] = x4;
        }
        // 反序输出 X35, X34, X33, X32
        return wordsToBytes(new int[]{x[3], x[2], x[1], x[0]});
    }

    /**
     * ECB 模式解密一个 16 字节数据块（使用反向轮密钥）
     */
    private static byte[] decryptBlock(byte[] ciphertext, int[] rk) {
        int[] x = bytesToWords(ciphertext);
        // 解密时轮密钥逆序使用
        for (int i = 0; i < 32; i++) {
            int x4 = roundFunction(x[0], x[1], x[2], x[3], rk[31 - i]);
            x[0] = x[1];
            x[1] = x[2];
            x[2] = x[3];
            x[3] = x4;
        }
        return wordsToBytes(new int[]{x[3], x[2], x[1], x[0]});
    }

    // ======================== 公开 API ========================

    /**
     * 密码学安全的随机数生成器（JDK 自带，无外部依赖）
     */
    private static final java.security.SecureRandom SECURE_RANDOM = new java.security.SecureRandom();

    /**
     * 生成 16 字节（128 位）随机 SM4 密钥，返回 HEX 字符串
     *
     * @return 32 字符 HEX 密钥
     */
    public static String generateKey() {
        return bytesToHex(generateKeyBytes());
    }

    /**
     * 生成 16 字节（128 位）随机 SM4 密钥（使用 {@link java.security.SecureRandom}，密码学安全）
     *
     * @return 16 字节随机密钥
     */
    public static byte[] generateKeyBytes() {
        byte[] key = new byte[16];
        SECURE_RANDOM.nextBytes(key);
        return key;
    }

    /**
     * 生成 16 字节（128 位）随机 IV（初始化向量），返回 HEX 字符串
     *
     * @return 32 字符 HEX IV
     */
    public static String generateIv() {
        return bytesToHex(generateIvBytes());
    }

    /**
     * 生成 16 字节（128 位）随机 IV（初始化向量）（使用 {@link java.security.SecureRandom}，密码学安全）
     *
     * @return 16 字节随机 IV
     */
    public static byte[] generateIvBytes() {
        byte[] iv = new byte[16];
        SECURE_RANDOM.nextBytes(iv);
        return iv;
    }

    /**
     * 两个 16 字节块按位异或（用于 CBC 模式）
     */
    private static byte[] xor16(byte[] a, byte[] b) {
        byte[] out = new byte[16];
        for (int i = 0; i < 16; i++) {
            out[i] = (byte) (a[i] ^ b[i]);
        }
        return out;
    }

    // ======================== ECB 模式 ========================

    /**
     * SM4 ECB 模式加密
     *
     * @param plaintext 明文字节数组
     * @param key       16 字节密钥
     * @return 密文字节数组
     */
    public static byte[] encryptECB(byte[] plaintext, byte[] key) {
        int[] rk = keyExpansion(key);
        byte[] padded = pkcs7Pad(plaintext);
        byte[] result = new byte[padded.length];
        for (int i = 0; i < padded.length; i += 16) {
            byte[] block = new byte[16];
            System.arraycopy(padded, i, block, 0, 16);
            byte[] enc = encryptBlock(block, rk);
            System.arraycopy(enc, 0, result, i, 16);
        }
        return result;
    }

    /**
     * SM4 ECB 模式加密（密钥为 HEX 格式）
     *
     * @param plaintext 明文字节数组
     * @param keyHex    HEX 格式密钥（32 字符，16 字节）
     * @return 密文 HEX 字符串
     */
    public static String encryptECB(byte[] plaintext, String keyHex) {
        return bytesToHex(encryptECB(plaintext, hexToBytes(keyHex)));
    }

    /**
     * SM4 ECB 模式加密字符串
     *
     * @param data   明文字符串（UTF-8）
     * @param keyHex HEX 格式密钥
     * @return 密文 HEX 字符串
     */
    public static String encryptECBFromString(String data, String keyHex) {
        return encryptECB(data.getBytes(java.nio.charset.StandardCharsets.UTF_8), keyHex);
    }

    /**
     * SM4 ECB 模式解密
     *
     * @param ciphertext 密文字节数组
     * @param key        16 字节密钥
     * @return 解密后的字节数组（已去除填充）
     */
    public static byte[] decryptECB(byte[] ciphertext, byte[] key) {
        int[] rk = keyExpansion(key);
        byte[] result = new byte[ciphertext.length];
        for (int i = 0; i < ciphertext.length; i += 16) {
            byte[] block = new byte[16];
            System.arraycopy(ciphertext, i, block, 0, 16);
            byte[] dec = decryptBlock(block, rk);
            System.arraycopy(dec, 0, result, i, 16);
        }
        return pkcs7Unpad(result);
    }

    /**
     * SM4 ECB 模式解密（密钥为 HEX 格式）
     *
     * @param cipherHex 密文 HEX 字符串
     * @param keyHex    HEX 格式密钥
     * @return 解密后的字节数组（已去除填充）
     */
    public static byte[] decryptECB(String cipherHex, String keyHex) {
        return decryptECB(hexToBytes(cipherHex), hexToBytes(keyHex));
    }

    /**
     * SM4 ECB 模式解密为 UTF-8 字符串
     *
     * @param cipherHex 密文 HEX 字符串
     * @param keyHex    HEX 格式密钥
     * @return 解密后的 UTF-8 字符串
     */
    public static String decryptECBToString(String cipherHex, String keyHex) {
        return new String(decryptECB(cipherHex, keyHex), java.nio.charset.StandardCharsets.UTF_8);
    }

    // ======================== CBC 模式 ========================

    /**
     * SM4 CBC 模式加密
     *
     * @param plaintext 明文字节数组
     * @param key       16 字节密钥
     * @param iv        16 字节初始化向量
     * @return 密文字节数组
     */
    public static byte[] encryptCBC(byte[] plaintext, byte[] key, byte[] iv) {
        int[] rk = keyExpansion(key);
        byte[] padded = pkcs7Pad(plaintext);
        byte[] result = new byte[padded.length];
        byte[] prev = iv;
        for (int i = 0; i < padded.length; i += 16) {
            byte[] block = new byte[16];
            System.arraycopy(padded, i, block, 0, 16);
            byte[] xored = xor16(block, prev);
            byte[] enc = encryptBlock(xored, rk);
            System.arraycopy(enc, 0, result, i, 16);
            prev = enc;
        }
        return result;
    }

    /**
     * SM4 CBC 模式加密（密钥 / IV 为 HEX 格式）
     *
     * @param plaintext 明文字节数组
     * @param keyHex    HEX 格式密钥
     * @param ivHex     HEX 格式初始化向量（32 字符，16 字节）
     * @return 密文 HEX 字符串
     */
    public static String encryptCBC(byte[] plaintext, String keyHex, String ivHex) {
        return bytesToHex(encryptCBC(plaintext, hexToBytes(keyHex), hexToBytes(ivHex)));
    }

    /**
     * SM4 CBC 模式加密字符串
     *
     * @param data   明文字符串（UTF-8）
     * @param keyHex HEX 格式密钥
     * @param ivHex  HEX 格式初始化向量
     * @return 密文 HEX 字符串
     */
    public static String encryptCBCFromString(String data, String keyHex, String ivHex) {
        return encryptCBC(data.getBytes(java.nio.charset.StandardCharsets.UTF_8), keyHex, ivHex);
    }

    /**
     * SM4 CBC 模式解密
     *
     * @param ciphertext 密文字节数组
     * @param key        16 字节密钥
     * @param iv         16 字节初始化向量
     * @return 解密后的字节数组（已去除填充）
     */
    public static byte[] decryptCBC(byte[] ciphertext, byte[] key, byte[] iv) {
        int[] rk = keyExpansion(key);
        byte[] result = new byte[ciphertext.length];
        byte[] prev = iv;
        for (int i = 0; i < ciphertext.length; i += 16) {
            byte[] block = new byte[16];
            System.arraycopy(ciphertext, i, block, 0, 16);
            byte[] dec = decryptBlock(block, rk);
            byte[] xored = xor16(dec, prev);
            System.arraycopy(xored, 0, result, i, 16);
            prev = block;
        }
        return pkcs7Unpad(result);
    }

    /**
     * SM4 CBC 模式解密（密钥 / IV 为 HEX 格式）
     *
     * @param cipherHex 密文 HEX 字符串
     * @param keyHex    HEX 格式密钥
     * @param ivHex     HEX 格式初始化向量
     * @return 解密后的字节数组（已去除填充）
     */
    public static byte[] decryptCBC(String cipherHex, String keyHex, String ivHex) {
        return decryptCBC(hexToBytes(cipherHex), hexToBytes(keyHex), hexToBytes(ivHex));
    }

    /**
     * SM4 CBC 模式解密为 UTF-8 字符串
     *
     * @param cipherHex 密文 HEX 字符串
     * @param keyHex    HEX 格式密钥
     * @param ivHex     HEX 格式初始化向量
     * @return 解密后的 UTF-8 字符串
     */
    public static String decryptCBCToString(String cipherHex, String keyHex, String ivHex) {
        return new String(decryptCBC(cipherHex, keyHex, ivHex), java.nio.charset.StandardCharsets.UTF_8);
    }

    // ======================== HEX / 字节数组工具 ========================

    /**
     * HEX 字符串转字节数组
     */
    public static byte[] hexToBytes(String hex) {
        int len = hex.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(hex.charAt(i), 16) << 4)
                    | Character.digit(hex.charAt(i + 1), 16));
        }
        return data;
    }

    /**
     * 字节数组转大写 HEX 字符串
     */
    public static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder(bytes.length * 2);
        for (byte b : bytes) {
            sb.append(String.format("%02X", b & 0xFF));
        }
        return sb.toString();
    }

    // ======================== BASE64 / BASE64URL / BASE32 / UTF8 / BINARY 编码工具 ========================

    /**
     * Base32 字母表（RFC 4648，去除 I、L、O、0、1 等易混淆字符）
     */
    private static final char[] BASE32_ALPHABET = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567".toCharArray();

    /**
     * 字节数组转 BASE32 字符串（RFC 4648，带 {@code =} 填充）
     *
     * @param bytes 字节数组
     * @return BASE32 字符串
     */
    public static String bytesToBase32(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        int buffer = 0;
        int bitsLeft = 0;
        for (byte b : bytes) {
            buffer = (buffer << 8) | (b & 0xFF);
            bitsLeft += 8;
            while (bitsLeft >= 5) {
                bitsLeft -= 5;
                sb.append(BASE32_ALPHABET[(buffer >>> bitsLeft) & 0x1F]);
            }
        }
        if (bitsLeft > 0) {
            sb.append(BASE32_ALPHABET[(buffer << (5 - bitsLeft)) & 0x1F]);
        }
        while (sb.length() % 8 != 0) {
            sb.append('=');
        }
        return sb.toString();
    }

    /**
     * BASE32 字符串转字节数组（RFC 4648，兼容 {@code =} 填充）
     *
     * @param data BASE32 字符串
     * @return 字节数组
     */
    public static byte[] base32ToBytes(String data) {
        String cleaned = data.replace("=", "").toUpperCase();
        java.io.ByteArrayOutputStream out = new java.io.ByteArrayOutputStream();
        int buffer = 0;
        int bitsLeft = 0;
        for (int i = 0; i < cleaned.length(); i++) {
            int val = base32CharValue(cleaned.charAt(i));
            if (val < 0) {
                throw new IllegalArgumentException("非法 BASE32 字符: " + cleaned.charAt(i));
            }
            buffer = (buffer << 5) | val;
            bitsLeft += 5;
            if (bitsLeft >= 8) {
                bitsLeft -= 8;
                out.write((buffer >>> bitsLeft) & 0xFF);
            }
        }
        return out.toByteArray();
    }

    /**
     * 获取 Base32 字符对应的 5 位值
     *
     * @param c Base32 字符
     * @return 5 位值，非法字符返回 -1
     */
    private static int base32CharValue(char c) {
        if (c >= 'A' && c <= 'Z') {
            return c - 'A';
        }
        if (c >= '2' && c <= '7') {
            return c - '2' + 26;
        }
        return -1;
    }

    /**
     * 字节数组转 BINARY（二进制 0/1 字符串，每字节 8 位）
     *
     * @param bytes 字节数组
     * @return 二进制字符串
     */
    public static String bytesToBinary(byte[] bytes) {
        StringBuilder sb = new StringBuilder(bytes.length * 8);
        for (byte b : bytes) {
            for (int i = 7; i >= 0; i--) {
                sb.append((b >>> i) & 1);
            }
        }
        return sb.toString();
    }

    /**
     * BINARY（二进制 0/1 字符串）转字节数组
     *
     * @param data 二进制字符串（长度须为 8 的倍数）
     * @return 字节数组
     */
    public static byte[] binaryToBytes(String data) {
        if (data.length() % 8 != 0) {
            throw new IllegalArgumentException("二进制字符串长度必须是 8 的倍数");
        }
        byte[] out = new byte[data.length() / 8];
        for (int i = 0; i < out.length; i++) {
            int val = 0;
            for (int j = 0; j < 8; j++) {
                char c = data.charAt(i * 8 + j);
                if (c != '0' && c != '1') {
                    throw new IllegalArgumentException("非法二进制字符: " + c);
                }
                val = (val << 1) | (c - '0');
            }
            out[i] = (byte) val;
        }
        return out;
    }

    /**
     * 按指定编码格式将字节数组编码为字符串
     *
     * @param bytes    字节数组
     * @param encoding 编码格式
     * @return 编码后的字符串
     */
    public static String encodeBytes(byte[] bytes, Encoding encoding) {
        return switch (encoding) {
            case HEX -> bytesToHex(bytes);
            case BASE64 -> java.util.Base64.getEncoder().encodeToString(bytes);
            case BASE64URL -> java.util.Base64.getUrlEncoder().encodeToString(bytes);
            case BASE32 -> bytesToBase32(bytes);
            case UTF8 -> new String(bytes, java.nio.charset.StandardCharsets.UTF_8);
            case BINARY -> bytesToBinary(bytes);
        };
    }

    /**
     * 按指定编码格式将字符串解码为字节数组
     *
     * @param data     编码后的字符串
     * @param encoding 编码格式
     * @return 字节数组
     */
    public static byte[] decodeBytes(String data, Encoding encoding) {
        return switch (encoding) {
            case HEX -> hexToBytes(data);
            case BASE64 -> java.util.Base64.getDecoder().decode(data);
            case BASE64URL -> java.util.Base64.getUrlDecoder().decode(data);
            case BASE32 -> base32ToBytes(data);
            case UTF8 -> data.getBytes(java.nio.charset.StandardCharsets.UTF_8);
            case BINARY -> binaryToBytes(data);
        };
    }

    /**
     * 4 字节 → 1 个 int（大端序）
     */
    private static int bytesToWord(byte[] b, int off) {
        return (b[off] & 0xFF) << 24
                | (b[off + 1] & 0xFF) << 16
                | (b[off + 2] & 0xFF) << 8
                | b[off + 3] & 0xFF;
    }

    /**
     * 字节数组 → int 数组（每 4 字节一组）
     */
    private static int[] bytesToWords(byte[] b) {
        int[] words = new int[b.length / 4];
        for (int i = 0; i < words.length; i++) {
            words[i] = bytesToWord(b, i * 4);
        }
        return words;
    }

    /**
     * int 数组（大端序）→ 字节数组
     */
    private static byte[] wordsToBytes(int[] words) {
        byte[] b = new byte[words.length * 4];
        for (int i = 0; i < words.length; i++) {
            b[i * 4] = (byte) (words[i] >>> 24);
            b[i * 4 + 1] = (byte) (words[i] >>> 16);
            b[i * 4 + 2] = (byte) (words[i] >>> 8);
            b[i * 4 + 3] = (byte) words[i];
        }
        return b;
    }

    // ======================== 命令行入口 ========================

    /**
     * SM4 密钥生成 / 加解密命令行工具
     *
     * <p>参数说明：</p>
     * <ul>
     *     <li>{@code -m <ECB|CBC>} / {@code --mode <ECB|CBC>} — 加密模式（默认 ECB）</li>
     *     <li>{@code -k <data>} / {@code --key <data>} — SM4 密钥（编码格式由 {@code --key-encoding} 指定，可选，未指定则自动生成）</li>
     *     <li>{@code -i <data>} / {@code --iv <data>} — 初始化向量 IV（CBC 模式使用，编码格式与密钥一致，未指定则自动生成）</li>
     *     <li>{@code -e <text>} / {@code --encrypt <text>} — 待加密的明文字符串</li>
     *     <li>{@code -d <data>} / {@code --decrypt <data>} — 待解密的密文（编码格式由 {@code --cipher-encoding} 指定，兼容 SM4ENC(...) 格式）</li>
     *     <li>{@code -ke <HEX|BASE64|BASE64URL|BASE32|UTF8|BINARY>} / {@code --key-encoding} — 密钥 / IV 编码格式（默认 HEX；UTF8 表示直接按 UTF-8 字节序列使用，仅适用于密钥 / IV）</li>
     *     <li>{@code -ce <HEX|BASE64|BASE64URL|BASE32|BINARY>} / {@code --cipher-encoding} — 密文输入 / 输出编码格式（默认 HEX，不支持 UTF8）</li>
     *     <li>{@code -h} / {@code --help} — 显示帮助信息</li>
     * </ul>
     *
     * <p>使用示例：</p>
     * <pre>{@code
     * java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
     * java main.SM4Utils -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -i 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
     * java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...
     * java main.SM4Utils -ke BASE64 -k ASNFZ4mrze8= -ce BASE64 -e HelloWorld
     * java main.SM4Utils -ke UTF8 -k 1234567890abcdef -e HelloWorld
     * java main.SM4Utils -e HelloWorld
     * }</pre>
     */
    public static void main(String[] args) {
        String key = null;
        String encryptText = null;
        String decryptText = null;
        String iv = null;
        boolean cbcMode = false;
        boolean showHelp = false;
        Encoding keyEncoding = Encoding.HEX;
        Encoding cipherEncoding = Encoding.HEX;

        for (int i = 0; i < args.length; i++) {
            switch (args[i]) {
                case "-h", "--help" -> showHelp = true;
                case "-m", "--mode" -> {
                    if (i + 1 >= args.length) {
                        System.err.println("错误：-m 缺少模式参数");
                        printUsage();
                        return;
                    }
                    String mode = args[++i];
                    if ("CBC".equalsIgnoreCase(mode)) {
                        cbcMode = true;
                    } else if (!"ECB".equalsIgnoreCase(mode)) {
                        System.err.println("错误：不支持的加密模式: " + mode + "（仅支持 ECB / CBC）");
                        printUsage();
                        return;
                    }
                }
                case "-i", "--iv" -> {
                    if (i + 1 >= args.length) {
                        System.err.println("错误：-i 缺少 IV 参数");
                        printUsage();
                        return;
                    }
                    iv = args[++i];
                }
                case "-k", "--key" -> {
                    if (i + 1 >= args.length) {
                        System.err.println("错误：-k 缺少密钥参数");
                        printUsage();
                        return;
                    }
                    key = args[++i];
                }
                case "-e", "--encrypt" -> {
                    if (i + 1 >= args.length) {
                        System.err.println("错误：-e 缺少明文参数");
                        printUsage();
                        return;
                    }
                    encryptText = args[++i];
                }
                case "-d", "--decrypt" -> {
                    if (i + 1 >= args.length) {
                        System.err.println("错误：-d 缺少密文参数");
                        printUsage();
                        return;
                    }
                    decryptText = args[++i];
                }
                case "-ke", "--key-encoding" -> {
                    if (i + 1 >= args.length) {
                        System.err.println("错误：-ke 缺少编码格式参数");
                        printUsage();
                        return;
                    }
                    try {
                        keyEncoding = Encoding.parse(args[++i]);
                    } catch (IllegalArgumentException e) {
                        System.err.println("错误：" + e.getMessage());
                        printUsage();
                        return;
                    }
                }
                case "-ce", "--cipher-encoding" -> {
                    if (i + 1 >= args.length) {
                        System.err.println("错误：-ce 缺少编码格式参数");
                        printUsage();
                        return;
                    }
                    try {
                        cipherEncoding = Encoding.parse(args[++i]);
                    } catch (IllegalArgumentException e) {
                        System.err.println("错误：" + e.getMessage());
                        printUsage();
                        return;
                    }
                    if (cipherEncoding == Encoding.UTF8) {
                        System.err.println("错误：UTF8 明文编码仅适用于密钥 / IV（--key-encoding），密文编码请使用 HEX / BASE64 / BASE64URL / BASE32 / BINARY");
                        printUsage();
                        return;
                    }
                }
                default -> {
                    // 忽略未知参数
                }
            }
        }

        if (showHelp || (encryptText == null && decryptText == null)) {
            printUsage();
            return;
        }

        byte[] keyBytes;
        byte[] ivBytes = null;
        try {
            if (key == null) {
                keyBytes = generateKeyBytes();
                System.out.println("密钥（自动生成）：" + encodeBytes(keyBytes, keyEncoding));
            } else {
                keyBytes = decodeBytes(key, keyEncoding);
                System.out.println("密钥：" + encodeBytes(keyBytes, keyEncoding));
            }
            if (keyBytes.length != 16) {
                System.err.println("错误：SM4 密钥必须为 16 字节，当前为 " + keyBytes.length + " 字节");
                return;
            }
            if (cbcMode) {
                if (iv == null) {
                    ivBytes = generateIvBytes();
                    System.out.println("IV（自动生成）：" + encodeBytes(ivBytes, keyEncoding));
                } else {
                    ivBytes = decodeBytes(iv, keyEncoding);
                    System.out.println("IV：" + encodeBytes(ivBytes, keyEncoding));
                }
                if (ivBytes.length != 16) {
                    System.err.println("错误：IV 必须为 16 字节，当前为 " + ivBytes.length + " 字节");
                    return;
                }
            }
        } catch (Exception e) {
            System.err.println("密钥 / IV 解码失败：" + e.getMessage());
            return;
        }

        String modeLabel = cbcMode ? "CBC" : "ECB";

        try {
            if (encryptText != null) {
                byte[] encrypted = cbcMode
                        ? encryptCBC(encryptText.getBytes(java.nio.charset.StandardCharsets.UTF_8), keyBytes, ivBytes)
                        : encryptECB(encryptText.getBytes(java.nio.charset.StandardCharsets.UTF_8), keyBytes);
                System.out.println("加密模式：" + modeLabel);
                System.out.println("加密明文：" + encryptText);
                System.out.println("加密密文：" + encodeBytes(encrypted, cipherEncoding));
            }
        } catch (Exception e) {
            System.err.println("SM4 加密失败：" + e.getMessage());
        }

        try {
            if (decryptText != null) {
                String plain = decryptText;
                if (plain.startsWith("SM4ENC(") && plain.endsWith(")")) {
                    plain = plain.substring(7, plain.length() - 1);
                }
                byte[] cipherBytes = decodeBytes(plain, cipherEncoding);
                byte[] plainBytes = cbcMode
                        ? decryptCBC(cipherBytes, keyBytes, ivBytes)
                        : decryptECB(cipherBytes, keyBytes);
                System.out.println("解密模式：" + modeLabel);
                System.out.println("解密密文：" + decryptText);
                System.out.println("解密明文：" + new String(plainBytes, java.nio.charset.StandardCharsets.UTF_8));
            }
        } catch (Exception e) {
            System.err.println("SM4 解密失败：" + e.getMessage());
        }
    }

    private static void printUsage() {
        System.out.println("SM4 国密算法工具 — 密钥生成 / 加解密");
        System.out.println();
        System.out.println("用法: java " + SM4Utils.class.getName() + " [选项]");
        System.out.println();
        System.out.println("选项:");
        System.out.println("  -m, --mode <ECB|CBC>           加密模式（默认 ECB）");
        System.out.println("  -k, --key <data>               SM4 密钥（编码格式由 --key-encoding 指定，可选，未指定则自动生成；须 16 字节）");
        System.out.println("  -i, --iv <data>                初始化向量 IV（CBC 模式使用，编码格式与密钥一致，未指定则自动生成；须 16 字节）");
        System.out.println("  -e, --encrypt <text>           待加密的明文字符串");
        System.out.println("  -d, --decrypt <data>           待解密的密文（编码格式由 --cipher-encoding 指定，兼容 SM4ENC(...) 格式）");
        System.out.println("  -ke, --key-encoding <fmt>      密钥 / IV 编码格式：HEX | BASE64 | BASE64URL | BASE32 | UTF8 | BINARY（默认 HEX；UTF8 仅适用于密钥 / IV）");
        System.out.println("  -ce, --cipher-encoding <fmt>   密文输入 / 输出编码格式：HEX | BASE64 | BASE64URL | BASE32 | BINARY（默认 HEX，不支持 UTF8）");
        System.out.println("  -h, --help                     显示此帮助信息");
        System.out.println();
        System.out.println("使用示例:");
        System.out.println("  java " + SM4Utils.class.getName() + " -k 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld");
        System.out.println("  java " + SM4Utils.class.getName() + " -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...");
        System.out.println("  java " + SM4Utils.class.getName() + " -ke BASE64 -k ASNFZ4mrze8= -ce BASE64 -e HelloWorld");
        System.out.println("  java " + SM4Utils.class.getName() + " -ke UTF8 -k 1234567890abcdef -e HelloWorld");
        System.out.println("  java " + SM4Utils.class.getName() + " -e HelloWorld");
        System.out.println("  java " + SM4Utils.class.getName() + " -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -i 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld");
        System.out.println("  java " + SM4Utils.class.getName() + " -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...");
    }
}

```

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
  -m, --mode <ECB|CBC>  加密模式（默认 ECB）
  -k, --key <hex>       SM4 密钥（32 字符 HEX，可选，未指定则自动生成）
  -i, --iv <hex>        初始化向量 IV（CBC 模式使用，32 字符 HEX，未指定则自动生成）
  -e, --encrypt <text>  待加密的明文字符串
  -d, --decrypt <hex>   待解密的密文（HEX 格式，兼容 SM4ENC(...) 格式）
  -h, --help            显示此帮助信息

使用示例:
  java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
  java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...
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
 * <p>支持 ECB / CBC 模式加解密、16 进制 HEX 编码输出。
 * 可直接通过 {@code java main.SM4Utils} 命令行调用。</p>
 *
 * <p>命令行参数：</p>
 * <ul>
 *     <li>{@code -m <ECB|CBC>} / {@code --mode <ECB|CBC>} — 加密模式（默认 ECB）</li>
 *     <li>{@code -k <hex>} / {@code --key <hex>} — SM4 密钥（32 字符 HEX，可选，未指定则自动生成）</li>
 *     <li>{@code -i <hex>} / {@code --iv <hex>} — 初始化向量 IV（CBC 模式使用，未指定则自动生成）</li>
 *     <li>{@code -e <text>} / {@code --encrypt <text>} — 待加密的明文字符串</li>
 *     <li>{@code -d <hex>} / {@code --decrypt <hex>} — 待解密的密文（HEX 格式，兼容 SM4ENC(...) 格式）</li>
 *     <li>{@code -h} / {@code --help} — 显示帮助信息</li>
 * </ul>
 *
 * <p>使用示例：</p>
 * <pre>{@code
 * java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
 * java main.SM4Utils -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -i 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
 * java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...
 * java main.SM4Utils -e HelloWorld
 * }</pre>
 *
 * @author HouKunLin
 */
public class SM4Utils {

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
     * 生成 16 字节（128 位）随机 SM4 密钥，返回 HEX 字符串
     *
     * @return 32 字符 HEX 密钥
     */
    public static String generateKey() {
        byte[] key = new byte[16];
        for (int i = 0; i < 16; i++) {
            key[i] = (byte) (Math.random() * 256);
        }
        return bytesToHex(key);
    }

    /**
     * 生成 16 字节（128 位）随机 IV（初始化向量），返回 HEX 字符串
     *
     * @return 32 字符 HEX IV
     */
    public static String generateIv() {
        byte[] iv = new byte[16];
        for (int i = 0; i < 16; i++) {
            iv[i] = (byte) (Math.random() * 256);
        }
        return bytesToHex(iv);
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
     * @param keyHex    HEX 格式密钥（32 字符，16 字节）
     * @return 密文 HEX 字符串
     */
    public static String encryptECB(byte[] plaintext, String keyHex) {
        byte[] key = hexToBytes(keyHex);
        int[] rk = keyExpansion(key);
        byte[] padded = pkcs7Pad(plaintext);
        byte[] result = new byte[padded.length];
        for (int i = 0; i < padded.length; i += 16) {
            byte[] block = new byte[16];
            System.arraycopy(padded, i, block, 0, 16);
            byte[] enc = encryptBlock(block, rk);
            System.arraycopy(enc, 0, result, i, 16);
        }
        return bytesToHex(result);
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
     * @param cipherHex 密文 HEX 字符串
     * @param keyHex    HEX 格式密钥
     * @return 解密后的字节数组（已去除填充）
     */
    public static byte[] decryptECB(String cipherHex, String keyHex) {
        byte[] ciphertext = hexToBytes(cipherHex);
        byte[] key = hexToBytes(keyHex);
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
     * @param keyHex    HEX 格式密钥
     * @param ivHex     HEX 格式初始化向量（32 字符，16 字节）
     * @return 密文 HEX 字符串
     */
    public static String encryptCBC(byte[] plaintext, String keyHex, String ivHex) {
        byte[] key = hexToBytes(keyHex);
        byte[] iv = hexToBytes(ivHex);
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
        return bytesToHex(result);
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
     * @param cipherHex 密文 HEX 字符串
     * @param keyHex    HEX 格式密钥
     * @param ivHex     HEX 格式初始化向量
     * @return 解密后的字节数组（已去除填充）
     */
    public static byte[] decryptCBC(String cipherHex, String keyHex, String ivHex) {
        byte[] ciphertext = hexToBytes(cipherHex);
        byte[] key = hexToBytes(keyHex);
        byte[] iv = hexToBytes(ivHex);
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
     *     <li>{@code -k <hex>} / {@code --key <hex>} — SM4 密钥（32 字符 HEX，可选，未指定则自动生成）</li>
     *     <li>{@code -e <text>} / {@code --encrypt <text>} — 待加密的明文字符串</li>
     *     <li>{@code -d <hex>} / {@code --decrypt <hex>} — 待解密的密文（HEX 格式）</li>
     * </ul>
     *
     * <p>使用示例：</p>
     * <pre>{@code
     * java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld
     * java main.SM4Utils -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...
     * java main.SM4Utils -e HelloWorld
     * }</pre>
     */
    public static void main(String[] args) {
        String key = null;
        String encryptText = null;
        String decryptHex = null;
        String iv = null;
        boolean cbcMode = false;
        boolean showHelp = false;

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
                    decryptHex = args[++i];
                }
                default -> {
                    // 忽略未知参数
                }
            }
        }

        if (showHelp || (encryptText == null && decryptHex == null)) {
            printUsage();
            return;
        }

        if (cbcMode && iv == null) {
            iv = generateIv();
            System.out.println("IV（自动生成）：" + iv.toUpperCase());
        } else if (cbcMode) {
            System.out.println("IV：" + iv.toUpperCase());
        }

        if (key == null) {
            key = generateKey();
            System.out.println("密钥（自动生成）：" + key.toUpperCase());
        } else {
            System.out.println("密钥：" + key.toUpperCase());
        }

        String modeLabel = cbcMode ? "CBC" : "ECB";

        try {
            if (encryptText != null) {
                String encrypted = cbcMode
                        ? encryptCBCFromString(encryptText, key, iv)
                        : encryptECBFromString(encryptText, key);
                System.out.println("加密模式：" + modeLabel);
                System.out.println("加密明文：" + encryptText);
                System.out.println("加密密文：" + encrypted.toUpperCase());
            }
        } catch (Exception e) {
            System.err.println("SM4 加密失败：" + e.getMessage());
        }

        try {
            if (decryptHex != null) {
                String plain = decryptHex;
                if (plain.startsWith("SM4ENC(") && plain.endsWith(")")) {
                    plain = plain.substring(7, plain.length() - 1);
                }
                String decrypted = cbcMode
                        ? decryptCBCToString(plain, key, iv)
                        : decryptECBToString(plain, key);
                System.out.println("解密模式：" + modeLabel);
                System.out.println("解密密文：" + decryptHex);
                System.out.println("解密明文：" + decrypted);
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
        System.out.println("  -m, --mode <ECB|CBC>  加密模式（默认 ECB）");
        System.out.println("  -k, --key <hex>       SM4 密钥（32 字符 HEX，可选，未指定则自动生成）");
        System.out.println("  -i, --iv <hex>        初始化向量 IV（CBC 模式使用，32 字符 HEX，未指定则自动生成）");
        System.out.println("  -e, --encrypt <text>  待加密的明文字符串");
        System.out.println("  -d, --decrypt <hex>   待解密的密文（HEX 格式，兼容 SM4ENC(...) 格式）");
        System.out.println("  -h, --help            显示此帮助信息");
        System.out.println();
        System.out.println("使用示例:");
        System.out.println("  java " + SM4Utils.class.getName() + " -k 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld");
        System.out.println("  java " + SM4Utils.class.getName() + " -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...");
        System.out.println("  java " + SM4Utils.class.getName() + " -e HelloWorld");
        System.out.println("  java " + SM4Utils.class.getName() + " -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -i 0123456789ABCDEFFEDCBA9876543210 -e HelloWorld");
        System.out.println("  java " + SM4Utils.class.getName() + " -m CBC -k 0123456789ABCDEFFEDCBA9876543210 -d 3F4A...");
    }
}

```

---
title: 自动解密 SpringBoot 配置文件中的密文配置
date: 2026-08-01 17:00:11
updated: 2026-08-01 17:00:11
tags:
---

我们经常说密码明文、KEY不要写在配置文件中，以及不要上传到GIT，但是有时候可能真的一不小心就写了，一不小心就传了。

已知有一个很成熟的库 `jasypt-spring-boot-starter` 可以实现帮我们自动解密配置文件中的加密配置。 但是它好像不支持 SM4 算法，也没办法接入我们自己的加解密方案，特别是有专用加解密设备、密码机的场景。

在此提供一个 `EnvironmentPostProcessor` 的实现，用来稍微改改代码，就能接入自有的一套加解密方案。

下面是自己用的一套基于 SM4 解密的代码：

```java
package cn.shitcode.framework.crypto;

import cn.shitcode.framework.crypto.bc.SM4HEX;
import org.jspecify.annotations.NonNull;
import org.springframework.boot.EnvironmentPostProcessor;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.env.OriginTrackedMapPropertySource;
import org.springframework.boot.origin.Origin;
import org.springframework.boot.origin.OriginTrackedValue;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.MapPropertySource;
import org.springframework.core.env.MutablePropertySources;
import org.springframework.core.env.PropertySource;
import org.springframework.core.io.ClassPathResource;
import org.springframework.util.StreamUtils;
import org.springframework.util.StringUtils;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Path;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

/**
 * <p>Environment 后置处理器，在 Spring Boot 加载完所有配置文件后，
 * 遍历所有 PropertySource，将 {@code SM4ENC(...)} 格式的密文解密为明文。</p>
 *
 * <p>工作原理：</p>
 * <ul>
 *     <li>在 {@link EnvironmentPostProcessor#postProcessEnvironment} 阶段执行，
 *         早于任何 Bean 初始化</li>
 *     <li>遍历 {@link ConfigurableEnvironment#getPropertySources()} 中的每个 PropertySource</li>
 *     <li>对于 {@link MapPropertySource} 及 {@link OriginTrackedMapPropertySource} 类型的来源，
 *         逐一检查值是否以 {@code SM4ENC(} 开头并以 {@code )} 结尾</li>
 *     <li>若检测到加密值，使用 SM4/ECB 模式解密并用解密后的值替换原值；
 *         对于 {@link OriginTrackedMapPropertySource} 保留 {@link OriginTrackedValue} 修饰</li>
 * </ul>
 *
 * <p>密钥加载优先级：</p>
 * <ol>
 *     <li>JVM 参数 {@code -Ddecrypt_secret_key=...} 或全大写形式</li>
 *     <li>环境变量 {@code decrypt_secret_key=...} 或全大写形式</li>
 *     <li>文件系统 {@code decrypt_secret_key.key} / {@code decrypt_secret_key.properties}</li>
 *     <li>classpath {@code decrypt_secret_key.key} / {@code decrypt_secret_key.properties}</li>
 * </ol>
 *
 * @author HouKunLin
 */
@Order(Ordered.LOWEST_PRECEDENCE - 10)
public class SM4DecryptEnvironmentPostProcessor implements EnvironmentPostProcessor {
    /**
     * SM4 解密密钥的环境变量 / JVM 参数键名
     */
    public static final String KEY = "decrypt_secret_key";
    /**
     * 配置文件加密值前缀，格式：{@code SM4ENC(密文)}
     */
    public static final String PREFIX = "SM4ENC(";
    /**
     * 配置文件加密值后缀
     */
    public static final String SUFFIX = ")";

    /**
     * 从 JVM 参数 / 环境变量 / 密钥文件中读取到的 SM4 解密密钥（HEX 格式）
     */
    private String secretKey;

    /**
     * 遍历所有 PropertySource，解密 {@code SM4ENC(...)} 前缀的配置属性
     *
     * <p>在 Spring Boot 完成配置文件加载后、任何 Bean 初始化前执行。
     * 对每个 {@link MapPropertySource} 或 {@link OriginTrackedMapPropertySource}，
     * 检测并解密其中以 {@link #PREFIX} 开头、{@link #SUFFIX} 结尾的属性值，
     * 然后用解密后的值替换原始 PropertySource。</p>
     *
     * @param environment 当前 Spring 环境，包含所有已加载的 PropertySource
     * @param application 当前 Spring 应用
     */
    @Override
    public void postProcessEnvironment(@NonNull ConfigurableEnvironment environment, @NonNull SpringApplication application) {
        System.out.println("[DEBUG] " + getClass().getName() + ".postProcessEnvironment 被调用");
        secretKey = geSecretKey();
        if (secretKey == null) {
            System.out.println("[WARN] 未找到 SM4 解密密钥，跳过配置文件解密");
            logTips();
            return;
        }

        MutablePropertySources propertySources = environment.getPropertySources();
        Map<String, PropertySource<?>> replacements = new HashMap<>();

        for (PropertySource<?> source : propertySources) {
            if (source instanceof OriginTrackedMapPropertySource originTrackedMapPropertySource) {
                handler(replacements, originTrackedMapPropertySource);
            } else if (source instanceof MapPropertySource mapSource) {
                handler(replacements, mapSource);
            }
        }

        replacements.forEach(propertySources::replace);

        if (!replacements.isEmpty()) {
            System.out.println("[INFO] SM4 配置文件解密完成，共处理 " + replacements.size() + " 个 PropertySource");
        }
    }

    /**
     * 处理 {@link OriginTrackedMapPropertySource} 中的加密属性
     *
     * <p>解密后使用 {@link OriginTrackedValue} 包装结果，保留属性来源信息（如 YAML 行号），
     * 确保 Spring Boot 错误报告和 actuator 能正确溯源。</p>
     *
     * @param replacements 待替换的 PropertySource 映射，收集满足条件的替换源
     * @param source       当前遍历的 OriginTrackedMapPropertySource
     */
    private void handler(Map<String, PropertySource<?>> replacements, OriginTrackedMapPropertySource source) {
        Map<String, Object> decrypted = new HashMap<>();
        boolean hasEncrypted = false;
        for (String name : source.getPropertyNames()) {
            Object raw = source.getProperty(name);
            if (raw instanceof String value && isCipherText(value)) {
                try {
                    String decryptedValue = getDecryptText(value);
                    Origin origin = source.getOrigin(name);
                    if (origin == null) {
                        decrypted.put(name, OriginTrackedValue.of(decryptedValue));
                    } else {
                        decrypted.put(name, OriginTrackedValue.of(decryptedValue, origin));
                    }
                    hasEncrypted = true;
                    System.out.println("[INFO] 解密配置属性：" + name + "，配置来源：" + source.getName());
                } catch (Exception e) {
                    System.err.println("[ERROR] 无法解密配置属性: " + name + "，原始值: " + value + "，配置来源：" + source.getName());
                    e.printStackTrace(System.err);
                    throw new RuntimeException("无法解密配置属性: " + name + "，原始值: " + value + "，配置来源：" + source.getName(), e);
                }
            }
        }
        if (hasEncrypted) {
            Map<String, Object> merged = new HashMap<>(source.getSource());
            merged.putAll(decrypted);
            replacements.put(source.getName(), new OriginTrackedMapPropertySource(source.getName(), merged));
        }
    }

    /**
     * 处理普通 {@link MapPropertySource} 中的加密属性
     *
     * <p>解密后直接以明文值替换，不保留来源追踪信息。</p>
     *
     * @param replacements 待替换的 PropertySource 映射
     * @param source       当前遍历的 MapPropertySource
     */
    private void handler(Map<String, PropertySource<?>> replacements, MapPropertySource source) {
        Map<String, Object> decrypted = new HashMap<>();
        boolean hasEncrypted = false;
        for (String name : source.getPropertyNames()) {
            Object raw = source.getProperty(name);
            if (raw instanceof String value && isCipherText(value)) {
                try {
                    String decryptedValue = getDecryptText(value);
                    decrypted.put(name, decryptedValue);
                    hasEncrypted = true;
                    System.out.println("[INFO] 解密配置属性：" + name + "，配置来源：" + source.getName());
                } catch (Exception e) {
                    System.err.println("[ERROR] 无法解密配置属性: " + name + "，原始值: " + value + "，配置来源：" + source.getName());
                    e.printStackTrace(System.err);
                    throw new RuntimeException("无法解密配置属性: " + name + "，原始值: " + value + "，配置来源：" + source.getName(), e);
                }
            }
        }
        if (hasEncrypted) {
            Map<String, Object> merged = new HashMap<>(source.getSource());
            merged.putAll(decrypted);
            replacements.put(source.getName(), new MapPropertySource(source.getName(), merged));
        }
    }

    /**
     * 判断字符串是否为 {@code SM4ENC(...)} 格式的密文
     *
     * @param value 待判断的字符串
     * @return 若以 {@link #PREFIX} 开头且以 {@link #SUFFIX} 结尾则返回 true
     */
    private boolean isCipherText(String value) {
        return value.startsWith(PREFIX) && value.endsWith(SUFFIX);
    }

    /**
     * 从 {@code SM4ENC(密文)} 格式中提取括号内的密文字符串
     *
     * @param value 完整的加密值，如 {@code SM4ENC(abcd1234)}
     * @return 括号内的密文 HEX 字符串
     */
    private String getCipherText(String value) {
        return value.substring(PREFIX.length(), value.length() - SUFFIX.length());
    }

    /**
     * 解密 {@code SM4ENC(...)} 格式的密文
     *
     * <p>先从加密值中提取括号内的密文，再使用 SM4/ECB 模式解密为明文。</p>
     *
     * @param value 完整的加密值，如 {@code SM4ENC(abcd1234)}
     * @return 解密后的明文字符串
     * @throws Exception 解密失败时抛出异常
     */
    private String getDecryptText(String value) throws Exception {
        String cipherText = getCipherText(value);
        return SM4HEX.decryptECBToString(cipherText, secretKey);
    }

    /**
     * 打印密钥配置提示信息
     *
     * <p>当无法获取解密密钥时，输出所有可行的密钥配置方式供运维参考。</p>
     */
    private void logTips() {
        System.out.println("[WARN] 程序启动时无法从JVM参数、环境变量、密钥文件中读取到解密密钥，因此无法解密数据，程序可能会启动失败、运行错误，您可以从以下方案中任选一种方案设置 SM4 密钥");
        System.out.println("[WARN] 1. 请从 JVM参数 设置 SM4 密钥：-D" + KEY + "=SM4密钥（HEX格式）  或  -D" + KEY.toUpperCase() + "=SM4密钥（HEX格式）");
        System.out.println("[WARN] 2. 请从 环境变量 设置 SM4 密钥：" + KEY + "=SM4密钥（HEX格式）  或  " + KEY.toUpperCase() + "=SM4密钥（HEX格式）");
        System.out.println("[WARN] 3. 请在 [filesystem] " + new File(KEY + ".key").getAbsolutePath() + " 文件写入密钥内容：SM4密钥（HEX格式）");
        System.out.println("[WARN] 4. 请在 [filesystem] " + new File(KEY + ".properties").getAbsolutePath() + " 文件写入密钥内容：secret_key=SM4密钥（HEX格式）");
        try {
            Path filePath = new ClassPathResource("").getFilePath();
            System.out.println("[WARN] 5. 请在 [classpath] " + filePath + File.separator + KEY + ".key" + " 文件写入密钥内容：SM4密钥（HEX格式）");
            System.out.println("[WARN] 6. 请在 [classpath] " + filePath + File.separator + KEY + ".properties" + " 文件写入密钥内容：secret_key=SM4密钥（HEX格式）");
        } catch (IOException ignore) {
        }
    }

    /**
     * 按优先级依次从 JVM 参数、环境变量、文件系统、classpath 中获取 SM4 解密密钥
     *
     * @return SM4 密钥（HEX 格式），所有来源均未找到时返回 null
     */
    private String geSecretKey() {
        String property = System.getProperty(KEY);
        if (StringUtils.hasText(property)) {
            System.out.println("[INFO] 从JVM参数 " + KEY + " 获取到 SM4 解密密钥");
            return property;
        }
        property = System.getProperty(KEY.toUpperCase());
        if (StringUtils.hasText(property)) {
            System.out.println("[INFO] 从JVM参数 " + KEY.toUpperCase() + " 获取到 SM4 解密密钥");
            return property;
        }
        property = System.getenv(KEY);
        if (StringUtils.hasText(property)) {
            System.out.println("[INFO] 从环境变量 " + KEY + " 获取到 SM4 解密密钥");
            return property;
        }
        property = System.getenv(KEY.toUpperCase());
        if (StringUtils.hasText(property)) {
            System.out.println("[INFO] 从环境变量 " + KEY.toUpperCase() + " 获取到 SM4 解密密钥");
            return property;
        }

        String key = getSecretKeyByFile();
        if (StringUtils.hasText(key)) {
            return key;
        }
        key = getSecretKeyByClasspath();
        if (StringUtils.hasText(key)) {
            return key;
        }
        System.out.println("[WARN] 无法加载 SM4 配置文件属性解密密钥，如果配置文件中存在 SM4 加密的密文配置，系统有可能无法正常运行。如果系统没有使用到 SM4 加密配置文件属性，那么请忽略该提示内容");
        return null;
    }

    /**
     * 从文件系统读取 SM4 解密密钥
     *
     * <p>按以下顺序尝试读取：</p>
     * <ol>
     *     <li>{@code decrypt_secret_key.key} — 文件内容直接作为密钥</li>
     *     <li>{@code decrypt_secret_key.properties} — 从 {@code secret_key} 属性读取</li>
     * </ol>
     *
     * @return SM4 密钥（HEX 格式），文件不存在或无密钥内容时返回 null
     */
    private String getSecretKeyByFile() {
        File keyFile = new File(KEY + ".key");
        if (keyFile.exists()) {
            try (InputStream inputStream = new FileInputStream(keyFile)) {
                String text = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);
                String key = text.replace("\r", "").replace("\n", "");
                if (StringUtils.hasText(key)) {
                    System.out.println("[INFO] [filesystem] 从密钥文件获取到 SM4 解密密钥：" + keyFile.getAbsolutePath());
                    return key;
                }
                System.err.println("[ERROR] [filesystem] 密钥文件无密钥内容：" + keyFile.getAbsolutePath());
            } catch (IOException e) {
                throw new RuntimeException("[filesystem] 无法读取 \"" + keyFile.getAbsolutePath() + "\" 文件", e);
            }
        }
        File propertiesFile = new File(KEY + ".properties");
        if (propertiesFile.exists()) {
            try (InputStream inputStream = new FileInputStream(propertiesFile)) {
                Properties properties = new Properties();
                properties.load(inputStream);
                String key = properties.getProperty("secret_key", null);
                if (StringUtils.hasText(key)) {
                    System.out.println("[INFO] [filesystem] 从属性配置文件获取到 SM4 解密密钥：" + propertiesFile.getAbsolutePath());
                    return key;
                }
                System.err.println("[ERROR] [filesystem] 属性配置文件无密钥内容：" + propertiesFile.getAbsolutePath());
            } catch (IOException e) {
                throw new RuntimeException("[filesystem] 无法读取 \"" + propertiesFile.getAbsolutePath() + "\" 文件", e);
            }
        }
        return null;
    }

    /**
     * 从 classpath 读取 SM4 解密密钥
     *
     * <p>按以下顺序尝试读取：</p>
     * <ol>
     *     <li>{@code classpath:decrypt_secret_key.key} — 文件内容直接作为密钥</li>
     *     <li>{@code classpath:decrypt_secret_key.properties} — 从 {@code secret_key} 属性读取</li>
     * </ol>
     *
     * @return SM4 密钥（HEX 格式），资源不存在或无密钥内容时返回 null
     */
    private String getSecretKeyByClasspath() {
        ClassPathResource keyResource = new ClassPathResource(KEY + ".key");
        if (keyResource.exists()) {
            try (InputStream inputStream = keyResource.getInputStream()) {
                String text = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);
                String key = text.replace("\r", "").replace("\n", "");
                if (StringUtils.hasText(key)) {
                    System.out.println("[INFO] [classpath] 从密钥文件获取到 SM4 解密密钥：" + keyResource.getURI());
                    return key;
                }
                System.err.println("[ERROR] [classpath] 密钥文件无密钥内容：" + keyResource.getURI());
            } catch (IOException e) {
                throw new RuntimeException("[classpath] 无法读取 \"" + KEY + ".key" + "\" 文件", e);
            }
        }
        ClassPathResource propertiesResource = new ClassPathResource(KEY + ".properties");
        if (propertiesResource.exists()) {
            try (InputStream inputStream = keyResource.getInputStream()) {
                Properties properties = new Properties();
                properties.load(inputStream);
                String key = properties.getProperty("secret_key", null);
                if (StringUtils.hasText(key)) {
                    System.out.println("[INFO] [classpath] 从属性配置文件获取到 SM4 解密密钥：" + propertiesResource.getURI());
                    return key;
                }
                System.err.println("[ERROR] [classpath] 属性配置文件无密钥内容：" + propertiesResource.getURI());
            } catch (IOException e) {
                throw new RuntimeException("[classpath] 无法读取 \"" + KEY + ".properties" + "\" 文件", e);
            }
        }
        return null;
    }
}

```

除了增加上面的 Java 文件外，还需要增加一个配置，需要在 `resources/META-INF/spring.factories` 配置文件中增加如下配置：

```
# EnvironmentPostProcessor 实现类注册，用于 SM4 配置文件解密
org.springframework.boot.EnvironmentPostProcessor=\
cn.shitcode.framework.crypto.SM4DecryptEnvironmentPostProcessor

```

只有增加如上配置，才能正式生效。

以及 Java 代码中为什么不用 Slf4j 这类日志框架打印日志？这是因为此时 Slf4j 日志框架还未初始化，即使用了，也无法在控制台和日志文件中打印日志，因此只能使用常规的 System.out 来输出信息

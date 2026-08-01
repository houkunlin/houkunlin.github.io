---
title: Gradle 缓存依赖转换成 Maven 仓库存储结构
date: 2026-08-01 16:44:00
updated: 2026-08-01 16:44:00
tags:
---

有时候我们在使用 Gradle 来开发项目的时候，遇到要把项目整体离线迁移和与其他正常 Maven 项目共用依赖的场景，此时项目会有很多的jar文件和依赖信息，一个个的迁移很麻烦，官方也没有什么很方便的迁移方案，因此写了一个脚本来完成对 Gradle 缓存依赖转换成 Maven 仓库结构。

脚本使用说明

```
将 Gradle 本地缓存中的依赖复制到 Maven 本地仓库目录，并自动从远程仓库补充缺失的元数据文件（POM、module、jar）。

完成以下工作:
  1. 从 Gradle 缓存 (~/.gradle/caches/modules-2/files-2.1) 复制 jar 到目标目录
  2. 补充缺失的 .pom 文件
  3. 自动检测 build.gradle 中的插件声明并下载对应的 Gradle 插件 marker POM
  4. 递归解析父 POM 和 BOM 导入
  5. 可选: 下载标准 jar (--jars)
  6. 可选: 下载 .module 元数据文件 (--modules)
  7. 可选: 下载 .module 中声明的所有变体 jar (--variants)

用法: bun run scripts/gradle2maven.ts --target <path> [选项]

选项:
  --target <path>      输出 Maven 仓库目录路径（必填）
  --source <path>      Gradle 缓存目录路径（默认: ~/.gradle/caches/modules-2/files-2.1）
  --repo <url>         远程 Maven 仓库地址（默认: https://repo1.maven.org/maven2）
  --jars               下载缺失的标准 jar（默认跳过）
  --modules            下载 .module 元数据文件（默认跳过）
  --variants           下载 .module 中声明的所有变体 jar（默认跳过）
  -h, --help           显示此帮助信息

示例:
  bun run scripts/gradle2maven.ts --target libs/maven --jars
  bun run scripts/gradle2maven.ts --target libs/maven --jars --modules --variants
  bun run scripts/gradle2maven.ts --target libs/maven --repo https://maven.aliyun.com/repository/public
```

以下是一个使用 `Bun` 运行的 js 脚本，用来完成此项工作。

```js
#!/usr/bin/env bun
/**
 * 将 Gradle 本地缓存中的依赖复制到 Maven 本地仓库目录
 *
 * Gradle 缓存结构：
 *   ~/.gradle/caches/modules-2/files-2.1/<group>/<artifact>/<version>/<hash>/<file>
 *
 * 转换后 Maven 结构：
 *   <output>/<groupPath>/<artifact>/<version>/<file>
 *   其中 groupPath = group 中的 '.' 替换为 '/'
 *
 * 如果 Maven 目标目录中缺少 .pom 或 .module 文件，则自动从远程仓库下载补充。
 * 同时递归解析 Maven 父 POM（<parent>）和 BOM 导入（<dependencyManagement>），
 * 确保所有元数据完整。
 */

// @ts-nocheck
import { existsSync, mkdirSync, copyFileSync, readdirSync, statSync, writeFileSync, readFileSync } from 'node:fs'
import { join, resolve } from 'node:path'
import { homedir } from 'node:os'

const GRADLE_CACHE = join(
  homedir(),
  '.gradle',
  'caches',
  'modules-2',
  'files-2.1',
)

const DEFAULT_MAVEN_REPO = 'https://repo1.maven.org/maven2'

function printUsage() {
  console.log(`将 Gradle 本地缓存中的依赖复制到 Maven 本地仓库目录，并自动从远程仓库补充缺失的元数据文件（POM、module、jar）。

完成以下工作:
  1. 从 Gradle 缓存 (~/.gradle/caches/modules-2/files-2.1) 复制 jar 到目标目录
  2. 补充缺失的 .pom 文件
  3. 自动检测 build.gradle 中的插件声明并下载对应的 Gradle 插件 marker POM
  4. 递归解析父 POM 和 BOM 导入
  5. 可选: 下载标准 jar (--jars)
  6. 可选: 下载 .module 元数据文件 (--modules)
  7. 可选: 下载 .module 中声明的所有变体 jar (--variants)

用法: bun run scripts/gradle2maven.ts --target <path> [选项]

选项:
  --target <path>      输出 Maven 仓库目录路径（必填）
  --source <path>      Gradle 缓存目录路径（默认: ~/.gradle/caches/modules-2/files-2.1）
  --repo <url>         远程 Maven 仓库地址（默认: https://repo1.maven.org/maven2）
  --jars               下载缺失的标准 jar（默认跳过）
  --modules            下载 .module 元数据文件（默认跳过）
  --variants           下载 .module 中声明的所有变体 jar（默认跳过）
  -h, --help           显示此帮助信息

示例:
  bun run scripts/gradle2maven.ts --target libs/maven --jars
  bun run scripts/gradle2maven.ts --target libs/maven --jars --modules --variants
  bun run scripts/gradle2maven.ts --target libs/maven --repo https://maven.aliyun.com/repository/public
`)
}

function parseArgs(): { source: string; target: string; repo: string; variants: boolean; modules: boolean; jars: boolean } {
  const args = process.argv.slice(2)
  let source = GRADLE_CACHE
  let target = ''
  let repo = DEFAULT_MAVEN_REPO
  let variants = false
  let modules = false
  let jars = false

  if (args.length === 0 || args.includes('--help') || args.includes('-h')) {
    printUsage()
    process.exit(0)
  }

  for (let i = 0; i < args.length; i++) {
    if (args[i] === '--source' && i + 1 < args.length) {
      source = resolve(args[++i])
    } else if (args[i] === '--target' && i + 1 < args.length) {
      target = resolve(args[++i])
    } else if (args[i] === '--repo' && i + 1 < args.length) {
      repo = args[++i]
    } else if (args[i] === '--variants') {
      variants = true
    } else if (args[i] === '--no-variants') {
      variants = false
    } else if (args[i] === '--modules') {
      modules = true
    } else if (args[i] === '--no-modules') {
      modules = false
    } else if (args[i] === '--jars') {
      jars = true
    } else if (args[i] === '--no-jars') {
      jars = false
    } else if (args[i] === '--help' || args[i] === '-h') {
      printUsage()
      process.exit(0)
    } else if (!args[i].startsWith('--')) {
      target = resolve(args[i])
    }
  }

  if (!target) {
    console.error('❌ 错误: --target 参数是必填的\n')
    printUsage()
    process.exit(1)
  }

  return { source, target, repo, variants, modules, jars }
}

function* walk(dir: string): Generator<{ file: string; relative: string }> {
  const entries = readdirSync(dir, { recursive: true }) as string[]
  for (const entry of entries) {
    const fullPath = join(dir, entry)
    if (statSync(fullPath).isFile()) {
      yield { file: fullPath, relative: entry }
    }
  }
}

function toMavenPath(relative: string): string | null {
  // 路径格式: <group>/<artifact>/<version>/<hash>/<filename>
  const parts = relative.replace(/\\/g, '/').split('/')
  if (parts.length < 5) return null

  const group = parts[0]
  const artifact = parts[1]
  const version = parts[2]
  const filename = parts.slice(4).join('/')

  const groupPath = group.replace(/\./g, '/')
  return join(groupPath, artifact, version, filename)
}

function getVersionDir(mavenPath: string): string {
  return mavenPath.replace(/\\/g, '/').split('/').slice(0, -1).join('/')
}

async function download(url: string, dest: string): Promise<boolean> {
  try {
    const response = await fetch(url)
    if (!response.ok) return false
    const buffer = await response.arrayBuffer()
    mkdirSync(join(dest, '..'), { recursive: true })
    writeFileSync(dest, new Uint8Array(buffer))
    return true
  } catch {
    return false
  }
}

function parseXmlTag(xml: string, tag: string): string | null {
  const regex = new RegExp(`<${tag}>([^<]*)</${tag}>`)
  const match = xml.match(regex)
  return match?.[1] ?? null
}

function resolveProperties(text: string, props: Record<string, string>, fallbackVersion?: string): string {
  return text.replace(/\$\{([^}]+)\}/g, (_, key) => {
    if (key === 'project.version' || key === 'pom.version') return fallbackVersion ?? text
    return props[key] ?? text
  })
}

function collectProperties(pomContent: string): Record<string, string> {
  const props: Record<string, string> = {}
  const propsMatch = pomContent.match(/<properties>([\s\S]*?)<\/properties>/)
  if (propsMatch) {
    const propRegex = /<([^>]+)>([^<]*)<\/\1>/g
    let m
    while ((m = propRegex.exec(propsMatch[1])) !== null) {
      props[m[1]] = m[2]
    }
  }
  return props
}

/**
 * 从 POM 中提取 <parent> 信息，并解析属性引用
 */
function extractParent(pomContent: string, props: Record<string, string>, currentVersion: string) {
  const parentMatch = pomContent.match(/<parent>([\s\S]*?)<\/parent>/)
  if (!parentMatch) return null

  const xml = parentMatch[1]
  const rawGroupId = parseXmlTag(xml, 'groupId')
  const rawArtifactId = parseXmlTag(xml, 'artifactId')
  const rawVersion = parseXmlTag(xml, 'version')

  if (!rawGroupId || !rawArtifactId || !rawVersion) return null

  const groupId = resolveProperties(rawGroupId, props, currentVersion)
  const artifactId = resolveProperties(rawArtifactId, props, currentVersion)
  const version = resolveProperties(rawVersion, props, currentVersion)

  return version.startsWith('${') ? null : { groupId, artifactId, version }
}

/**
 * 从 POM 中提取所有 BOM 导入（<dependencyManagement> 中 type=pom, scope=import 的依赖）
 */
function extractBomImports(pomContent: string, props: Record<string, string>, currentVersion: string) {
  const depsMatch = pomContent.match(/<dependencyManagement>([\s\S]*?)<\/dependencyManagement>/)
  if (!depsMatch) return []

  const boms: Array<{ groupId: string; artifactId: string; version: string }> = []
  const depRegex = /<dependency>([\s\S]*?)<\/dependency>/g
  let m
  while ((m = depRegex.exec(depsMatch[1])) !== null) {
    const xml = m[1]
    const type = parseXmlTag(xml, 'type') || 'jar'
    const scope = parseXmlTag(xml, 'scope') || 'compile'
    if (type !== 'pom' || scope !== 'import') continue

    const rawGroupId = parseXmlTag(xml, 'groupId')
    const rawArtifactId = parseXmlTag(xml, 'artifactId')
    const rawVersion = parseXmlTag(xml, 'version')
    if (!rawGroupId || !rawArtifactId || !rawVersion) continue

    const groupId = resolveProperties(rawGroupId, props, currentVersion)
    const artifactId = resolveProperties(rawArtifactId, props, currentVersion)
    const version = resolveProperties(rawVersion, props, currentVersion)

    if (!version.startsWith('${')) {
      boms.push({ groupId, artifactId, version })
    }
  }
  return boms
}

/**
 * 从 build.gradle / settings.gradle 中提取插件声明
 */
function extractPlugins(buildFile: string): Array<{ id: string; version: string }> {
  if (!existsSync(buildFile)) return []

  const content = readFileSync(buildFile, 'utf-8')
  const blockMatch = content.match(/plugins\s*\{([\s\S]*?)\}/)
  if (!blockMatch) return []

  const plugins: Array<{ id: string; version: string }> = []
  for (const line of blockMatch[1].split('\n')) {
    const m = line.match(/id\s+['"]([^'"]+)['"]\s+.*?version\s+['"]([^'"]+)['"]/)
    if (m) {
      plugins.push({ id: m[1], version: m[2] })
    }
  }
  return plugins
}

/**
 * 下载 Gradle 插件的 marker POM
 * marker POM 的坐标规则: group = pluginId, artifact = pluginId.gradle.plugin, version = pluginVersion
 */
async function downloadPluginMarkers(
  repo: string,
  target: string,
  plugins: Array<{ id: string; version: string }>,
): Promise<{ downloaded: number; failed: number }> {
  let downloaded = 0
  let failed = 0

  for (const plugin of plugins) {
    const markerGroup = plugin.id
    const markerArtifact = `${plugin.id}.gradle.plugin`
    const markerVersion = plugin.version
    const groupPath = markerGroup.replace(/\./g, '/')
    const pomFileName = `${markerArtifact}-${markerVersion}.pom`
    const pomRelPath = `${groupPath}/${markerArtifact}/${markerVersion}/${pomFileName}`
    const dest = join(target, pomRelPath)

    if (existsSync(dest)) continue

    const url = `${repo}/${pomRelPath}`
    if (await download(url, dest)) {
      console.log(`  📄 Marker POM: ${markerGroup}:${markerArtifact}:${markerVersion}`)
      downloaded++
    } else {
      failed++
    }
  }

  return { downloaded, failed }
}

/**
 * 递归解析 POM 的父 POM 和 BOM 导入依赖并下载缺失的元数据
 */
async function resolveMetadataRecursive(
  repo: string,
  target: string,
  progress: { pomDownloaded: number; pomFailed: number },
) {
  const visited = new Set<string>()

  // 找出目标目录下所有已存在的 POM 文件作为起始队列
  const queue: string[] = []
  function scanPoms(dir: string) {
    if (!existsSync(dir)) return
    for (const entry of readdirSync(dir) as string[]) {
      const full = join(dir, entry)
      if (statSync(full).isDirectory()) {
        scanPoms(full)
      } else if (entry.endsWith('.pom')) {
        queue.push(full)
      }
    }
  }
  scanPoms(target)

  while (queue.length > 0) {
    const pomPath = queue.shift()!
    if (visited.has(pomPath)) continue
    visited.add(pomPath)

    const content = readFileSync(pomPath, 'utf-8')
    const currentVersion = parseXmlTag(content, 'version') ?? ''
    const props = collectProperties(content)

    // 解析父 POM
    const parent = extractParent(content, props, currentVersion)
    if (parent) {
      const groupPath = parent.groupId.replace(/\./g, '/')
      const pomRelPath = `${groupPath}/${parent.artifactId}/${parent.version}/${parent.artifactId}-${parent.version}.pom`
      const dest = join(target, pomRelPath)
      if (!existsSync(dest)) {
        const url = `${repo}/${pomRelPath}`
        if (await download(url, dest)) {
          progress.pomDownloaded++
          console.log(`  📄 Parent POM: ${parent.groupId}:${parent.artifactId}:${parent.version}`)
          queue.push(dest)
        } else {
          progress.pomFailed++
        }
      }
    }

    // 解析 BOM 导入
    const boms = extractBomImports(content, props, currentVersion)
    for (const bom of boms) {
      const groupPath = bom.groupId.replace(/\./g, '/')
      const pomRelPath = `${groupPath}/${bom.artifactId}/${bom.version}/${bom.artifactId}-${bom.version}.pom`
      const dest = join(target, pomRelPath)
      if (!existsSync(dest)) {
        const url = `${repo}/${pomRelPath}`
        if (await download(url, dest)) {
          progress.pomDownloaded++
          console.log(`  📄 BOM POM: ${bom.groupId}:${bom.artifactId}:${bom.version}`)
          queue.push(dest)
        } else {
          progress.pomFailed++
        }
      }
    }
  }
}

async function main() {
  const { source, target, repo, variants, modules, jars } = parseArgs()

  if (!existsSync(source)) {
    console.error(`❌ Gradle 缓存目录不存在: ${source}`)
    console.error(`   请先执行 ./gradlew downloadDependencies 填充缓存`)
    process.exit(1)
  }

  console.log(`📦 源目录: ${source}`)
  console.log(`🎯 目标目录: ${target}`)
  console.log(`🌐 远程仓库: ${repo}`)
  console.log(`⚙️  module: ${modules ? '下载' : '跳过'}, 标准 jar: ${jars ? '下载' : '跳过'}, 变体 jar: ${variants ? '下载' : '跳过'}`)
  console.log()

  // === 第一阶段：从 Gradle 缓存复制文件 ===
  let copied = 0
  let skippedFile = 0
  const versionDirs = new Set<string>()

  for (const { file, relative } of walk(source)) {
    const mavenPath = toMavenPath(relative)
    if (!mavenPath) {
      skippedFile++
      continue
    }

    versionDirs.add(getVersionDir(mavenPath))

    const dest = join(target, mavenPath)
    const destDir = join(dest, '..')

    if (existsSync(dest)) {
      skippedFile++
      continue
    }

    mkdirSync(destDir, { recursive: true })
    copyFileSync(file, dest)
    copied++
  }

  console.log(`📁 共发现 ${versionDirs.size} 个版本目录`)
  console.log()

  // === 第二阶段：补充缺失的 .pom 和 .module 文件 ===
  let pomDownloaded = 0
  let moduleDownloaded = 0
  let pomSkipped = 0
  let moduleSkipped = 0
  let pomFailed = 0
  let moduleFailed = 0

  for (const versionDir of versionDirs) {
    const parts = versionDir.replace(/\\/g, '/').split('/')
    const artifactName = parts[parts.length - 2]
    const version = parts[parts.length - 1]
    const targetDir = join(target, versionDir)

    if (!existsSync(targetDir)) continue

    const pomFile = `${artifactName}-${version}.pom`
    const pomDest = join(targetDir, pomFile)
    if (!existsSync(pomDest)) {
      const url = `${repo}/${versionDir}/${pomFile}`
      if (await download(url, pomDest)) {
        console.log(`  📄 POM: ${versionDir}/${pomFile}`)
        pomDownloaded++
      } else {
        pomFailed++
      }
    } else {
      pomSkipped++
    }

    if (modules) {
      const moduleFile = `${artifactName}-${version}.module`
      const moduleDest = join(targetDir, moduleFile)
      if (!existsSync(moduleDest)) {
        const url = `${repo}/${versionDir}/${moduleFile}`
        if (await download(url, moduleDest)) {
          console.log(`  📄 MODULE: ${versionDir}/${moduleFile}`)
          moduleDownloaded++
        } else {
          moduleFailed++
        }
      } else {
        moduleSkipped++
      }
    }
  }

  console.log()
  if (pomDownloaded > 0 || pomFailed > 0) {
    console.log(`📋 POM 文件: 已存在 ${pomSkipped}, 下载成功 ${pomDownloaded}, 下载失败 ${pomFailed}`)
  }
  if (moduleDownloaded > 0 || moduleFailed > 0) {
    console.log(`📋 Module 文件: 已存在 ${moduleSkipped}, 下载成功 ${moduleDownloaded}, 下载失败 ${moduleFailed}`)
  }

  // === 第三阶段：自动检测并下载 Gradle 插件的 marker POM ===
  console.log()
  console.log('🔄 开始检测并下载 Gradle 插件 marker POM...')
  const buildFile = join(process.cwd(), 'build.gradle')
  const settingsFile = join(process.cwd(), 'settings.gradle')
  const allPlugins = [
    ...extractPlugins(buildFile),
    ...extractPlugins(settingsFile),
  ]
  if (allPlugins.length > 0) {
    console.log(`  检测到 ${allPlugins.length} 个插件声明`)
    const markerResult = await downloadPluginMarkers(repo, target, allPlugins)
    console.log(`📋 Marker POM: 下载成功 ${markerResult.downloaded}, 下载失败 ${markerResult.failed}`)
    if (markerResult.downloaded > 0) {
      // 将新下载的 marker 目录加入 versionDirs，方便后续遍历
      for (const plugin of allPlugins) {
        const markerDir = `${plugin.id.replace(/\./g, '/')}/${plugin.id}.gradle.plugin/${plugin.version}`
        versionDirs.add(markerDir)
      }
    }
  } else {
    console.log('  未检测到插件声明')
  }

  // === 第四阶段：递归解析父 POM 和 BOM 导入 ===
  console.log()
  console.log('🔄 开始递归解析父 POM 和 BOM 依赖...')
  const metadataProgress = { pomDownloaded: 0, pomFailed: 0 }
  await resolveMetadataRecursive(repo, target, metadataProgress)
  if (metadataProgress.pomDownloaded > 0 || metadataProgress.pomFailed > 0) {
    console.log(`📋 元数据解析: 下载成功 ${metadataProgress.pomDownloaded}, 下载失败 ${metadataProgress.pomFailed}`)
  } else {
    console.log('📋 元数据解析: 全部已存在，无需下载')
  }

  // === 第五阶段：下载缺失的标准 jar（无 classifier，默认跳过） ===
  if (jars) {
    console.log()
    console.log('🔄 开始检查并下载缺失的标准 jar...')
    let stdJarDownloaded = 0
    let stdJarSkipped = 0
    let stdJarFailed = 0

    for (const versionDir of versionDirs) {
      const targetDir = join(target, versionDir)
      if (!existsSync(targetDir)) continue

      const parts = versionDir.replace(/\\/g, '/').split('/')
      const artifactName = parts[parts.length - 2]
      const version = parts[parts.length - 1]
      const standardJar = `${artifactName}-${version}.jar`
      const jarDest = join(targetDir, standardJar)

      if (existsSync(jarDest)) {
        stdJarSkipped++
        continue
      }
      const url = `${repo}/${versionDir}/${standardJar}`
      if (await download(url, jarDest)) {
        console.log(`  📦 JAR: ${versionDir}/${standardJar}`)
        stdJarDownloaded++
      } else {
        stdJarFailed++
      }
    }
    console.log(`📋 标准 JAR: 已存在 ${stdJarSkipped}, 下载成功 ${stdJarDownloaded}, 下载失败 ${stdJarFailed}`)
  }

  // === 第六阶段：根据 .module 变体下载变体 jar（仅 --variants） ===
  if (variants) {
    console.log()
    console.log('🔄 开始解析 .module 变体信息并下载变体 jar...')
    let variantDownloaded = 0
    let variantSkipped = 0
    let variantFailed = 0

    for (const versionDir of versionDirs) {
      const targetDir = join(target, versionDir)
      if (!existsSync(targetDir)) continue

      const moduleFiles = (readdirSync(targetDir) as string[]).filter(f => f.endsWith('.module'))
      if (moduleFiles.length === 0) continue

      const neededFiles = new Set<string>()
      for (const mf of moduleFiles) {
        try {
          const moduleJson = JSON.parse(readFileSync(join(targetDir, mf), 'utf-8'))
          for (const variant of (moduleJson.variants ?? [])) {
            for (const f of (variant.files ?? [])) {
              if (f.name && f.name.endsWith('.jar')) {
                neededFiles.add(f.name)
              }
            }
          }
        } catch { /* ignore */ }
      }

      for (const jarName of neededFiles) {
        const jarDest = join(targetDir, jarName)
        if (existsSync(jarDest)) { variantSkipped++; continue }
        const url = `${repo}/${versionDir}/${jarName}`
        if (await download(url, jarDest)) {
          console.log(`  📦 JAR: ${versionDir}/${jarName}`)
          variantDownloaded++
        } else {
          variantFailed++
        }
      }
    }
    console.log(`📋 变体 JAR: 已存在 ${variantSkipped}, 下载成功 ${variantDownloaded}, 下载失败 ${variantFailed}`)
  }

  console.log()
  console.log(`✅ 完成: 复制 ${copied} 个文件, 跳过 ${skippedFile} 个文件`)
  console.log(`   目标: ${target}`)
}

main().catch(console.error)
```

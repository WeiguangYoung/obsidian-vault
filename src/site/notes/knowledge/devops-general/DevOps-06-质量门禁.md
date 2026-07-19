---
{"dg-publish":true,"permalink":"/knowledge/devops-general/DevOps-06-质量门禁/","tags":["DevOps","质量","测试","安全","静态分析"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","质量","测试","安全","静态分析"]}}
---


# 四、质量门禁

## 4.1 静态分析

| 工具 | 语言 | 说明 |
|:----|:------|:------|
| **SonarQube** | 多语言 | 综合质量平台，技术债务度量 |
| **Cppcheck** | C/C++ | 内存泄漏、越界、未初始化检测 |
| **ESLint** | JavaScript/TypeScript | 代码风格 + 潜在错误 |
| **Checkstyle** | Java | 代码规范检查 |
| **MISRA-C** | C | 汽车行业安全编码标准 |

### SonarQube 质量门禁

```
覆盖率 < 80%         → ❌ 阻断
新增 Bug > 0         → ❌ 阻断
代码异味 > 阈值       → ⚠️ 警告
安全漏洞 > Critical  → ❌ 阻断
```

### CI 集成示例

```groovy
stage('Static Analysis') {
    steps {
        sh 'cppcheck --enable=all --xml src/ 2> cppcheck.xml'
        sh 'sonar-scanner -Dsonar.projectKey=myapp'
    }
}
```

## 4.2 单元测试

| 框架 | 语言 | 特点 |
|:----|:------|:------|
| **GTest** | C++ | Google 出品，xUnit 风格 |
| **JUnit** | Java | 最成熟 |
| **pytest** | Python | 简洁，fixture 机制强大 |

### CI 集成

```groovy
stage('Unit Test') {
    steps {
        sh 'cmake --build build && cd build && ctest --output-on-failure'
    }
    post {
        always {
            junit 'build/test-results/*.xml'
        }
    }
}
```

### 覆盖率

| 工具 | 语言 | CI 输出 |
|:----|:------|:------|
| **gcov/lcov** | C/C++ | HTML 报告 + XML |
| **JaCoCo** | Java | XML 报告 |
| **coverage.py** | Python | HTML 报告 |

## 4.3 安全扫描

| 扫描类型 | 工具 | 说明 |
|:----|:------|:------|
| **SAST**（静态） | SonarQube / Semgrep | 源码级别漏洞检测 |
| **DAST**（动态） | OWASP ZAP | 运行时攻击模拟 |
| **容器扫描** | Trivy / Clair | 镜像漏洞扫描 |
| **依赖扫描** | Snyk / OWASP Dependency-Check | 开源依赖漏洞检测 |
| **SBOM** | Syft / CycloneDX | 软件物料清单生成 |

### CI 集成

```groovy
stage('Security') {
    steps {
        sh 'trivy image --severity HIGH,CRITICAL myapp:${BUILD_TAG}'
        sh 'dependency-check --project myapp --scan ./'
    }
}
```

## 4.4 行业专项

| 领域 | 标准/工具 | 说明 |
|:----|:------|:------|
| **汽车** | MISRA-C | 嵌入式 C 代码安全编码规范 |
| **汽车** | ASPICE | 软件过程评估模型，需求-代码-测试追溯 |
| **功能安全** | ISO 26262 | 道路车辆功能安全标准 |
| **智驾** | HIL / SIL | 硬件在环 / 软件在环测试 |
| **智驾** | CANoe | 车载网络仿真与测试 |

## 4.5 质量门禁流水线设计

```
代码提交
  ↓
静态分析（SonarQube / Cppcheck）
  ↓ 通过 ← 质量门禁：覆盖率 ≥ 80%，无新增 Bug
单元测试（GTest / pytest）
  ↓ 通过 ← 必须 100%
安全扫描（Trivy / SAST）
  ↓ 通过 ← 无 HIGH/CRITICAL 漏洞
镜像构建 + 推送到测试仓库
  ↓
部署到测试环境
```

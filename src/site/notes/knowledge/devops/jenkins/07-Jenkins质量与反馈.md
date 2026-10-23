---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/07-Jenkins质量与反馈/","tags":["Jenkins","CI/CD","DevOps","质量门禁","SonarQube","通知"],"dg-note-properties":{"date":"2026-09-16","tags":["Jenkins","CI/CD","DevOps","质量门禁","SonarQube","通知"]}}
---


# 🧩 Jenkins 质量与反馈

> Jenkins 功能模块全景 · 第七章。构建跑完之后——质量怎么把关、结果怎么反馈给人。

---

## 一、Test Report（测试报告）

把单测/覆盖率结果**解析成可视化报告**，并在 UI 上展示趋势。

```groovy
post {
    always {
        junit 'build/reports/**/*.xml'            // JUnit 单测
        jacoco execPattern: '**/*.exec'           // JaCoCo 覆盖率
        publishHTML(target: [                     // 任意 HTML 报告
            reportDir: 'reports', reportFiles: 'index.html',
            reportName: '测试报告'])
    }
}
```

| 插件 | 用途 |
|:-----|:-----|
| **JUnit** | 单测结果（通过率、失败用例、趋势） |
| **JaCoCo / Cobertura** | 代码覆盖率 |
| **HTML Publisher** | 任意静态报告（HTML） |

- 报告带 **趋势图**：可以看到"通过率是涨还是跌"
- **失败用例会直接标红**，点进去看堆栈

## 二、代码扫描集成（质量门禁）

把静态分析工具接进流水线，不过门禁不让合入。

| 工具 | 语言/领域 | 说明 |
|:-----|:-----|:-----|
| **SonarQube** | 多语言 | 老牌代码质量管理平台，`withSonarQubeEnv` 接入 |
| **cppcheck** | C/C++ | 轻量静态分析，**你的车载项目在用** |
| **clang-tidy** | C/C++ | 基于 Clang 的静态检查 + 修复建议 |
| **MISRA** | 汽车 C | 汽车行业编码规范检查 |
| **Warnings NG** | 通用 | 聚合各类警告，通过/失败阈值可配 |
| **SCA / 二进制成分扫描** | 供应链 | 开源合规、漏洞检测 |

**接入方式**：

```groovy
stage('CodeCheck') {
    steps {
        // 静态分析，产出报告
        sh 'cppcheck --xml --xml-version=2 src/ 2> cppcheck.xml || true'
        // 解析报告并设门禁
        recordIssues tools: [cppCheck(pattern: 'cppcheck.xml')],
                     qualityGates: [[threshold: 1, type: 'TOTAL', unstable: true]]
    }
}
```

- **门禁思路**：告警数超阈值 → 标记 `UNSTABLE` 或 `FAILURE`，拦住合并
- **⚠️ 坑**：
  - 通配符 `**` 可能扫到多份残留报告 → 条数翻倍误报
  - 第三方 SDK 头文件的噪声要 **suppress**（否则 94% 都是无用告警）
  - 别用 `|| true` 把真故障也静默吞掉

## 三、构建状态语义

Jenkins 的构建结果**不只成功/失败**，五种状态含义不同：

| 状态 | 含义 | 是否阻断 |
|:-----|:-----|:---:|
| **SUCCESS** | 全绿 | — |
| **UNSTABLE** | 构建成功但有告警/测试失败 | 通常不阻断合入，但要关注 |
| **FAILURE** | 构建失败 | 阻断 |
| **ABORTED** | 被中止（超时/人工） | **≠ 故障**，别误判 |
| **NOT_BUILT** | 未执行（前置条件不满足） | — |

> 💡 **ABORTED 陷阱**：看到 ABORTED 别直接当故障查——多半是超时被砍、构被人手动停、或资源排队太久。先看原因。

## 四、通知（结果推给人）

| 渠道 | 插件 |
|:-----|:-----|
| 邮件 | Email Extension（可定制模板、按状态定收件人） |
| Slack | Slack Notification |
| 企业微信 | 企微机器人 Webhook |
| 钉钉 | 钉钉机器人 |
| 通用 Webhook | 回调任意 HTTP 接口 |

```groovy
post {
    failure {
        emailext(
            to: 'team@example.com',
            subject: "构建失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: '请检查构建日志')
    }
    success {
        sh 'curl -X POST -H "Content-Type: application/json" \
            -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"构建成功\"}}" \
            https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx'
    }
}
```

- **⚠️ 坑**：邮件发不出去先查 ① 是否配了 SMTP ② 系统配置里的邮件地址 ③ 安全策略/防火墙

## 五、PR / MR 状态回写

把构建结果**打在代码评审页面上**，评审人一眼看到 CI 过没过。

- **Gitea/GitHub/GitLab 插件**：自动回写 commit status
- 效果：PR 显示 ✅/❌ 徽章，未通过不让合并
- **配合**：`when { changeRequest() }` 让 PR 触发专用检查（如"只验证不发布"）

## 六、质量闭环全景

```
   提交 / MR
      │
      ▼
 ┌──────────┐    ┌───────────┐    ┌──────────┐
 │ 构建编译  │───▶│ 测试 + 扫描│───▶│  门禁判定 │
 └──────────┘    └───────────┘    └────┬─────┘
                                       │
                     ┌─────────────────┼─────────────────┐
                     ▼                 ▼                 ▼
                ✅ 通过            ⚠️ UNSTABLE        ❌ FAILURE
                允许合入           提醒关注           阻断合入
                     │                 │                 │
                     └────────┬────────┴────────┬────────┘
                              ▼                 ▼
                       PR/MR 状态回写      通知（邮件/企微/Slack）
                              │
                              ▼
                        质量看板 / 趋势
```

> 💡 **实战三条**：① 门禁阈值要可配、分阶段（提交级轻、版本级全）；② 第三方噪声必须 suppress，否则门禁形同虚设；③ 扫描/Fuzz 这类重活只触发、算力放常驻服务，别占 Jenkins 控制器。

---

> 🦐 虾管家 · 2026-09-16

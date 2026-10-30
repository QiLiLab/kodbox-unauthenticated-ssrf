# kodbox 公开分享接口的 SSRF 与响应内容泄露

[English](README.md) · [本地复现步骤](POC.md)

本仓库整理所附报告中的 kodbox 服务端请求伪造（SSRF）问题。攻击者需要知道一个可访问的公开分享 ID，但无需登录账号。所附报告称已在官方 Docker 镜像 `kodcloud/kodbox:1.68.10` 上完成本地复现；我们还核对了上游 `1.70.03` 发布提交中的关键 URL 放行代码。这里没有独立复现 `1.70.03` 的运行行为。

## 问题概述

`explorer.share.fileGet` 将请求中的 `path` 交给 `share.class.php::parsePath()`。当 `path` 是被允许的 URL 时，该函数会在检查分享目录边界之前直接返回 URL。后续文件读取流程由服务器访问该 URL，并将响应正文放进返回给请求者的 JSON `data.content` 字段。

这使知道有效公开分享 ID 的匿名访问者有机会读取 kodbox 服务器可访问的内部 HTTP 服务。所附报告还记录了 `fileOut` / `fileOutBy` 对 URL 路径产生外部重定向的次要问题。

| 项目 | 内容 |
| --- | --- |
| 产品 | [kalcaddle/kodbox](https://github.com/kalcaddle/kodbox) |
| 报告中的运行复现版本 | 官方 Docker 镜像 `kodcloud/kodbox:1.68.10` |
| 已核对源码版本 | 上游 `1.70.03` 发布提交 [`d5b517b`](https://github.com/kalcaddle/kodbox/commit/d5b517b) |
| 漏洞类型 | CWE-918，SSRF |
| 前提 | 有效且可访问的公开分享；无需登录账号 |
| 报告中的结果 | 容器内测试服务的完整响应正文进入 `data.content` |
| 参考评分 | `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:N/A:N`（8.6），尚非最终评分 |

## 根因与证据

上游 [`share.class.php` 第 311–317 行](https://github.com/kalcaddle/kodbox/blob/d5b517b/app/controller/explorer/share.class.php#L311-L317) 先执行 `if(request_url_safe($path)) return $path;`，随后才进行分享路径校验。[`fileGet()`](https://github.com/kalcaddle/kodbox/blob/d5b517b/app/controller/explorer/share.class.php#L460-L465) 调用该函数处理请求路径。[`request_url_safe()`](https://github.com/kalcaddle/kodbox/blob/d5b517b/app/function/web.function.php#L760-L779) 允许 `http`、`https`、`ftp`，默认没有强制拦截回环、内网或链路本地地址；站点可以自行定义可选的 `request_url_safe_self` 钩子。

所附报告在隔离 Docker 环境内建立只绑定容器 `127.0.0.1:9999` 的测试服务，然后通过匿名公开分享请求获得 `INTERNAL-ONLY-DATA`。报告中的 `file://` 路径被拒绝，无效分享 ID 也被拒绝。[POC.md](POC.md) 给出脱敏后的本地复现步骤，未包含真实会话令牌或可复用的分享 ID。

## 影响与边界

直接影响是读取 kodbox 服务器可访问的服务响应。能否访问云元数据或其他敏感服务取决于实际网络环境。本仓库没有证据表明曾访问任何第三方实例或真实云凭据。此问题需要一个有效的公开分享 ID，不能表述成完全不依赖分享上下文。

## 与已有公告的关系

[CVE-2026-5618 / GHSA-582g-j67j-5q79](https://github.com/advisories/GHSA-582g-j67j-5q79) 记录了 kodbox `shareMake/shareCheck` 组件中通过 `siteFrom/siteTo` 触发的 SSRF。此报告关注的是 `explorer.share.fileGet` 的 `path` 参数，入口不同；两者是否被维护者视为同一问题，需要进一步确认。本仓库不主张已有或新的 CVE 编号。

## 修复建议

1. 公开分享文件读取路径应拒绝任意 URL，或只允许分享者明确登记的 URL 对象。
2. 通用 URL 获取流程应检查 DNS 解析后的目标和重定向目标，拦截回环、私网和链路本地地址等。
3. 避免 `fileOut` / `fileOutBy` 把 URL 路径直接写入 `Location` 响应头。
4. 增加公开分享匿名请求访问内网地址的回归用例。

## 披露状态

本仓库为公开的脱敏技术记录。公开仓库不等于通知 kodbox 维护者，也不等于向其仓库提交私密漏洞报告。

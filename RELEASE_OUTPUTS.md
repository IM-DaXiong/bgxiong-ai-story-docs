# 本仓交付归集入口（BAS-3，2026-11-23）

用户指定唯一最终交付根 `L:\00Dev\bgxiong-00-release`。所有智能体完成打包发行、开发/测试版编译或开发人员拿走测试前，必须按下表归集已完成成果；只报target/dist/旧artifacts路径不算交付。

完整权威合同（本机开发文档，不是对外SDK构建依赖）：`L:/00Dev/bgxiong-ai-story/docs/dev/guides/20261123-GUIDE-unified-delivery-root.md`。本仓源码、缓存、运行路径与用户安装布局不改，外部用户无需此L盘。密钥、私有配置、数据库/备份、源码checkout、缓存和日志不入交付目录；不移动正在运行的程序。

- **文档站/手册正式导出**：旧 `release/现行文档；主仓/Cloud手册消费副本` → 唯一根下 `04-deploy/docs-site/<version-build>`。入口 `release/README.md; README.md`。当前无证据支持独立npm构建入口；只归集既有白名单发布导出，不虚构dist/站点成品，不发布archive/history或内部报告。

CDN按已配置URL的相对键完整保留；打开本地与远端相同前缀目录，拖**其中内容**，不得把03-cdn/主机名/分类/批次层额外传上去。ZIP/manifest/latest/index/signature/checksum必须同批，生产验签/资格/序号按原规范，不能用测试签名冒充生产。

若入口有输出参数，显式传对应目标；否则构建成功并完成现行验收后，用主仓`scripts/stage-delivery-output.py --source <完整已验成品> --relative <上表相对路径> --check`核对，再去掉--check归集。主仓不在本机时按相同规则人工白名单复制并核每文件摘要，不因此要求外部SDK用户安装主程序。此轮只规划和改规范，除主仓安装器归集、Resolve输出常量及独立stage辅助外，其余旧构建默认未全部改线；禁止声称已自动迁移/构建。

完成前回报真实归集路径、版本/源码/平台、文件闭包与摘要、实际验证层及NOT_RUN；空目录/复制成功不能代替发布/真实运行验收。待签名、缺包、未归集或依赖原checkout的启动器均标待交付。

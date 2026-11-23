> **BAS-3 修订 / 用户 t372–t374 / 2026-11-23**：唯一交付根 `L:\00Dev\bgxiong-00-release` 下必须直接平级 `dev-项目名`、`release-项目名`、`cdn-项目名`、`deploy-项目名`，不再有编号分类/主机名/client/ai-story导航外壳。进入项目目录就选/拖成品；包内模块/平台结构照旧，CDN实际对象键不变。按本仓 RELEASE_OUTPUTS.md；同项目/用途/类型/平台/渠道只保留有版本/构建证据的最新完整成品，核验后删旧版，不按复制mtime判断；运行安装目录、源码、配置、缓存不迁移。

# 本仓交付归集入口（BAS-3，2026-11-23）

用户指定唯一最终交付根 `L:\00Dev\bgxiong-00-release`。所有智能体完成打包发行、开发/测试版编译或开发人员拿走测试前，必须按下表归集已完成成果；只报target/dist/旧artifacts路径不算交付。

完整权威合同（本机开发文档，不是对外SDK构建依赖）：`L:/00Dev/bgxiong-ai-story/docs/dev/guides/20261123-GUIDE-unified-delivery-root.md`。本仓源码、缓存、运行路径与用户安装布局不改，外部用户无需此L盘。密钥、私有配置、数据库/备份、源码checkout、缓存和日志不入交付目录；不移动正在运行的程序。

- **文档站/手册正式导出**：旧 `release/现行文档；主仓/Cloud手册消费副本` → 唯一根下 `deploy-bgxiong-ai-story-docs`。入口 `release/README.md; README.md`。当前无证据支持独立npm构建入口；只归集既有白名单发布导出，不虚构dist/站点成品，不发布archive/history或内部报告。

CDN项目平级目录只放该远端前缀的载荷，拖**其中内容**；本机 cdn-项目名 不是远端对象目录，不要作为包装层上传。ZIP/manifest/latest/index/signature/checksum必须同批，生产验签/资格/序号按原规范，不能用测试签名冒充生产。

若入口有输出参数，显式传对应目标；否则构建成功并完成现行验收后，用主仓`scripts/stage-delivery-output.py --source <完整已验成品> --relative <上表相对路径> --check`核对，再去掉--check归集。主仓不在本机时按相同规则人工白名单复制并核每文件摘要，不因此要求外部SDK用户安装主程序。自动归集入口与显式stage边界见权威合同；本次包含已确认交付成品迁移，但没有构建、打包、签名、上传或部署。源码内部工作目录/运行安装档案不因归集要求迁移。

完成前回报真实归集路径、版本/源码/平台、文件闭包与摘要、实际验证层及NOT_RUN；空目录/复制成功不能代替发布/真实运行验收。待签名、缺包、未归集或依赖原checkout的启动器均标待交付。
